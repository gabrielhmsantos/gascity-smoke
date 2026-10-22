# Fluxo de trabalho do rig `gascity-smoke`

Este repositório é um smoke test: `README.md` o identifica e `hello.txt` contém
`GAS CITY OK`. A estrutura está descrita em `docs/architecture.md`. Não há
aplicação, suíte de testes ou build neste rig. O Beads versionado em `.beads/`
usa o banco Dolt `gs` em modo `proxied-server`; os Beads de trabalho têm prefixo
`gs-` e ficam no banco do rig, não nos arquivos Git da documentação.
`docs/architecture.md` descreve um momento sem remoto Git; antes de publicar
uma branch, confirme o `origin` do checkout atual com `git remote -v`.

## Da tarefa ao Polecat

O Mayor pode criar um Bead diretamente no banco deste rig e encaminhá-lo ao
pool de Polecats com a fórmula de trabalho:

```sh
gc bd create "Descrever uma mudança" --rig gascity-smoke
# Após ler o ID retornado, atribua-o à variável WORK_ID:
gc sling gascity-smoke/gastown.polecat "$WORK_ID" --on mol-polecat-work
```

`WORK_ID` representa o ID `gs-...` retornado na criação. `city.toml` configura o
agente `gastown.polecat` deste rig com `mol-polecat-work` como fórmula padrão e
define `main` como branch padrão do rig. O roteamento com `--on` cria um
workflow e um convoy de entrada que acompanha o Bead de trabalho. A fórmula
materializa etapas próprias; o pool entrega uma etapa disponível a um
Polecat. O agente usa `gc hook --claim` para reivindicar uma etapa, lê sua
descrição com `gc bd show "$STEP_ID"` e encontra o Bead de trabalho pelo convoy
indicado nela (`gc convoy status "$CONVOY_ID" --json`). O Bead da etapa registra o
`assignee` e, após o claim, metadados de sessão como `gc.session_id`. Esses
valores identificam a execução atual, não uma configuração permanente do rig.

## Trabalho isolado e verificações

Na etapa de preparação, o Polecat busca `origin/main`, cria um worktree para
o Bead e usa a branch exclusiva `polecat/gs-<id>` baseada na referência remota
atualizada. A fórmula registra `metadata.work_dir` e `metadata.branch` no Bead
de trabalho para permitir retomada. Se houver `metadata.rejection_reason`, a
fórmula orienta a retomar a branch anterior, corrigir a rejeição e atualizar
sua base. O trabalho e o commit acontecem no worktree do Bead.

Para este repositório, uma alteração de documentação pode ser conferida com:

```sh
git status --short
git diff --check
test "$(cat hello.txt)" = 'GAS CITY OK'
git add docs/workflow.md
git diff --cached --check
git diff --cached --name-only
git diff --cached -- docs/workflow.md
```

Antes do commit, confira se somente o caminho esperado está preparado. Depois, compare
`git show --stat --oneline HEAD` com o escopo solicitado. A fórmula prevê
comandos de setup, typecheck, lint, build e testes quando configurados; os
valores resolvidos para este workflow estão vazios. Portanto, esses comandos
não representam verificações existentes neste rig.

O Polecat registra no Bead as verificações e o SHA do commit produzido, faz
push somente de `polecat/gs-<id>` e confirma que a referência remota aponta
para o mesmo SHA. O SHA desse commit pode mudar na integração por rebase; por
isso, ele é evidência da entrega do Polecat, enquanto `merged_sha` identifica
o commit integrado. O Polecat registra `metadata.target=main`, mantém
`metadata.branch` e reatribui o Bead aberto a
`gascity-smoke/gastown.refinery`. Ele não fecha o Bead de implementação nem
faz push direto para `main`.

## Refinery e resultado no Bead

A fórmula `mol-refinery-patrol` do pacote Gastown lê a branch e o destino do
Bead, atualiza a base, executa as verificações configuradas e processa a
integração. A estratégia padrão é `direct`: após confirmar que a branch traz
uma alteração real, o Refinery faz merge fast-forward e push para `main`,
verifica a referência remota e então grava `metadata.merge_result=merged`,
`metadata.merged_sha` e `metadata.merged_target=main`, fechando o Bead com a
razão do merge. Se houver conflito ou falha nas verificações, ele pode devolver
o trabalho ao pool com `metadata.rejection_reason`. A fórmula também admite
estratégias de pull request por metadado; não são o caminho padrão descrito
para esta entrega.

Consulte o registro durável e confirme a integração, sem presumir que uma
branch publicada já chegou a `main`. `MERGED_SHA` representa o valor de
`metadata.merged_sha` do Bead fechado:

```sh
gc bd show "$WORK_ID" --json
git fetch origin main
git log --oneline origin/main -5
git show --stat "$MERGED_SHA"
```

No JSON do Bead, confira `assignee` e os metadados de executor/sessão
registrados durante a execução, `metadata.branch`, o SHA do commit entregue
nas notas ou metadados, e depois `status`, a razão do fechamento e os metadados
`merge_result`, `merged_sha` e `merged_target`. Os nomes e IDs concretos de
sessão, branch e commit variam a cada trabalho. O fechamento e o SHA integrado
são responsabilidade do Refinery.

| Evidência | Onde consultar |
| --- | --- |
| Executor e sessão | `assignee` e `gc.session_id` na etapa reivindicada; notas ou metadados de execução no Bead de trabalho. |
| Branch e destino | `metadata.branch` e `metadata.target` no Bead de trabalho. |
| Commit entregue | SHA registrado pelo Polecat no Bead e `git show` na branch publicada. |
| Resultado da integração direta | `status=closed`, razão do fechamento, `metadata.merge_result`, `metadata.merged_sha` e `metadata.merged_target` no Bead de trabalho. |
