# Implementação do Sistema de Auto-Update

Este documento descreve todas as etapas e regras implementadas para o sistema de auto-update, suportando Windows, macOS e Linux.

## Visão Geral

O sistema de auto-update utiliza o plugin `tauri-plugin-updater` do Tauri v2, que verifica atualizações no GitHub Releases e baixa/instala automaticamente quando uma nova versão está disponível. O sistema inclui verificação de assinaturas para garantir a integridade dos downloads.

## Pré-requisitos

- Tauri v2
- tauri-plugin-updater v2
- Chaves de signing (minisign) para verificação de assinaturas
- GitHub Actions para builds automatizados
- GitHub Releases para distribuição

## Etapas de Implementação

### 1. Configuração do Plugin Updater (Rust)

#### 1.1 Adicionar dependência no Cargo.toml

```toml
[dependencies]
tauri-plugin-updater = "2"
```

#### 1.2 Registrar o plugin em src-tauri/src/lib.rs

```rust
use tauri_plugin_updater::Builder;

fn main() {
    tauri::Builder::default()
        .plugin(Builder::new().build())
        // ...
}
```

#### 1.3 Criar comandos Rust para o updater

```rust
#[tauri::command]
async fn check_for_update(app: tauri::AppHandle) -> Result<serde_json::Value, String> {
    use tauri_plugin_updater::UpdaterExt;
    
    match app.updater() {
        Ok(updater) => {
            match updater.check().await {
                Ok(Some(update)) => {
                    Ok(serde_json::json!({
                        "available": true,
                        "version": update.version,
                        "body": update.body,
                        "date": update.date.map(|d| d.to_string())
                    }))
                }
                Ok(None) => {
                    Ok(serde_json::json!({
                        "available": false
                    }))
                }
                Err(e) => Err(format!("Failed to check for updates: {}", e))
            }
        }
        Err(e) => Err(format!("Failed to get updater: {}", e))
    }
}

#[tauri::command]
async fn install_update(app: tauri::AppHandle) -> Result<String, String> {
    use tauri_plugin_updater::UpdaterExt;
    
    match app.updater() {
        Ok(updater) => {
            match updater.check().await {
                Ok(Some(update)) => {
                    match update.download_and_install(
                        |chunk_length, content_length| {
                            let content_len = content_length.unwrap_or(1);
                            let progress = chunk_length as f64 / content_len as f64;
                            println!("Download progress: {:.2}%", progress * 100.0);
                        },
                        || {
                            println!("Download finished");
                        },
                    ).await {
                        Ok(_) => {
                            app.restart();
                        }
                        Err(e) => Err(format!("Failed to install update: {}", e))
                    }
                }
                Ok(None) => Err("No update available".to_string()),
                Err(e) => Err(format!("Failed to check for updates before install: {}", e))
            }
        }
        Err(e) => Err(format!("Failed to get updater: {}", e))
    }
}

#[tauri::command]
fn get_current_version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}
```

### 2. Configuração do tauri.conf.json

#### 2.1 Adicionar configuração do updater

```json
{
  "bundle": {
    "createUpdaterArtifacts": true
  },
  "plugins": {
    "updater": {
      "pubkey": "CHAVE_PUBLICA",
      "endpoints": [
        "https://github.com/alanenggb/whiplash-music-player/releases/latest/download/latest.json"
      ]
    }
  }
}
```

**Regras importantes:**
- `createUpdaterArtifacts: true` é obrigatório para gerar arquivos de assinatura `.sig`
- `pubkey` deve ser a chave pública gerada pelo minisign
- `endpoints` deve apontar para o `latest.json` no GitHub Releases

### 3. Geração de Chaves de Signing

#### 3.1 Gerar par de chaves

```bash
npm run tauri signer generate
```

Digite um password e confirme. Guarde este password em um local seguro.

Isso gera dois valores no terminal:
- Private: (Keep it secret!): [valor]
- Public: [valor]

#### 3.2 Configurar variáveis de ambiente

No seu computador, adicione as seguintes variáveis de ambiente:

- `TAURI_SIGNING_PRIVATE_KEY`: O valor da chave privada gerada
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`: A senha configurada durante a geração

Execute o comando de build para testar:

```bash
npm run tauri build
```

#### 3.3 Configurar variáveis de ambiente no GitHub

No GitHub, vá em Settings > Secrets and variables > Actions e adicione:

- `TAURI_SIGNING_PRIVATE_KEY`: O valor da chave privada gerada
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`: A senha configurada durante a geração

