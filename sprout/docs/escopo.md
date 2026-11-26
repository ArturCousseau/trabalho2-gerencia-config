# Sprout: escopo do projeto

Mini clone do Scion, o sistema de controle de versão interno da Anthropic, feito
para a demonstração do Trabalho 2 de Gerência de Configuração (ENS4000, UCS).

Este documento define o projeto do zero. Não considera nenhuma versão ou
repositório anteriores.

## 1. Objetivo

Mostrar, rodando, **a diferença entre o git e o Scion quando muitos agentes
fazem commit ao mesmo tempo**.

O Sprout não pretende substituir o git nem ser um sistema de controle de versão
completo. Ele existe para uma única demonstração, e tudo o que não aparece nela
fica fora do escopo.

### A pergunta que a demo responde

> Dez agentes, cada um mudando um arquivo diferente, enviam commits ao mesmo
> tempo. Quanto trabalho é desperdiçado no git, e quanto no Sprout?

- **No git:** o push é rejeitado sempre que a `main` andou, mesmo com arquivos
  diferentes. O agente precisa buscar, refazer o commit (rebase) e tentar de novo.
- **No Sprout:** o servidor cria o changeset e, se os arquivos forem diferentes,
  só troca o pai e avança a `main` com um compare-and-swap. Nenhum agente precisa
  refazer nada.
- **Nos dois:** quando dois agentes mudam o mesmo arquivo, um deles é rejeitado.

## 2. O que o Sprout demonstra e o que não demonstra

| Demonstra                                             | Não demonstra                                       |
| ----------------------------------------------------- | --------------------------------------------------- |
| Commit feito no servidor, sem push                    | Escala horizontal (é um processo só, com SQLite)    |
| Changeset como diff em relação ao pai                 | Cache em camadas (daemon, cache regional, servidor) |
| Objetos imutáveis e content-addressed                 | Checkout virtualizado                               |
| `main` avançada por compare-and-swap, sem lock        | Blame, histórico por caminho, skip lists            |
| Reparent automático quando os arquivos são diferentes | Merge ou rebase de conteúdo                         |
| Derived data recursiva (só a árvore de arquivos)      | Code review, permissões, autenticação               |

**Atenção:** o artigo da Anthropic não detalha o que o Scion faz quando o
compare-and-swap da `main` falha. A política de reparent do Sprout é uma decisão
de projeto nossa, inspirada no artigo.

## 3. Princípios de arquitetura

Estes princípios são a base do `CLAUDE.md` do repositório. Qualquer pedido que
pareça exigir quebrar um deles deve parar e ser discutido antes.

1. **O servidor é dono de tudo.** Changesets, árvores e decisões de commit são
   calculados no servidor. Os agentes da demo só enviam pedidos e mostram
   respostas.
2. **Não existe push.** O agente envia os arquivos alterados, e o servidor cria
   o changeset. Não existe commit local nem fila de envio.
3. **Changeset é diff em relação ao primeiro pai**, não uma foto da árvore:
   `{ id, parent, author, message, timestamp, changes: [{ path, blobHash | null }] }`.
   `null` significa arquivo removido. O `id` é um inteiro sequencial atribuído
   pelo servidor.
4. **Objetos são imutáveis e content-addressed.** Blobs são identificados pelo
   SHA-256 do conteúdo e guardados comprimidos com zlib. Um objeto existente
   nunca é sobrescrito.
5. **Derived data é recursiva.** A árvore de um changeset é a árvore do pai com
   as mudanças aplicadas, calculada sob demanda e guardada em cache.
6. **Só existe uma ref, a `main`**, atualizada com compare-and-swap. Não existem
   branches nem tags dentro do Sprout.
7. **Commits sem lock.** O gatilho é o `parent` desatualizado do pedido, não
   uma falha do compare-and-swap. Se nenhum caminho do pedido foi alterado
   depois do `parent`, o servidor troca o pai para a cabeça, num salto só (no
   máximo uma troca por pedido); se algum foi, rejeita com `409`. Sem laço de
   tentativas. Nada de lock, fila ou merge de conteúdo (ADR 0003).

