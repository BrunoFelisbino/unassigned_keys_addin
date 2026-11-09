# Instalação e atualização

## Pré-requisitos

- Conta **MyGeotab** com permissão para gerenciar add-ins (Administração).
- Os arquivos publicados em um host HTTPS acessível (aqui: **GitHub Pages**).

## Publicação (GitHub Pages)

O repositório já está configurado para publicar a partir da branch `main`, na raiz.

1. Envie as alterações para `origin/main`:
   ```powershell
   git add -A
   git commit -m "descrição da mudança"
   git push origin main
   ```
2. Confirme em **Settings → Pages**:
   - Source: `Deploy from a branch`
   - Branch: `main` / `/ (root)`
3. O arquivo `.nojekyll` já está presente (evita o processamento Jekyll).

Após o deploy, os recursos ficam em:

- Config: `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_addin_config.json`
- Página: `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys.html`
- Ícone: `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_icon.svg`

## Cadastro no MyGeotab

1. Abra **Administração → Sistema → Add-ins**.
2. Clique em **Novo add-in** (ou "Add").
3. Informe a **URL do config**:
   `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_addin_config.json`
4. Salve. O add-in aparece no menu **ActivityLink → Chaves Não Cadastradas**.
5. Abra a página. Os dados do dia carregam automaticamente.

## Atualização

O GitHub Pages usa CDN e pode continuar servindo a versão antiga por alguns minutos.
Para garantir que a nova versão chegue ao navegador:

1. Edite `unassigned_keys.html`.
2. Em `unassigned_keys_addin_config.json`:
   - suba `"version"` (ex.: `1.0.6` → `1.0.7`);
   - suba o `?v=N` da URL (ex.: `?v=7` → `?v=8`).
3. Commit + push.
4. No MyGeotab:
   - remova e re-adicione o add-in **ou** aguarde o CDN alguns minutos;
   - abra o add-in e pressione **Ctrl+F5**.

> Se após 5 minutos a página ainda estiver antiga, faça um commit vazio para forçar um novo deploy:
> `git commit --allow-empty -m "chore: force pages rebuild"` e `git push`.

## Verificação rápida

- Config publicado: abra a URL do config e confira `"version"`.
- Conteúdo publicado (sem cache do navegador):
  `https://raw.githubusercontent.com/BrunoFelisbino/unassigned_keys_addin/main/unassigned_keys_addin_config.json`
- Deploy mais recente do Pages:
  `https://api.github.com/repos/BrunoFelisbino/unassigned_keys_addin/deployments?per_page=1`

## Remoção

No MyGeotab, **Administração → Sistema → Add-ins**, selecione o add-in e remova.