### 4. Configuração do GitHub Actions Workflow

#### 4.1 Criar arquivo .github/workflows/release.yml

```yaml
name: Build e Release
on:
  push:
    branches:
      - main
    paths:
      - 'src-tauri/tauri.conf.json'

jobs:
  publish:
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        platform: [macos-latest, ubuntu-22.04, windows-latest]

    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: 'npm'

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
      
      - name: Rust Cache
        uses: swatinem/rust-cache@v2
        with:
          key: ${{ matrix.platform }}

      - name: Install dependencies
        run: npm install

      - name: Install Dependencies (Linux only)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libgtk-3-dev libwebkit2gtk-4.1-dev libayatana-appindicator3-dev librsvg2-dev libsoup-3.0-dev libjavascriptcoregtk-4.1-dev

      - name: Build and Package
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
        with:
          tagName: whiplash-music-player-v__VERSION__
          releaseName: "whiplash-music-player v__VERSION__"
          releaseDraft: true
          prerelease: false

      - name: Save Windows signature
        if: matrix.platform == 'windows-latest'
        shell: pwsh
        run: |
          $NSIS_DIR = "src-tauri/target/release/bundle/nsis"
          $MSI_DIR = "src-tauri/target/release/bundle/msi"
          
          $SIG_FILES = Get-ChildItem -Path $NSIS_DIR -Filter "*.sig" -ErrorAction SilentlyContinue
          if ($SIG_FILES) {
            foreach ($SIG_FILE in $SIG_FILES) {
              $SIG_CONTENT = Get-Content $SIG_FILE.FullName -Raw
              $SIG_CONTENT | Out-File -FilePath "windows-x86_64.sig" -Encoding utf8 -NoNewline
              break
            }
          }
          
          if (-not $SIG_CONTENT) {
            $SIG_FILES = Get-ChildItem -Path $MSI_DIR -Filter "*.sig" -ErrorAction SilentlyContinue
            if ($SIG_FILES) {
              foreach ($SIG_FILE in $SIG_FILES) {
                $SIG_CONTENT = Get-Content $SIG_FILE.FullName -Raw
                $SIG_CONTENT | Out-File -FilePath "windows-x86_64.sig" -Encoding utf8 -NoNewline
                break
              }
            }
          }

      - name: Save macOS signatures
        if: matrix.platform == 'macos-latest'
        run: |
          find src-tauri/target/release/bundle -name "*.sig" -exec cp {} . \;
          
          ARCH=$(uname -m)
          
          if [ -f "whiplash-music-player.app.tar.gz.sig" ]; then
            if [ "$ARCH" = "arm64" ]; then
              mv whiplash-music-player.app.tar.gz.sig darwin-aarch64.sig
            else
              mv whiplash-music-player.app.tar.gz.sig darwin-x86_64.sig
            fi
          else
            SIG_FILE=$(ls *.sig 2>/dev/null | head -1)
            if [ -n "$SIG_FILE" ]; then
              if [ "$ARCH" = "arm64" ]; then
                mv "$SIG_FILE" darwin-aarch64.sig
              else
                mv "$SIG_FILE" darwin-x86_64.sig
              fi
            fi
          fi

      - name: Save Linux signature
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          find src-tauri/target/release/bundle -name "*.sig" -exec sh -c 'cp "$1" "$(basename "$1" .sig).sig" .' _ {} \;
          
          if [ -f "whiplash-music-player.AppImage.sig" ]; then
            mv whiplash-music-player.AppImage.sig linux-x86_64.sig
          fi

      - name: Upload signatures as artifacts
        uses: actions/upload-artifact@v4
        with:
          name: signatures-${{ matrix.platform }}
          path: "*.sig"
          if-no-files-found: ignore

  generate-latest-json:
    needs: publish
    permissions:
      contents: write
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: lts/*
      
      - name: Download Windows signature
        uses: actions/download-artifact@v4
        with:
          name: signatures-windows-latest
          path: windows
      
      - name: Download macOS signature
        uses: actions/download-artifact@v4
        with:
          name: signatures-macos-latest
          path: macos
      
      - name: Download Linux signature
        uses: actions/download-artifact@v4
        with:
          name: signatures-ubuntu-22.04
          path: linux
      
      - name: Generate latest.json
        run: |
          VERSION=$(node -p "require('./src-tauri/tauri.conf.json').version")
          
          WINDOWS_SIG=""
          DARWIN_X86_64_SIG=""
          DARWIN_AARCH64_SIG=""
          LINUX_X86_64_SIG=""
          
          if [ -f "windows/windows-x86_64.sig" ]; then
            WINDOWS_SIG=$(cat windows/windows-x86_64.sig)
          fi
          
          if [ -f "macos/darwin-x86_64.sig" ]; then
            DARWIN_X86_64_SIG=$(cat macos/darwin-x86_64.sig)
          fi
          
          if [ -f "macos/darwin-aarch64.sig" ]; then
            DARWIN_AARCH64_SIG=$(cat macos/darwin-aarch64.sig)
          fi
          
          if [ -f "linux/linux-x86_64.sig" ]; then
            LINUX_X86_64_SIG=$(cat linux/linux-x86_64.sig)
          fi
          
          cat > generate-latest.cjs << 'EOF'
          const fs = require('fs');
          const version = process.env.VERSION;
          const windowsSig = process.env.WINDOWS_SIG || "";
          const darwinX86_64Sig = process.env.DARWIN_X86_64_SIG || "";
          const darwinAarch64Sig = process.env.DARWIN_AARCH64_SIG || "";
          const linuxX86_64Sig = process.env.LINUX_X86_64_SIG || "";
          
          const latestJson = {
            version: version,
            notes: "whiplash-music-player v" + version,
            pub_date: new Date().toISOString(),
            platforms: {
              "windows-x86_64": {
                signature: windowsSig.trim(),
                url: "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v" + version + "/whiplash-music-player_" + version + "_x64-setup.exe"
              },
              "darwin-x86_64": {
                signature: darwinX86_64Sig.trim(),
                url: "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v" + version + "/whiplash-music-player_" + version + "_x64.dmg"
              },
              "darwin-aarch64": {
                signature: darwinAarch64Sig.trim(),
                url: "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v" + version + "/whiplash-music-player_" + version + "_aarch64.dmg"
              },
              "linux-x86_64": {
                signature: linuxX86_64Sig.trim(),
                url: "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v" + version + "/whiplash-music-player_" + version + "_amd64.AppImage"
              }
            }
          };
          fs.writeFileSync("latest.json", JSON.stringify(latestJson, null, 2));
          EOF
          
          export VERSION=$VERSION
          export WINDOWS_SIG=$WINDOWS_SIG
          export DARWIN_X86_64_SIG=$DARWIN_X86_64_SIG
          export DARWIN_AARCH64_SIG=$DARWIN_AARCH64_SIG
          export LINUX_X86_64_SIG=$LINUX_X86_64_SIG
          node generate-latest.cjs
          
          TAG_NAME="whiplash-music-player-v$VERSION"
          gh release upload "$TAG_NAME" latest.json --repo alanenggb/whiplash-music-player --clobber
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Regras importantes:**
- A chave privada nunca deve ser commitada no repositório
- Use secrets do GitHub para armazenar informações sensíveis
- A senha da chave privada deve ser a mesma usada durante a geração

### 5. Implementação do Frontend (TypeScript)

#### 5.1 Adicionar botão de verificação manual

```typescript
private async checkForUpdates() {
    const originalText = this.elements.checkUpdateBtn.textContent;
    this.elements.checkUpdateBtn.disabled = true;
    this.elements.checkUpdateBtn.textContent = '🔄 Verificando...';
    
    try {
      const result = await invoke<{ available: boolean; version?: string; body?: string; date?: string }>('check_for_update');
      
      if (result.available) {
        const currentVersion = await invoke<string>('get_current_version');
        this.showUpdateModal(result.version || 'desconhecida', currentVersion, result.body || '');
      } else {
        this.showToast('Você já está usando a versão mais recente!', 'success');
      }
    } catch (error) {
      const errorMessage = String(error);
      console.error('Erro ao verificar atualizações:', error);
      
      if (errorMessage.includes('Could not fetch a valid release JSON') || 
          errorMessage.includes('Failed to check for updates')) {
        this.showToast('Nenhuma atualização disponível.', 'success');
      } else {
        this.showToast('Erro ao verificar atualizações. Tente novamente.', 'error');
      }
    } finally {
      this.elements.checkUpdateBtn.disabled = false;
      this.elements.checkUpdateBtn.textContent = originalText;
    }
}
```

#### 5.2 Adicionar verificação ao iniciar

```typescript
private async checkForUpdatesOnStartup() {
    try {
      const result = await invoke<{ available: boolean; version?: string; body?: string; date?: string }>('check_for_update');
      
      if (result.available) {
        const currentVersion = await invoke<string>('get_current_version');
        this.showUpdateModal(result.version || 'desconhecida', currentVersion, result.body || '');
      }
    } catch (error) {
      console.log('Verificação de update ao iniciar: update não disponível ou erro ao verificar');
    }
}
```

#### 5.3 Adicionar verificação periódica (24 horas)

```typescript
private setupPeriodicUpdateCheck() {
    const CHECK_INTERVAL = 24 * 60 * 60 * 1000; // 24 horas
    
    setInterval(async () => {
      try {
        const result = await invoke<{ available: boolean; version?: string; body?: string; date?: string }>('check_for_update');
        
        if (result.available) {
          const currentVersion = await invoke<string>('get_current_version');
          this.showUpdateModal(result.version || 'desconhecida', currentVersion, result.body || '');
        }
      } catch (error) {
        console.log('Verificação periódica de update: erro ao verificar');
      }
    }, CHECK_INTERVAL);
}
```

#### 5.4 Adicionar modal de atualização

```html
<div id="updateModal" class="modal">
  <div class="modal-content">
    <div class="modal-header">
      <h2>Atualização Disponível</h2>
      <button class="close-btn" onclick="closeUpdateModal()">&times;</button>
    </div>
    <div class="modal-body">
      <p>Nova versão: <strong id="newVersion"></strong></p>
      <p>Versão atual: <strong id="currentVersion"></strong></p>
      <div id="updateNotes"></div>
      <div class="modal-actions">
        <button id="installUpdateBtn" onclick="installUpdate()">Instalar Agora</button>
        <button onclick="closeUpdateModal()">Depois</button>
      </div>
    </div>
  </div>
