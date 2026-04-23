# Plan: Corrigir GitHub Actions — Auto Pull Request

**Diagnóstico**: O workflow existe e está correto sintaticamente, mas **não está disparando** porque o último push para `feature/create-project` aconteceu antes (ou no mesmo momento em que) o arquivo foi commitado nessa branch. Para o GitHub Actions re-disparar o evento `push`, um novo commit precisa ser enviado. Além disso, o workflow tem um problema de tratamento de erros que esconde falhas reais.

---

**Problemas encontrados**

1. **Workflow não dispara**: O commit que adicionou o arquivo de workflow já foi enviado ao remote (`8706288`). Para acionar novamente, é necessário um novo `git push`.
2. **Supressão de erros**: O `|| echo "PR already exists..."` captura **todo tipo de erro** (permissão negada, falha de API, token inválido), mascarando problemas reais.
3. **Sem trigger manual**: Não há `workflow_dispatch`, impossibilitando testar o workflow sem criar um commit.
4. **Sem verificação prévia de PR existente**: O workflow tenta criar a PR sem checar antes se já existe uma aberta para aquela branch.

---

**Steps**

### Fase 1 — Corrigir o Workflow

1. Adicionar `workflow_dispatch` ao bloco `on:` para permitir teste manual pela aba Actions.

2. Adicionar `fetch-depth: 0` ao step `actions/checkout@v4` (boa prática para que o git tenha o histórico completo da branch).

3. Substituir o `gh pr create ... || echo` por uma verificação explícita:
   - Checar com `gh pr list --head "$BRANCH" --base main --json number --jq 'length'` se já existe PR
   - Se existe → logar e sair com sucesso
   - Se não existe → criar com `gh pr create` (sem `|| echo`)
   - Erros reais devem **falhar o job** para ficarem visíveis na aba Actions

### Fase 2 — Disparar o Workflow na Branch Atual

4. Após a correção do arquivo ser commitada no `main`, fazer **merge ou rebase** de `main` em `feature/create-project` para que a branch tenha a versão corrigida.

5. Fazer qualquer novo commit na branch (ou um commit vazio: `git commit --allow-empty -m "trigger: auto PR workflow"`) e `git push` para disparar o workflow.

---

**Arquivo modificado**

- `.github/workflows/auto-pull-request.yml` — único arquivo a alterar

---

**Verificação**

1. Após o push, acessar **github.com → repositório → aba Actions** e confirmar que um run de "Auto Pull Request" aparece
2. Confirmar que o run finaliza com sucesso (check verde)
3. Acessar a aba **Pull Requests** e verificar se a PR de `feature/create-project → main` foi criada
4. Opcionalmente, usar o botão **"Run workflow"** (via `workflow_dispatch`) para testar sem precisar criar commit

---

**Decisões**

- Criação da PR aponta sempre para `main` (conforme já está no workflow)
- O workflow cobre qualquer branch futura `feature/**` ou `fix/**` automaticamente, desde que seja criada a partir de `main` após a correção estar lá
- Branches criadas **antes** da adição do workflow precisarão de um rebase/merge de `main` para herdar o arquivo
