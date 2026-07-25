---
name: deploy
description: Deploy seguro do app Mimba — commita só o index.html, faz push e verifica que o GitHub Pages publicou. Use quando for subir uma mudança do index.html para produção.
---

# Deploy do app (index.html → GitHub Pages)

O push na `main` publica no Pages; o workflow `versionar.yml` arquiva a versão anterior em `versions/`. **Só o `index.html`** deve ir no commit.

## Passos
1. **Sanidade — o JS compila?**
   `node -e "new Function(require('fs').readFileSync('index.html','utf8').split('<script>').pop().split('</script>')[0]); console.log('ok')"`
2. **Stage só o index.html** e confirme:
   `git add index.html && git diff --cached --name-only` → tem que ser exatamente `index.html`. Se vier outra coisa, **PARE**.
3. **Commit** — mensagem clara + trailer `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
4. **Rebase + push:** `git fetch -q origin main && git pull --rebase -q origin main && git push origin main`
   (o `versionar.yml` costuma ter criado um commit no remoto — o rebase resolve; conflito é raro porque a mudança dele é em `versions/`).
5. **Verificar o build do Pages:** aguarde e confira
   `gh api repos/<owner>/<repo>/pages/builds/latest --jq '{status,error:.error.message}'` → `built`, sem erro.
6. **Conferir o conteúdo ao vivo** (com cache-buster `?cb=<timestamp>`): baixe a URL publicada e cheque um marcador da mudança (uma string nova que você adicionou).

## Regras
- Nunca commitar segredos nem outros arquivos junto.
- Build `errored`? Investigue antes de qualquer novo push.
- Domínio do app: `app.mimba.com.br` (pós-migração) / `<owner>.github.io/cabanha`.