</div>
```

#### 5.5 Adicionar função de instalação

```typescript
async function installUpdate() {
  const installBtn = document.getElementById('installUpdateBtn');
  installBtn.disabled = true;
  installBtn.textContent = 'Baixando...';
  
  try {
    await invoke('install_update');
  } catch (error) {
    console.error('Erro ao instalar atualização:', error);
    alert('Erro ao instalar atualização: ' + String(error));
    installBtn.disabled = false;
    installBtn.textContent = 'Instalar Agora';
  }
}
```

### 6. Configuração de Permissões (Tauri)

Adicione as permissões necessárias em `src-tauri/capabilities/default.json`:

```json
{
  "identifier": "default",
  "description": "Default capabilities",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "dialog:allow-confirm",
    "dialog:allow-message",
    "updater:allow-check-for-update",
    "updater:allow-download-and-install"
  ]
}
```

**Regras importantes:**
- `dialog:allow-confirm` e `dialog:allow-message` são necessários para diálogos de confirmação
- `updater:allow-check-for-update` permite verificar atualizações
- `updater:allow-download-and-install` permite baixar e instalar atualizações

### 7. Processo de Release

#### 7.1 Atualizar versão

Atualize a versão em três arquivos:
- `package.json`
- `src-tauri/Cargo.toml`
- `src-tauri/tauri.conf.json`

```json
// package.json
{
  "version": "0.1.0"
}

