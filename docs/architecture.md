# Arquitetura do rig `gascity-smoke`

## Propósito

Este repositório é um projeto mínimo de smoke test do fluxo local do Gas City. Ele não contém uma aplicação, serviço, biblioteca ou pipeline de build. O sinal verificável do projeto é o texto `GAS CITY OK` em `hello.txt`; `README.md` identifica o repositório como "Gas City Smoke Test".

## Arquivos versionados

| Caminho | Papel |
| --- | --- |
| `README.md` | Identifica o projeto como smoke test. |
| `hello.txt` | Contém o marcador `GAS CITY OK`. |
| `.gitignore` | Exclui do Git arquivos locais do Dolt/Beads, bancos, locks e a chave de credenciais. |
| `.beads/README.md` | Guia genérico do Beads criado na inicialização; não descreve componentes adicionais deste projeto. |
| `.beads/config.yaml` | Modelo de configuração do Beads, com opções comentadas. |
| `.beads/metadata.json` | Registra o backend Dolt, o modo `proxied-server`, o banco `gs` e o identificador do projeto. |
| `.beads/.gitignore` | Exclui estado de execução do Beads, como sockets, locks, logs, backups e redirecionamentos de worktree. |
| `docs/architecture.md` | Descreve a estrutura e a verificação deste rig. |

O estado de execução ignorado pelo Git não faz parte do artefato versionado. A configuração versionada do Beads é apenas a estrutura inicial de rastreamento de tarefas; o repositório não define processos de aplicação.

## Relação com Gas City e Beads

Gas City usa este repositório como rig para delegar trabalho local. As tarefas são Beads com prefixo `gs-`, mantidas no armazenamento do rig; `gc bd show <id>` permite ler uma tarefa e `gc bd update` registra seu andamento e resultado. Um agente pode receber uma etapa de fórmula por `gc hook --claim`; a etapa aponta para um convoy que acompanha o Bead de trabalho. Neste trabalho, a fórmula `mol-do-work` usa esse fluxo para chegar ao Bead que solicita este documento.

O código e a documentação do projeto são versionados no Git. O encaminhamento de trabalho, os heartbeats e os resultados de Beads pertencem ao fluxo de coordenação do Gas City; não exigem alterações nos arquivos versionados de configuração do rig. Este repositório não tem remote Git configurado, portanto a entrega é um commit local em uma branch e um worktree disponíveis para integração local.

## Verificação

Na raiz do worktree, confira o conteúdo e os arquivos versionados:

```sh
cat README.md hello.txt
git ls-files
test "$(cat hello.txt)" = 'GAS CITY OK'
```

Para uma mudança de documentação, confira também o diff e se ele não contém erros de whitespace:

```sh
git diff --check
git diff -- docs/architecture.md
git status --short
```

Depois do commit, `git show --stat --oneline HEAD` deve listar somente `docs/architecture.md` para esta mudança. Não há suíte de testes ou build no repositório.
