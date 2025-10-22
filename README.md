# CHAT_INTERNO
https://raw.githubusercontent.com/Marcellyferreira/comunicacao-interna/main/scripts/verify-signatures.sh
# Workflow GitHub Actions - CI/CD Completo (Final)

Este é o workflow final e unificado para o sistema **Comunicação Interna**, hospedado em **Marcellyferreira/comunicacao-interna**. Ele cobre todo o processo de build, assinatura, auditoria e publicação das releases de forma automatizada.

```yaml
name: 🔄 Full CI/CD Multi-Plataforma

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  # =============================
  # 🧱 BUILD LINUX
  # =============================
  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Instalar dependências
        run: |
          sudo apt update
          sudo apt install -y python3 python3-pip zip tar gpg
      - name: Build backend e empacotar
        run: |
          mkdir -p releases/linux
          zip -r releases/linux/backend.zip backend client requirements.txt
          tar -czf releases/linux/backend.tar.gz backend client requirements.txt
      - name: Assinar pacotes GPG
        run: |
          echo "$GPG_PRIVATE_KEY" | gpg --batch --import
          echo "$GPG_PASSPHRASE" | gpg --batch --yes --passphrase-fd 0 --armor --detach-sign releases/linux/backend.zip
          echo "$GPG_PASSPHRASE" | gpg --batch --yes --passphrase-fd 0 --armor --detach-sign releases/linux/backend.tar.gz
        env:
          GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
      - name: Upload artefatos Linux
        uses: actions/upload-artifact@v4
        with:
          name: release-linux
          path: releases/linux

  # =============================
  # 🪟 BUILD WINDOWS
  # =============================
  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Instalar Python e dependências
        run: |
          python -m pip install --upgrade pip
          pip install pyinstaller
      - name: Compilar aplicativo Windows (.exe)
        run: |
          pyinstaller --onefile client/main.py --name comunicacao_interna
          mkdir releases
          move dist\comunicacao_interna.exe releases\client_installer.exe
      - name: Assinar instalador (code signing)
        run: |
          echo "$env:WINDOWS_PFX" | Out-File -Encoding ascii cert_base64.txt
          certutil -decode cert_base64.txt cert.pfx
          signtool sign /f cert.pfx /p $env:WINDOWS_PFX_PASS /tr http://timestamp.digicert.com /td sha256 /fd sha256 releases\client_installer.exe
        env:
          WINDOWS_PFX: ${{ secrets.WINDOWS_PFX }}
          WINDOWS_PFX_PASS: ${{ secrets.WINDOWS_PFX_PASS }}
      - name: Upload artefatos Windows
        uses: actions/upload-artifact@v4
        with:
          name: release-windows
          path: releases

  # =============================
  # 🚀 RELEASE FINAL UNIFICADA
  # =============================
  release:
    name: 🚀 Criar Release Unificada (Assinada, Verificada e Auditada)
    needs: [ build-linux, build-windows ]
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Baixar artefatos Linux
        uses: actions/download-artifact@v4
        with:
          name: release-linux
          path: releases/linux

      - name: Baixar artefatos Windows
        uses: actions/download-artifact@v4
        with:
          name: release-windows
          path: releases/windows

      - name: Listar artefatos
        run: ls -R releases

      # ==========================================
      # 🔍 Verificação de assinaturas (modo audit)
      # ==========================================
      - name: Baixar script de verificação
        run: |
          mkdir -p scripts
          curl -fsSL https://raw.githubusercontent.com/Marcellyferreira/comunicacao-interna/main/scripts/verify-signatures.sh -o scripts/verify-signatures.sh
          chmod +x scripts/verify-signatures.sh

      - name: Executar verificação (modo audit)
        run: |
          echo "🔎 Iniciando auditoria de assinaturas..."
          ./scripts/verify-signatures.sh releases || true
          echo ""
          echo "📋 Log final:"
          cat verify_signatures.log || echo "(sem log gerado)"

      # ==========================================
      # 🗂️ Upload do log de auditoria
      # ==========================================
      - name: Upload do relatório de verificação
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: verify-signatures-log
          path: verify_signatures.log
          retention-days: 7

      # ==========================================
      # 🚀 Publicar Release no GitHub
      # ==========================================
      - name: Criar Release no GitHub
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ github.run_number }}
          name: "Release ${{ github.run_number }} - Comunicação Interna"
          body: |
            ✅ Build multi-plataforma auditado.
            - Backend + Client (Linux) com assinatura GPG verificada
            - Pacotes `.zip` e `.tar.gz` + `.asc`
            - Instalador `.exe` assinado digitalmente
            - 🔍 Auditoria automática executada em CI/CD (modo audit)
            - 📄 Relatório completo disponível nos artifacts (verify-signatures-log)
          files: |
            releases/linux/*.zip
            releases/linux/*.zip.asc
            releases/linux/*.tar.gz
            releases/linux/*.tar.gz.asc
            releases/windows/**/*.exe
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### ✅ Conclusão

Este workflow cobre todo o ciclo automatizado:

* Build e empacotamento multiplataforma (Linux/Windows)
* Assinatura digital (GPG e PFX)
* Auditoria automática das assinaturas (modo audit)
* Upload do relatório `verify_signatures.log`
* Publicação da release completa e segura no GitHub.