// src-tauri/Cargo.toml
[package]
version = "0.1.0"

// src-tauri/tauri.conf.json
{
  "version": "0.1.0"
}
```

#### 7.2 Commit e push

```bash
git add .
git commit -m "Release v0.1.0"
git push origin main
```

#### 7.3 Aguardar build do GitHub Actions

O workflow será acionado automaticamente pelo push em `src-tauri/tauri.conf.json`. Aguarde a conclusão do build.

#### 7.4 Publicar release

Vá até a release criada no GitHub (estará em draft) e clique em "Publish release".

**Regras importantes:**
- O updater só funcionará quando a release for publicada (não draft)
- O workflow cria releases como draft por segurança
- Sempre revise os artefatos antes de publicar

## Arquitetura do latest.json

O arquivo `latest.json` deve ter a seguinte estrutura:

```json
{
  "version": "0.1.0",
  "notes": "whiplash-music-player v0.1.0",
  "pub_date": "2026-04-16T15:52:01.178Z",
  "platforms": {
    "windows-x86_64": {
      "signature": "base64_encoded_signature",
      "url": "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v0.1.0/whiplash-music-player_0.1.0_x64-setup.exe"
    },
    "darwin-x86_64": {
      "signature": "base64_encoded_signature",
      "url": "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v0.1.0/whiplash-music-player_0.1.0_x64.dmg"
    },
    "darwin-aarch64": {
      "signature": "base64_encoded_signature",
      "url": "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v0.1.0/whiplash-music-player_0.1.0_aarch64.dmg"
    },
    "linux-x86_64": {
      "signature": "base64_encoded_signature",
      "url": "https://github.com/alanenggb/whiplash-music-player/releases/download/whiplash-music-player-v0.1.0/whiplash-music-player_0.1.0_amd64.AppImage"
    }
  }
}
```

**Regras importantes:**
- `signature` deve estar em base64
- `url` deve apontar para o arquivo correto na release
- O nome do arquivo deve corresponder exatamente ao gerado pelo Tauri
- macOS tem duas arquiteturas: `darwin-x86_64` (Intel) e `darwin-aarch64` (Apple Silicon)

## Problemas Comuns e Soluções

### 1. Erro: "Invalid encoding in minisign data"

**Causa:** Assinatura vazia ou inválida no `latest.json`

**Solução:** Verifique se o workflow está gerando corretamente as assinaturas e se elas estão sendo incluídas no `latest.json`.

### 2. Erro: "The signature verification failed"

**Causa:** Assinatura não corresponde ao arquivo ou chave pública incorreta

**Solução:** 
- Verifique se a chave pública no `tauri.conf.json` está correta
- Verifique se as assinaturas estão sendo geradas corretamente
- Verifique se o URL do arquivo no `latest.json` está correto

### 3. Erro: "Could not fetch a valid release JSON"

**Causa:** `latest.json` não existe ou está malformado

**Solução:** 
- Verifique se o `latest.json` foi gerado e uploadado para a release
- Verifique se o endpoint no `tauri.conf.json` está correto
- Verifique se a release foi publicada (não draft)

### 4. Erro: "pubkey is required"

**Causa:** Chave pública não configurada no `tauri.conf.json`

**Solução:** Adicione a chave pública gerada pelo minisign na configuração do updater.

## Plataformas Suportadas

### Windows
- **Arquitetura:** x86_64
- **Formato:** NSIS (setup.exe) e MSI
- **Assinatura:** windows-x86_64.sig

### macOS
- **Arquiteturas:** 
  - aarch64 (Apple Silicon/M1/M2/M3)
  - x86_64 (Intel) - *limitado*
- **Formato:** DMG e app.tar.gz
- **Assinatura:** darwin-aarch64.sig e darwin-x86_64.sig

### Linux
- **Arquitetura:** x86_64
- **Formato:** AppImage, DEB, RPM
- **Assinatura:** linux-x86_64.sig

## Boas Práticas

1. **Sempre teste localmente antes de publicar**
2. **Mantenha a chave privada segura**
3. **Use releases como draft para revisão**
4. **Atualize a versão em todos os arquivos simultaneamente**
5. **Verifique os logs do GitHub Actions em caso de erro**
6. **Teste o updater em cada plataforma antes de anunciar a atualização**
7. **Mantenha notas de versão claras e informativas**

## Resumo do Fluxo de Trabalho

1. Desenvolver nova versão
2. Atualizar versão em `package.json`, `Cargo.toml` e `tauri.conf.json`
3. Commit e push para main
4. GitHub Actions gera builds para todas as plataformas
5. Assinaturas são geradas e coletadas como artifacts
6. Job `generate-latest-json` cria o `latest.json` com todas as assinaturas
7. `latest.json` é uploadado para a release
8. Publicar release no GitHub
9. Usuários recebem notificação de atualização
10. Usuários baixam e instalam a atualização automaticamente

## Arquivos Modificados

- `src-tauri/Cargo.toml` - Adicionada dependência do updater
- `src-tauri/src/lib.rs` - Comandos do updater e registro do plugin
- `src-tauri/tauri.conf.json` - Configuração do updater
- `src-tauri/capabilities/default.json` - Permissões do updater
- `src/main.ts` - Lógica de verificação e instalação de atualizações
- `index.html` - Modal de atualização
- `src/styles.css` - Estilos do modal
- `.github/workflows/release.yml` - Workflow de build e geração de latest.json