## 4. Componentes

```
sprout/
├── src/
│   ├── core/       # tipos do domínio, hashing, normalização de caminhos
│   └── server/     # Fastify + SQLite + blob store em disco
├── demo/           # agentes do git e do Sprout, placar, cenários
├── docs/
│   ├── adr/        # decisões de arquitetura
│   └── resultados.md
├── .github/        # CI, templates de issue e PR
├── CLAUDE.md
├── CHANGELOG.md
├── CONTRIBUTING.md
└── README.md
```

### Um pacote só, não um monorepo

Com o escopo reduzido, o projeto tem um único pacote Node, com pastas separadas
para o core, o servidor e a demo. Um monorepo com vários pacotes exigiria ordem
de build entre eles e project references do TypeScript, uma complexidade que não
se paga neste tamanho. Essa decisão vai para o ADR 0001.

### Stack

- Node (versão LTS fixada em `.node-version`) + TypeScript strict
- pnpm
- Fastify (servidor HTTP)
- better-sqlite3 (metadados: changesets e a ref `main`)
- Blobs em arquivos no disco
- vitest (testes), ESLint + Prettier
- tsx para rodar os scripts da demo

## 5. API do servidor

| Método e caminho           | O que faz                                                       |
| -------------------------- | --------------------------------------------------------------- |
| `PUT /blobs/:hash`         | Grava um blob. Rejeita se o hash do corpo não bater com `:hash` |
| `GET /blobs/:hash`         | Devolve o conteúdo de um blob                                   |
| `POST /changesets`         | Cria um changeset e avança a `main` (com reparent)              |
| `GET /changesets/:id`      | Devolve um changeset                                            |
| `GET /refs/main`           | Devolve o id da cabeça da `main`                                |
| `GET /changesets/:id/tree` | Devolve a árvore de arquivos (derived data)                     |

### `POST /changesets` em detalhe

Corpo: `{ parent, author, message, changes }`.

1. Valida o pedido: `changes` não vazio, caminhos válidos, sem caminho repetido,
   todos os blobs já gravados, `parent` existente.
2. Lê a cabeça atual da `main`.
3. Junta os caminhos alterados pelos changesets depois de `parent` até a
   cabeça (inclusive). Se `parent` é a cabeça, a lista é vazia.
4. **Algum caminho do pedido na lista:** responde `409` com `conflicts`, os
   caminhos em conflito. Nada é gravado.
5. **Nenhum caminho em comum:** cria o changeset com a cabeça como pai e avança
   a `main` com compare-and-swap. Se `parent` não era a cabeça, isso é a troca
   de pai, num salto só. Os passos 2 a 5 rodam sem `await` no meio, então o
   compare-and-swap não falha num processo só; se falhar, é bug do servidor
   (`500`).

Respostas: `201` (criado, possivelmente com o pai trocado), `400` (pedido
inválido ou blob ausente), `404` (`parent` não existe), `409` (conflito, com
`conflicts`), `500` (bug do servidor).

### `GET /changesets/:id/tree` em detalhe

Devolve a árvore de arquivos do changeset `:id`: a árvore do pai com as
`changes` do changeset aplicadas (princípio 5), calculada sob demanda e
guardada em cache (ADR 0005).

Corpo do `200`: `{ id, entries: [{ path, blobHash }] }`, com `entries`
ordenadas por `path` e sem os arquivos removidos. A raiz (id 1) tem `entries`
vazia. Remover um caminho que não está na árvore do pai não muda a árvore: o
`POST /changesets` aceita essa remoção, e o cálculo a ignora.

