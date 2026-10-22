# Verificar uma entrega do rig `gascity-smoke`

Este rig é um smoke test: `README.md` identifica o projeto e `hello.txt` contém `GAS CITY OK`. `docs/architecture.md` descreve os arquivos versionados e a relação com Beads. Não há aplicação, suíte de testes ou pipeline de build no repositório. Na configuração atual da City, `city.toml` define `main` como branch padrão do rig e `mol-polecat-work` como fórmula de entrega dos polecats; os comandos de typecheck, lint, build e teste dessa instância estão vazios. Portanto, a verificação de uma mudança de documentação é uma revisão do Git, do conteúdo e do Bead, sem inventar um check de aplicação.

Os comandos abaixo consultam o estado sem alterar arquivos do projeto; `git fetch` atualiza apenas as referências remotas locais para conferir a integração. Execute-os no worktree da entrega e substitua `gs-EXEMPLO` pelo ID do Bead de trabalho. A descrição de arquitetura registra um estado anterior sem remote; confirme o estado atual com `git remote -v` em vez de presumir que ele continua igual.

## 1. Identificar worktree, branch e escopo

```sh
BEAD=gs-EXEMPLO
git rev-parse --show-toplevel
git worktree list
git status --short --branch
git branch --show-current
git remote -v
gc bd show "$BEAD"
```

O worktree da entrega deve ser isolado do checkout compartilhado do rig. A branch do polecat segue `polecat/<id-do-bead>`, criada a partir de `origin/main`; `git status` deve mostrar apenas os arquivos esperados para a tarefa. Compare a branch atual e o diretório com `metadata.branch` e `metadata.work_dir` do Bead. O checkout compartilhado e os demais worktrees não devem conter alterações desta entrega.

Antes do commit, examine os nomes e o conteúdo alterado, inclusive arquivos novos:

```sh
git status --short
git diff --name-only
git diff --check
git diff --cached --check
git diff -- docs/verification.md
git diff --cached -- docs/verification.md
git ls-files --others --exclude-standard
```

Os comandos `git diff --check` e `git diff --cached --check` apontam erros de whitespace nas mudanças não preparadas e preparadas, respectivamente. Um arquivo novo aparece em `git ls-files --others --exclude-standard` até ser adicionado; leia seu conteúdo antes de aprovar o commit. Para esta entrega, o único caminho esperado é `docs/verification.md`.

## 2. Conferir commit e branch publicada

Depois do commit, use o SHA completo para fixar exatamente o artefato revisado:

```sh
SHA=$(git rev-parse HEAD)
BRANCH=$(git branch --show-current)
git status --short --branch
git show --format=fuller --stat "$SHA"
git diff-tree --no-commit-id --name-only -r "$SHA"
git show --check --oneline "$SHA"
git diff --name-only origin/main...HEAD
git diff --check origin/main...HEAD
git log --oneline origin/main..HEAD
git ls-remote --heads origin "$BRANCH"
```

Confira o autor, a mensagem e o SHA em `git show`. Para esta tarefa, `git diff-tree` e `git diff --name-only` devem listar somente `docs/verification.md`; `git status` deve estar limpo, e os comandos `--check` não devem relatar erros. `git log` mostra os commits da entrega ainda ausentes de `origin/main`. O SHA retornado por `git ls-remote` para `refs/heads/$BRANCH` deve corresponder a `$SHA`; isso confirma a publicação da branch, sem fazer push para `main`.

## 3. Conferir o Bead e a integração

```sh
gc bd show "$BEAD"
gc bd show "$BEAD" --json | jq '.[0] | {status, assignee, metadata, notes}'
git fetch origin main
git log --oneline origin/main -- docs/verification.md
git show origin/main:docs/verification.md
```

No Bead, confira o executor e a sessão registrados, `metadata.branch`, o SHA do commit, as verificações e o resultado. Durante a entrega, a atribuição ao `gastown.refinery`, a branch publicada e `metadata.target=main` indicam que o trabalho aguarda revisão; isso ainda não prova integração. Após o Refinery revisar e fechar o Bead, confira o resultado registrado e o conteúdo do arquivo em `origin/main`. Se o Refinery tiver feito rebase, o SHA original pode não ser ancestral de `main`; use o histórico e o conteúdo da branch principal, junto com o registro de revisão do Bead, para confirmar que a alteração entrou. Se `origin/main` ainda não contiver o documento, a integração está pendente, independentemente do estado da branch do polecat.