Respostas: `200` (a árvore), `400` (`:id` não é um inteiro decimal positivo
sem zero à esquerda, ou passa de `Number.MAX_SAFE_INTEGER`; a mesma validação
do `GET /changesets/:id`), `404` (changeset não existe), `414` (`:id` com mais
de 100 caracteres; quem responde é o roteador do Fastify, antes do handler,
com `error: "Bad Request"` e `code: "FST_ERR_MAX_PARAM_LENGTH"`).

## 6. A demo

Detalhada em `demo.md`. Em resumo:

- **Agentes do git:** repositório central local; cada agente clona, muda um
  arquivo, faz commit e push. Se o push for rejeitado, faz `pull --rebase` e
  tenta de novo.
- **Agentes do Sprout:** cada agente lê a `main`, grava o blob e envia o
  changeset.
- **Modos:** passo a passo (2 agentes, com pausa a cada etapa) e rápido
  (N agentes ao mesmo tempo).
- **Cenários:** arquivos diferentes e mesmo arquivo.
- **Placar:** aceitos, rejeições, tentativas e tempo total, para os dois sistemas.

## 7. Prioridades

Legenda: 🟢 essencial · 🟡 desejável · 🔵 extra

- 🟢 Servidor com blobs, changesets, compare-and-swap e reparent
- 🟢 Agentes do git e do Sprout, modos passo a passo e rápido
- 🟢 Placar comparativo e cenário de conflito
- 🟢 Resultados medidos em `docs/resultados.md`
- 🟢 Vídeo de reserva da demo
- 🟡 Endpoint da árvore (derived data) e verificação do estado final da `main`
- 🟡 Saída colorida e legível no terminal
- 🔵 Comparação de bytes: checkout sparse no Sprout × clone no git

**Fora de escopo, de propósito:** CLI completa, interface web, cache local,
blame, histórico por caminho, branches, merge e rebase de conteúdo,
autenticação.

## 8. Gerência de configuração no repositório

| Conceito                | Como aparece                                                         |
| ----------------------- | -------------------------------------------------------------------- |
| Itens de configuração   | Código, ADRs, scripts da demo, `.node-version`, `pnpm-lock.yaml`     |
| Controle de mudanças    | Issue → branch → PR com template → CI → autorrevisão → squash merge  |
| Baselines               | Tags semver em cada milestone                                        |
| Rastreabilidade         | Conventional Commits; commit na `main` → PR → issue (`Closes #N`)    |
| Registro de status      | `CHANGELOG.md` mantido à mão + GitHub Releases                       |
| Auditoria e verificação | CI (lint, typecheck, testes) em todo PR; branch protection na `main` |
| Decisões de projeto     | ADRs em `docs/adr/`                                                  |

### ADRs iniciais

- **0001:** pacote único, stack e versão fixa do Node
- **0002:** modelo de dados (changeset como diff, blobs content-addressed, id
  sequencial)
- **0003:** commits sem lock: compare-and-swap e política de reparent
- **0004:** SQLite num processo só e o que isso não demonstra

## 9. Limitações (vão para o README)

- **Um processo só, com SQLite:** as escritas acontecem em série. O Sprout mostra
  a separação entre criar o changeset e avançar a `main`, não a escala.
- **Sem merge de conteúdo:** dois agentes no mesmo arquivo geram rejeição.
- **Comparação local:** o repositório central do git roda na mesma máquina, sem
  rede. Isso favorece o git em tempo, o que torna a comparação conservadora.
- **Política de reparent própria:** o artigo não descreve esse comportamento.

## 10. Marcos

| Milestone | Conteúdo                                                         |
| --------- | ---------------------------------------------------------------- |
| `v0.1.0`  | Fundação: projeto, CI, processo de contribuição, ADRs            |
| `v0.2.0`  | Servidor: blobs, changesets, compare-and-swap, reparent, leitura |
| `v0.3.0`  | Demo: agentes, modos, cenários, placar, resultados               |
| `v1.0.0`  | Material final: README                                           |

Entrega: 26/11/2026.
