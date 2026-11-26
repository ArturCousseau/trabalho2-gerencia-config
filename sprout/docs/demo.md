# A demo: git × Sprout com agentes concorrentes

Funcionamento detalhado e comandos da demonstração.

## 1. Objetivo

Mostrar, com uma execução real, **quanto trabalho é desperdiçado** quando vários
agentes fazem commit ao mesmo tempo no git, e como a abordagem do Scion (commit
no servidor, compare-and-swap e troca de pai) evita esse desperdício.

### A mensagem em uma frase

> No git, o push é rejeitado sempre que a `main` andou, mesmo com arquivos
> diferentes. No Sprout, o servidor só rejeita quando há conflito de verdade.

### O que a demo não afirma

- **Não afirma que o git "falha".** Com retry, todos os commits acabam entrando
  no git. O que a demo mede é o **trabalho desperdiçado**: rejeições, rebases,
  tentativas e tempo.
- **Não demonstra escala.** O Sprout roda num processo só, com SQLite. A demo
  mostra o mecanismo; o Scion real faz isso distribuído.
- **Não reproduz o comportamento exato do Scion.** O artigo não detalha o que o
  Scion faz quando o compare-and-swap falha. A política de trocar o pai é nossa.
- **Não usa agentes de IA.** Os agentes da demo são rotinas com roteiro fixo,
  sem modelo de IA, que simulam o padrão de commits concorrentes de agentes
  reais.

## 2. Os cenários

| Cenário | Agentes | Arquivos    | Modo                    | Para quê                                            |
| ------- | ------- | ----------- | ----------------------- | --------------------------------------------------- |
| A       | 2       | Diferentes  | Passo a passo           | Explicar o mecanismo                                |
| B       | 10      | Diferentes  | Rápido                  | Mostrar a escala do desperdício                     |
| C       | 2       | **O mesmo** | Passo a passo ou rápido | Mostrar que o Sprout também rejeita conflitos reais |

Cada cenário roda **primeiro no git, depois no Sprout**, cada um em tela cheia.
O placar comparativo aparece só no final.

### Por que não rodar um agente de cada vez

Se o agente 1 termina antes de o agente 2 começar, o agente 2 parte da `main` já
atualizada, e o git aceita tudo. A rejeição só acontece quando dois agentes
**partem do mesmo ponto**. Por isso a demo tem dois modos: o passo a passo, que
mostra o conflito em câmera lenta com dois agentes, e o rápido, que mostra a
escala com dez.

## 3. Como os agentes do git funcionam

### Preparação

Cada execução começa do zero, e uma execução anterior não muda o resultado:

1. Apaga o que sobrou da execução anterior: `demo/tmp/central.git`, os clones
   `demo/tmp/git-agente-*` e o JSON do cenário (§6).
2. Cria o repositório central **bare** em `demo/tmp/central.git`
   (`git init --bare -b main`), com `gc.auto=0`, para o git não começar uma
   coleta de lixo no meio da execução.
3. Um clone de preparação (`demo/tmp/git-preparacao`, que não é agente) faz o
   commit inicial com um `README.md` de conteúdo fixo e autor `preparacao`,
   envia para a `main` do central e é apagado.
4. Cada agente ganha o seu próprio clone (`demo/tmp/git-agente-1`, `-2` etc.),
   com a configuração local daquele agente (`localGitConfig` em `demo/git.ts`):
   autor `agente-N`, commit sem assinatura e `gc.auto=0`.

Todo processo `git` da demo roda com `gitEnv` (`demo/git.ts`), que esconde a
configuração global e a do sistema. Por isso a identidade de cada agente fica na
configuração do próprio clone.

### O ciclo de um agente

```
1. escreve o seu arquivo (agente-N.txt, ou conflito.txt no cenário C)
   com o conteúdo "agente N"
2. git add + git commit
3. ── ponto de sincronização: espera todos os agentes chegarem aqui ──
4. git push --porcelain
5. se o push for aceito: termina
6. se for rejeitado:
     conta uma rejeição do tipo dela (fetch-first ou ref-lock)
     git pull --rebase
     se o rebase tiver conflito: git rebase --abort, conta como conflito
       real e termina
     volta ao passo 4
```

O **ponto de sincronização** (passo 3) é o que garante que todos os agentes
partem da mesma `main`. Sem ele, os agentes terminariam em momentos diferentes e
o resultado mudaria a cada execução.

Os passos 1 e 2 não rodam em paralelo: o harness faz o commit de todos os
agentes, um depois do outro, antes de criar a barreira. O resultado é o mesmo de
commits simultâneos, porque nenhum agente envia nada antes da barreira e todos
os commits ficam em cima do commit inicial. O motivo é a falha: se um commit
falhasse com outros agentes já parados na barreira, eles esperariam para sempre
por uma chegada que nunca vem, e a execução não terminaria. Com os commits antes
da barreira, uma falha nessa fase para a execução antes de qualquer agente
esperar.

### O ponto de sincronização

É uma barreira em memória (`demo/barrier.ts`): todos os agentes rodam no mesmo
processo Node, e cada um chama `arrive()` e espera. Quando o N-ésimo chega, a
barreira lê `performance.now()` uma vez e libera todos com esse mesmo instante,
o **instante da liberação**. É dele que o tempo total é medido (§6). Uma chegada
além de N é erro do harness.

### Os dois tipos de rejeição

No git, um push concorrente pode falhar de duas formas, e as duas contam como
rejeição. O código de saída é 1 nas duas, então o harness separa os tipos pela
linha de status do `git push --porcelain` (`classifyPush` em `demo/git.ts`):

| Tipo no registro | Quando acontece                                                   | Resumo na linha de status                          |
| ---------------- | ----------------------------------------------------------------- | -------------------------------------------------- |
| `fetch-first`    | A `main` andou antes do push: não é _fast-forward_                | `[rejected] (fetch first)`                         |
| `ref-lock`       | Dois pushes disputaram a ref ao mesmo tempo, e este perdeu o lock | `[remote rejected] (incorrect old value provided)` |

As linhas reais, com os campos separados por TAB (flag, refs, resumo), vêm do
git 2.56.0 e estão em `demo/fixtures/porcelain/`:

```
!	refs/heads/main:refs/heads/main	[rejected] (fetch first)
!	refs/heads/main:refs/heads/main	[remote rejected] (incorrect old value provided)
```

O push aceito sai com código 0 e flag ` `, `+` ou `*`. Qualquer outra saída
(resumo desconhecido, nenhuma ou mais de uma linha para `refs/heads/main`,
código que não bate com a flag) é erro do harness, não métrica.

### O `pull --rebase` e o conflito real

Depois de uma rejeição, o harness separa os casos do `git pull --rebase` pelo
código de saída e pela pasta `.git/rebase-merge`, que indica rebase em andamento
(`classifyPull` em `demo/git.ts`). O código sozinho não basta:

| Código                    | `.git/rebase-merge` | Resultado                                                             |
| ------------------------- | ------------------- | --------------------------------------------------------------------- |
| 0                         | ausente             | Rebase feito: o agente volta ao push                                  |
| 1                         | presente            | Conflito real: `git rebase --abort` e `outcome: "conflict"`           |
| qualquer outra combinação |                     | Erro do harness (por exemplo: 0 com a pasta presente, 1 sem ela, 128) |

O `git rebase --abort` desfaz o rebase interrompido, e o clone do agente que
desistiu fica com `git status` limpo. Na demo, só o cenário C chega a esse caso.

### Erro do harness

Saída inesperada do git (push fora do que "Os dois tipos de rejeição" descreve,
`pull --rebase` fora dos dois primeiros casos da tabela acima, comando de
preparação ou de commit que falhou) e teto de tentativas ultrapassado são erros
do harness, não métrica. Quando um agente falha assim, os outros não são
interrompidos: o harness espera todos terminarem (`Promise.allSettled`), para
não deixar processos `git` ainda escrevendo em `demo/tmp`. Depois disso, **não
grava o JSON**, escreve no stderr a mensagem de cada agente que falhou e sai com
**código 1**. Quando o git terminou com saída inesperada, a mensagem leva o
stderr do git, o único lugar onde ele aparece; no terminal, a execução normal
mostra só as linhas de evento e o resumo. Se o git nem chegou a rodar até o fim
(não foi possível iniciar o processo, ou ele foi morto por um sinal), não há
stderr para mostrar: a mensagem traz o comando e o motivo (`cannot start git`
ou `killed by <sinal>`).

A `main` do `central.git` diferente do esperado na conferência do fim (§6)
também é erro do harness, e a saída do `git ls-tree` ou do `git rev-list
--count` fora do formato nessa leitura também. A conferência roda depois de
todos os agentes terminarem, então a mensagem é uma só (`main check: …`), sem
JSON e com código 1.

### O teto de tentativas

Antes de cada push, o harness soma 1 ao total de pushes da execução (todos os
agentes juntos) e confere o teto de **N(N+1)/2** (`checkGitAttempts` em
`demo/git.ts`): 3 nos cenários A e C, 55 no B. Passar dele é erro do harness.

O teto vem do pior caso legítimo. Uma rejeição contada (`fetch-first` ou
`ref-lock`) só acontece quando outro agente foi aceito depois da base atual do
agente, e o `pull --rebase` seguinte traz esse aceite para a base. Então o
agente aceito em k-ésimo lugar faz no máximo k pushes, e a execução faz no
máximo 1 + 2 + … + N = N(N+1)/2. O teto era 3 × N, mas a primeira execução real
do cenário B passou de 30 tentativas sem nada de errado, e o teto mudou
(comentário na #15).

A frase "uma rejeição contada só acontece quando outro agente foi aceito" é uma
**premissa, não verificada no código do git**: nada no harness a garante. O
`classifyPush` só conta os dois resumos da tabela acima, e qualquer outra saída
do push é erro do harness; a premissa é que esses dois só aparecem depois de um
push aceito de outro agente. Ela é
sustentada pela E1 da #14 (225 de 225 rejeições por `ref-lock`, sempre com 1
aceito por onda) e pela coleta da D9 na #15 (20 execuções do cenário B, nenhuma
acima de 55). Se ela falhar, o teto dispara e a execução para com erro do
harness, em vez de contar errado.

**As ondas do cenário B.** No pior caso, a cada onda, todos os agentes que
ainda faltam empurram juntos e só um é aceito: o k-ésimo aceito faz k
tentativas, e a execução chega a N(N+1)/2 = 55. As 20 execuções da coleta da
#15 deram exatamente 55 tentativas cada. No terminal da apresentação (zsh, node
26.13), as 10 execuções de 18/11/2026 (5 do `demo:git` e 5 do `demo:medir`)
também deram 55. Numa execução de teste do `demo:medir` dentro do Claude Code,
com node 26.12 e a máquina mais carregada, o git B deu de 51 a 55 tentativas. A
causa dessa diferença não foi verificada. Como nenhum agente passa de k, um
total de 55 só acontece quando cada um faz exatamente k. O teste do cenário B
confere total ≤ 55, e não = 55: o teste só confere invariantes, e nada no git
obriga cada onda a reunir todos os agentes restantes.

### A largada não sai junta

A barreira libera todos os agentes no mesmo instante, mas os pushes não começam
juntos. O `spawn` do Node é síncrono: cada chamada cria o processo `git push`
antes de devolver e custa cerca de 3,7 ms. Na Fase 1 da #14 (experimento E3, 10
rodadas com 10 agentes), a primeira chamada saiu 0,02–0,07 ms depois da
liberação, mas as 10 chamadas se espalharam por 27–57 ms (mediana 31,6 ms).
Lançando os pushes de um script shell (E1), o espalhamento foi de 2,3 ms
(mediana).

- **A divisão das rejeições muda.** Na E3, cerca de metade foi `fetch-first` e
  metade `ref-lock` (46 e 44 de 90); na E1, todas as 225 foram `ref-lock`. Os
  primeiros pushes disputam o lock; os que saem depois de a `main` já ter mudado
  recebem `fetch first`.
- **A mensagem da demo não muda.** Cada onda de pushes ainda teve exatamente 1
  aceito e o resto rejeitado.
- **O tempo total do git inclui esses ~30 ms de lançamento**, que o Sprout não
  tem: o agente do Sprout faz um pedido HTTP, sem criar processo. É pouco perto
  do tempo total, mas é uma assimetria, e fica declarada aqui.

Decidimos aceitar e documentar, em vez de mudar a forma de lançar os pushes.

## 4. Como os agentes do Sprout funcionam

### Preparação

Cada execução começa do zero, como no git:

1. Apaga o que sobrou da execução anterior: o diretório de dados
   `demo/tmp/sprout-data` e o JSON do cenário (§6).
2. Sobe o servidor como processo filho da demo (`startSproutServer` em
   `demo/sprout-server.ts`), com `SPROUT_DATA_DIR=demo/tmp/sprout-data` e
   `PORT=0`. Ao abrir, o servidor recria o diretório de dados, o changeset raiz
   e a `main` apontando para ele.
3. Espera a linha `sprout server listening on <endereço>` no stdout do servidor
   e manda todos os pedidos para esse endereço. Com `PORT=0`, o sistema escolhe
   uma porta livre, e a linha traz a porta escolhida.

### O servidor como processo filho

O comando do filho é o próprio Node, com o tsx carregado por `--import`:
`process.execPath --import tsx src/server/main.ts`, a partir da raiz do
repositório. Não é o `pnpm start` nem o atalho `node_modules/.bin/tsx`: com o
atalho, o servidor roda num processo neto, e um SIGKILL no processo que a demo
conhece deixa o neto vivo, segurando a porta (experimento E2′ da Fase 1 da
#16). Com `--import`, o servidor é um processo só (E1), e o PID que a demo
conhece é o dele.

O stdout e o stderr do servidor chegam à demo por pipe. O stdout continua sendo
lido, e descartado, depois da linha de "listening": um pipe que ninguém lê
enche, e o servidor travaria ao escrever. O stderr só aparece dentro da
mensagem de erro, quando o servidor não fica pronto.

**Prontidão.** A demo espera a linha de "listening" por até 10 s
(`READY_TIMEOUT_MS`). Na Fase 1 da #16, em 10 medições no omarchy, a linha
apareceu entre 586 e 732 ms depois do início; a folga é para máquinas mais
lentas, como o runner do CI, que não foi medido. Servidor que sai antes da
linha, ou que passa do prazo, é erro do harness; no segundo caso, a demo mata o
servidor com SIGKILL e espera ele sair antes de parar.

**Encerramento.** No fim da execução, com sucesso ou com erro, a demo manda
SIGTERM ao servidor e espera ele sair por até 5 s (`STOP_TIMEOUT_MS`). Se ele
não sair, manda SIGKILL, espera, e a execução termina com erro do harness. O
servidor é encerrado **antes** de gravar o JSON: um servidor que não para dá
código 1 sem JSON. Quando a execução já tinha falhado, o erro dela é o que
aparece, e uma falha do encerramento é ignorada. Nos dois casos, nenhum
processo do servidor sobra: o encerramento só termina depois de o processo
sair.

**Ctrl+C.** Aqui está só o que foi observado. No experimento E5 da Fase 1 da
#16 (omarchy), com o servidor criado direto pelo `spawn` do Node e o stdout e o
stderr em pipe (a configuração da demo), o Ctrl+C terminou o processo que criou
o servidor e o próprio servidor, e nenhum processo sobrou. No modo rápido, a
cadeia completa do comando, `pnpm` → `tsx` → demo → servidor, **não foi
testada** (a execução termina antes de dar tempo).

No passo a passo (§5), a cadeia completa foi testada, porque a demo fica parada
esperando o Enter. A pausa lê o stdin com o `readline` em `terminal: false`, sem
handler de `SIGINT`. No experimento E3 da Fase 1 da #17 (omarchy, num
pseudo-terminal, node 26.12.0), com o padrão (`terminal: true` quando o stdout
é um terminal), o `readline` liga o modo raw. O Ctrl+C vira uma tecla, não gera
sinal, e a demo fica pendurada com o servidor vivo. Com `false`, o terminal
fica no modo normal e o Ctrl+C manda SIGINT para o grupo inteiro: o `pnpm`, a
demo e o servidor morrem pelo sinal, nenhum processo sobra e a porta fecha.
Nenhum teste automático vê isso (no vitest o stdin nunca é um terminal). A
verificação manual da #17, no zsh do omarchy com node 26.13.0, confirmou com o
comando da apresentação: Ctrl+C numa pausa do
`pnpm demo:sprout --cenario A --passo` e do
`pnpm demo:git --cenario C --passo`, o shell mostra o fim por SIGINT e nenhum
`tsx`, servidor ou `git` sobra. Com o servidor morto pelo sinal, o
encerramento acima (SIGTERM, espera de 5 s) não roda. Se o stdin fecha numa
pausa, a execução para com erro do harness, encerra o servidor normalmente e
sai com código 1 (§8). Um caso fica sem tratamento na demo: um SIGKILL no
processo do `tsx` deixa a demo e o servidor órfãos, com o servidor ainda
respondendo (E5b da Fase 1 da #17). O `tsx` não leva o neto junto, e nenhum
código da demo roda depois de um SIGKILL. Isso vai para o README (#23).

### O ciclo de um agente

```
1. GET /refs/main                     → guarda o id como parent
2. calcula o hash do conteúdo do arquivo
3. PUT /blobs/:hash                   → grava o conteúdo
4. ── ponto de sincronização: espera todos os agentes chegarem aqui ──
5. POST /changesets { parent, author, message, changes }
6. 201: termina (aceito, possivelmente com o pai trocado pelo servidor)
   409: conta como conflito real e termina
```

Os passos 1 a 3 não rodam em paralelo, pelo mesmo motivo do git (§3): o
harness faz o GET, o hash e o PUT de todos os agentes, um depois do outro, antes
de criar a barreira, e uma falha nessa fase para a execução antes de qualquer
agente esperar. O resultado é o mesmo de pedidos simultâneos: nenhum agente
envia changeset antes da barreira, então todos leem a mesma `main`, o changeset
raiz.

Depois da barreira, cada agente faz um único `POST /changesets`, sem nenhum
`await` entre a liberação e o envio (`runAgent` em `demo/sprout-agents.ts`): a
demo chama o `fetch` de todos os agentes antes de a primeira resposta chegar.

O agente do Sprout **não faz retry**: quem resolve a troca de pai é o servidor,
dentro do `POST /changesets`, no máximo uma vez por pedido e direto para a
cabeça. O agente detecta a troca comparando o `parent` da resposta `201` com o
que enviou, e isso entra no placar.

### O registro

Cada agente gera um registro no formato do §6:

- `outcome`: `"accepted"` no `201`, `"conflict"` no `409`.
- `attempts` é sempre 1 e `rejections` é sempre `{}`: um pedido por agente, sem
  retry.
- `reparented`: no `201`, se o `parent` da resposta é diferente do enviado; no
  `409`, `false`.
- `endMs`: do instante da liberação (§3) até a resposta ser lida e validada,
  como no git.

### Erro do harness

Como no git (§3), estes casos são erro do harness, não métrica:

- status fora do esperado (`checkStatus` em `demo/sprout-responses.ts`): `200`
  no `GET /refs/main`, `201` ou `200` no `PUT /blobs/:hash`, `201` ou `409` no
  `POST /changesets`, `200` no `GET /changesets/:id/tree`;
- corpo de resposta fora do formato: `id` que não é inteiro no
  `GET /refs/main`, `parent` que não é inteiro no `201`, `conflicts` que não é
  uma lista de textos não vazia no `409`, corpo que não é JSON; no
  `GET /changesets/:id/tree`, `id` diferente do pedido, `entries` que não é uma
  lista de `{ path, blobHash }` com textos, ou caminho repetido;
- `main` diferente do esperado na conferência do fim (§6);
- servidor inacessível (o `fetch` falhou: conexão recusada ou reiniciada) ou que
  morreu durante a execução;
- servidor que não fica pronto ou não para (acima).

O tratamento também é o do git: o harness espera todos os agentes terminarem
(`Promise.allSettled`), **não grava o JSON**, escreve no stderr a mensagem de
cada agente que falhou e sai com **código 1**. Uma falha antes da barreira para
a execução ali, com uma mensagem só. A conferência da `main` roda com o
servidor ainda de pé; quando ela falha, o servidor é encerrado do mesmo jeito
que nos outros erros, e a mensagem é uma só (`main check: …`).

## 5. Os modos

Nenhuma linha dos dois modos passa de 111 colunas, para caber na tela da
apresentação. A largura é a visível, sem os códigos de cor, e os símbolos
`✓ ✗ ━ · →` ocupam uma coluna cada. A exceção é a linha `resumo:` do Sprout
com tempo total a partir de 1000 ms, que passa de 111. A gravação com o VHS
(`Set FontSize 26`, `Set Width 1920`) tem 114 colunas.

As linhas saem coloridas quando a saída vai para um terminal (§8):

- em cinza: o prefixo `[git]`/`[sprout]`, o `→`, os hashes do git, o
  `(pai = 1)` do Sprout e o tipo da rejeição (`(fetch-first)`);
- em verde: `✓ aceito` e, na linha da conferência da `main` (§6), o
  `✓ 2 commits aceitos` inteiro;
- em vermelho: `✗ REJEITADO` e `✗ rejeitado`;
- em amarelo: `✗ CONFLITO` e `✗ conflito`;
- em azul: `pai trocado`;
- o título de cada execução tem os `━━` em cinza e o texto do meio em
  negrito; o separador do modo rápido sai inteiro em cinza.

O resto da linha (ids, nomes de arquivo, o `ok` do `pull --rebase`, o
`na main (arquivos, conteúdo e contagem)`) fica sem cor. Sem cor, o texto é o
mesmo, caractere por caractere.

### Passo a passo (cenários A e C)

`pnpm demo:git --cenario A|C --passo` e
`pnpm demo:sprout --cenario A|C --passo` (#17). A preparação roda logo no
início, sem pausa e sem linha própria: no git, os repositórios e os commits
dos dois agentes (§3); no Sprout, o servidor e o `GET` e o `PUT` de cada
agente (§4). Depois vêm duas linhas: o título da execução
(`━━ git · cenário A · passo a passo ━━`) e o aviso
`[git] Enter antes de cada etapa`. Então o script pausa antes de cada etapa,
inclusive a primeira, esperando um **Enter** do apresentador. Depois da última
etapa não há pausa: a execução termina. Cada etapa é uma operação real e
imprime uma linha, com o contador da etapa (`2/4` no git, que tem quatro
etapas; `2/3` no Sprout, que tem três) e o porquê dentro dela. Os valores vêm
da autoridade: no git, o hash de 7 caracteres da `main` do repositório central
(`git rev-parse --short=7`); no Sprout, o id da `main` lido com
`GET /refs/main` depois de cada etapa. Os hashes abaixo são exemplos.

Cenário A no git:

```
━━ git · cenário A · passo a passo ━━
[git] Enter antes de cada etapa
[git] 1/4 agente 1 e agente 2 partem da main no commit 3f2a1c9
[git] 2/4 agente 1: push                    → ✓ aceito, main = 9b81e04
[git] 3/4 agente 2: push                    → ✗ REJEITADO (a main andou; o git não olha os arquivos)
[git] 4/4 agente 2: pull --rebase + push    → ✓ aceito, main = 4c7d22a
```

No cenário C, o título diz `cenário C`, e a última linha do git é o conflito
real, com o `rebase --abort` (§3):

```
[git] 4/4 agente 2: pull --rebase           → ✗ CONFLITO em conflito.txt; rebase --abort, main = 9b81e04
```

Cenário A no Sprout:

```
━━ sprout · cenário A · passo a passo ━━
[sprout] Enter antes de cada etapa
[sprout] 1/3 agente 1 e agente 2 partem da main no changeset 1
[sprout] 2/3 agente 1: envia changeset (pai = 1)    → ✓ aceito, main = 2
[sprout] 3/3 agente 2: envia changeset (pai = 1)    → ✓ aceito, pai trocado para 2 (outro arquivo), main = 3
```

No cenário C, o título diz `cenário C`, e a última linha do Sprout é o `409`.
Com 111 colunas, é a linha mais larga da demo:

```
[sprout] 3/3 agente 2: envia changeset (pai = 1)    → ✗ CONFLITO (conflito.txt mudou depois do pai 1), main = 2
```

Num terminal, o Enter ecoa uma quebra de linha, então as etapas aparecem
separadas por uma linha em branco.

**Conferência contra o roteiro.** Antes de imprimir a linha, o script confere
que o resultado real é o do roteiro. No git: os dois agentes partem da `main`
do central, o push do agente 1 é aceito, o do agente 2 é rejeitado por
`fetch-first` (um `ref-lock` aqui é erro, porque não há disputa pelo lock), e o
`pull --rebase` termina limpo no A ou com conflito no C. No Sprout: os dois
partem da `main`, o agente 1 é aceito com o próprio pai, e o agente 2 tem o pai
trocado para o changeset do agente 1 no A ou recebe `409` no C. Na primeira
diferença, a execução para com erro do harness e sai com código 1. A linha
daquela etapa não aparece, e a mensagem diz a etapa, o esperado e o obtido. Por
exemplo:

```
demo:git: step 2 (agente 1: push): expected accepted, got fetch-first
```

No Sprout, o servidor é encerrado também nesse caminho (§4).

O passo a passo não imprime a linha `resumo:` e **não grava JSON**: apaga o do
cenário na preparação e não escreve outro (§6).

No modo passo a passo, os agentes não rodam ao mesmo tempo de verdade: o script
controla a ordem para que a turma veja cada etapa. A concorrência real fica para
o modo rápido.

### Rápido (cenário B)

Os N agentes rodam em paralelo, com o ponto de sincronização. A saída começa
com o título da execução, impresso depois da preparação e antes de a barreira
existir, portanto fora do tempo total (§6). Depois vem uma linha por evento
(pode rolar rápido, é o efeito desejado), cuja ordem varia de uma execução
para outra. No fim, a linha da conferência da `main` (§6), um separador de `━`
da largura do título e o resumo daquele sistema, com os mesmos rótulos e
valores do placar (§6). O separador e o resumo só aparecem quando a execução
termina; num erro do harness, nenhum dos dois sai. A linha da conferência só
aparece quando a `main` passou; numa divergência, ela também não sai. Um
trecho do git (os números variam de uma execução para outra):

```
━━ git · cenário B · modo rápido ━━
[git] agente 8: push → ✗ rejeitado (fetch-first)
[git] agente 7: push → ✗ rejeitado (fetch-first)
[git] agente 3: push → ✗ rejeitado (ref-lock)
[git] agente 2: push → ✗ rejeitado (ref-lock)
[git] agente 1: push → ✓ aceito
…
[git] agente 2: pull --rebase → ok
[git] agente 2: push → ✓ aceito
[git] main conferida → ✓ 10 commits aceitos na main (arquivos, conteúdo e contagem)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[git] resumo: aceitos 10, rejeições 45, tentativas 55, trocas de pai -, conflitos reais 0, tempo total 1232 ms
```

No cenário C, o conflito real do git sai assim, e a linha da conferência fica
no singular:

```
[git] agente 2: pull --rebase → ✗ conflito, rebase --abort
[git] main conferida → ✓ 1 commit aceito na main (arquivos, conteúdo e contagem)
```

No Sprout, cada agente imprime uma linha só, com o pai que enviou:

```
━━ sprout · cenário B · modo rápido ━━
[sprout] agente 1: changeset (pai 1) → ✓ aceito
…
[sprout] agente 10: changeset (pai 1) → ✓ aceito, pai trocado para 10
[sprout] main conferida → ✓ 10 commits aceitos na main (arquivos, conteúdo e contagem)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[sprout] resumo: aceitos 10, rejeições 0, tentativas 10, trocas de pai 9, conflitos reais 0, tempo total 121 ms
```

E, no cenário C, o `409`:

```
[sprout] agente 2: changeset (pai 1) → ✗ conflito (conflito.txt)
[sprout] main conferida → ✓ 1 commit aceito na main (arquivos, conteúdo e contagem)
```

Na linha do resumo, só o prefixo sai em cinza.

## 6. O placar

Aparece no final, com os dois sistemas lado a lado. Saída de
`pnpm demo:placar --cenario B`:

```
Cenário B: 10 agentes, arquivos diferentes

                       git    sprout
aceitos                 10        10
rejeições               45         0
tentativas              55        10
trocas de pai            -         9
conflitos reais          0         0
tempo total        1232 ms    121 ms

resultados: git 17/11/2026 22:17, sprout 17/11/2026 22:18
```

_(uma execução no omarchy em 17/11/2026; a média de várias execuções está em
[`docs/resultados.md`](resultados.md), §7)_

Num terminal (§8), o placar destaca a comparação. O cabeçalho
(`git`, `sprout`) sai em negrito. Nas linhas `rejeições`, `tentativas` e
`tempo total`, quando os dois valores mostrados são diferentes, o menor sai em
verde e o maior em vermelho. O tempo é comparado em milissegundos inteiros,
como aparece na tabela: `41 ms` contra `41 ms` é empate, mesmo que os valores
gravados sejam 40,6 e 41,4. Empates e as outras linhas (`aceitos`,
`trocas de pai`, `conflitos reais`) ficam sem cor, assim como o título
`Cenário…` e a linha `resultados:`. Os espaços de alinhamento ficam fora das
cores, e a tabela fica alinhada com ou sem elas.

O placar só lê os dois arquivos de resultados (abaixo) e calcula cada coluna
com o `summarize`: não roda git nem sobe servidor. O tempo total sai em
milissegundos inteiros, arredondado. A última linha traz a data de modificação
de cada arquivo, em hora local, para o apresentador ver se as duas execuções
são da mesma rodada.

### Definição de cada métrica

- **aceitos:** commits que entraram na `main`.
- **rejeições:** pushes recusados (git) ou pedidos recusados sem conflito real
  (Sprout; o esperado é zero).
- **tentativas:** total de pushes (git) ou de pedidos `POST /changesets` (Sprout).
- **trocas de pai:** quantas respostas `201` do Sprout vieram com um `parent`
  diferente do enviado. Cada pedido troca de pai no máximo uma vez, então, com
  10 agentes partindo da mesma cabeça, são no máximo 9 trocas. Mostra o
  trabalho que o servidor fez no lugar do agente.
- **conflitos reais:** mudanças no mesmo arquivo, rejeitadas nos dois sistemas.
- **tempo total:** do ponto de sincronização até o último agente terminar.

### O registro de cada agente

O placar sai só dos registros por agente (`AgentResult` em `demo/metrics.ts`):

| Campo        | Valor                        | Significado                                                                                               |
| ------------ | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `agent`      | inteiro ≥ 1                  | Número do agente                                                                                          |
| `outcome`    | `"accepted"` ou `"conflict"` | Aceito, ou conflito real (o agente desistiu)                                                              |
| `attempts`   | inteiro ≥ 1                  | Pushes (git) ou pedidos `POST /changesets` (Sprout)                                                       |
| `rejections` | objeto                       | Rejeições por tipo: chaves `fetch-first` e `ref-lock` (§3), valores inteiros ≥ 0. No Sprout é sempre `{}` |
| `reparented` | booleano ou `null`           | Sprout: se o `201` veio com um `parent` diferente do enviado. Git: sempre `null`                          |
| `endMs`      | número ≥ 0                   | Milissegundos do instante da liberação (§3) até o fim do agente                                           |

Cada coluna do placar é calculada desses registros (`summarize`): **aceitos** e
**conflitos reais** contam o `outcome`; **rejeições** somam todos os tipos de
todos os agentes; **tentativas** somam `attempts`; **trocas de pai** contam os
`reparented: true` (no git, onde todos são `null`, aparece `-`); **tempo total**
é o maior `endMs`.

### O arquivo de resultados

Cada execução grava um JSON em `demo/tmp/resultados/<sistema>-<cenário>.json`
(por exemplo, `git-B.json`), com o sistema, o cenário e um registro por agente.
Exemplo ilustrativo do cenário A no git:

```json
{
  "system": "git",
  "scenario": "A",
  "agents": [
    {
      "agent": 1,
      "outcome": "accepted",
      "attempts": 1,
      "rejections": {},
      "reparented": null,
      "endMs": 41.2
    },
    {
      "agent": 2,
      "outcome": "accepted",
      "attempts": 2,
      "rejections": { "ref-lock": 1 },
      "reparented": null,
      "endMs": 97.8
    }
  ]
}
```

A leitura (`readResult` em `demo/results.ts`) confere a forma campo a campo e
falha com uma mensagem que nomeia o arquivo e, quando é o caso, o campo: arquivo
ausente, JSON inválido, campo desconhecido ou de tipo errado, tipo de rejeição
fora da lista, `reparented` diferente de `null` no git ou diferente de booleano
no Sprout, número de agentes diferente do cenário, agente repetido ou faltando.
Como o arquivo fica em `demo/tmp`, o `pnpm demo:reset` também o apaga.

No git e no Sprout, o JSON do cenário é apagado no início de cada execução
(preparação do §3 e do §4). Uma execução que para com erro do harness não deixa
arquivo nenhum, então o placar nunca lê um resultado antigo do mesmo sistema
depois de uma execução que falhou. O passo a passo (§5) também apaga o JSON do
cenário e nunca grava outro: um `--passo` do C apaga o resultado de uma
execução rápida do C, e o placar do C pede para rodar de novo. Mas os dois
arquivos são independentes: se só um dos sistemas rodou de novo, o placar
junta um resultado novo com um antigo (por exemplo, um `git-B.json` de ontem
com um `sprout-B.json` de hoje). Por isso ele mostra a data de cada arquivo, e
não recusa datas diferentes: o apresentador decide.

### Verificação do estado final

Só no modo rápido, nos três cenários. Depois de todos os agentes terminarem,
fora do tempo total, o harness lê a `main` na fonte da verdade:

- no git, no `central.git`: `git ls-tree -r -z main` para os nomes,
  `git cat-file blob` para o conteúdo de cada arquivo e
  `git rev-list --count main` para o número de commits;
- no Sprout, no servidor: `GET /refs/main` para a cabeça e
  `GET /changesets/:id/tree` para os nomes e o `blobHash` de cada arquivo.

O esperado é, para cada agente aceito, o seu arquivo com o texto `agente N`:
no A, `agente-1.txt` e `agente-2.txt`; no B, `agente-1.txt` a
`agente-10.txt`; no C, `conflito.txt` com o texto do agente que venceu. No
git, soma-se o `README.md` do commit inicial. A conferência pede:

- nenhum arquivo faltando nem sobrando;
- o conteúdo igual ao esperado (no Sprout, o `blobHash` igual ao hash do
  conteúdo esperado);
- o número de commits igual a 1 + aceitos: o commit inicial (ou o changeset
  raiz) mais um por agente aceito;
- o número de aceitos certo para o cenário: A 2, B 10, C 1.

No C, o commit do perdedor fica fora da `main` de propósito: é o conflito
real. Por isso a conferência prova que todo commit aceito está na `main`, e
não que todos os agentes entraram.

Quando a `main` passa, a execução imprime a linha da conferência antes do
separador (§5), por exemplo
`[git] main conferida → ✓ 10 commits aceitos na main (arquivos, conteúdo e contagem)`.
O número é o de agentes aceitos, igual nos dois sistemas: o `README.md` do
git é conferido, mas não entra nele. Quando não passa, é erro do harness (§3,
§4): a linha não sai, o JSON não é gravado, a demo sai com código 1 e o
stderr diz o que falta, o que sobra, que arquivo tem conteúdo diferente e,
quando for o caso, a contagem esperada e a obtida. Por exemplo:

```
demo:git: main check: missing agente-2.txt
```

O passo a passo (§5) não tem essa conferência: ele confere etapas, não
arquivos.

**Premissa não verificada.** No Sprout, o número de commits é o id da cabeça:
o changeset raiz é o 1, e cada changeset aceito recebe o id seguinte. Isso
depende de uma regra do SQLite (numa `INTEGER PRIMARY KEY` sem
`AUTOINCREMENT`, o próximo id é o maior + 1), que o repositório não confere.
Se a regra falhar, a conferência falha com erro do harness; ela não passa em
silêncio.

## 7. Medição para o repositório

O placar da apresentação é de uma execução. Para o repositório,
`pnpm demo:medir` (#21) roda os cenários A, B e C nos dois sistemas, no modo
rápido, **5 vezes** cada (30 execuções), e grava
[`docs/resultados.md`](resultados.md). O arquivo commitado é o resultado
oficial.

### Como as execuções rodam

- **Um processo só.** O comando (`demo/measure-main.ts`, `demo/measure-cli.ts`
  e `demo/measure.ts`) chama as mesmas funções dos comandos da demo
  (`runGitScenario` e `runSproutScenario`), com as linhas dos agentes
  silenciosas, e usa o resultado que elas devolvem. O servidor do Sprout é um
  processo filho novo a cada execução, e o trabalho do git é feito por
  processos `git`; só o código do lado do cliente (os agentes) fica aquecido
  depois da primeira execução.
- **Ordem intercalada.** As execuções são sequenciais, nunca duas ao mesmo
  tempo. Em cada uma das 5 rodadas: A git, A sprout, B git, B sprout, C git,
  C sprout. Assim, os efeitos da máquina (carga, temperatura) se espalham pelos
  dois sistemas.
- **Sem aquecimento.** Nenhuma execução é descartada; as 30 aparecem no
  documento.
- **Pastas próprias.** Os repositórios do git, os dados do servidor e os JSON
  ficam em `demo/tmp/medicao/`. Nada fora dessa pasta muda em `demo/tmp`, e os
  resultados da última demo continuam lá. Os JSON da medição não são lidos (só
  sobra o da última rodada).

### O que o `docs/resultados.md` traz

- **Tempo total** de cada execução, o mesmo do placar (§6): os 5 valores, a
  média, a mediana e a variação, que é o **mínimo–máximo**. Não há desvio
  padrão: 5 amostras não o sustentam. A média e a mediana são calculadas sobre
  os valores brutos e arredondadas para ms inteiros só no fim; os 5 valores e o
  mínimo–máximo aparecem em ms inteiros.
- **Contagens**, as mesmas métricas do placar: aceitos, rejeições, tentativas,
  trocas de pai e conflitos reais. Quando as 5 execuções concordam, um valor;
  senão, o mínimo–máximo e os 5 valores. Trocas de pai no git: `-`, como no
  placar.
- **Máquina:** modelo do processador e número de CPUs, memória total, sistema
  (o `PRETTY_NAME` do `/etc/os-release` e o kernel; sem esse arquivo,
  `os.type()` e `os.release()`) e a data e a hora do fim da medição. Sem
  hostname.
- **Versões:** Node, git e SQLite. O SQLite aparece como
  `SQLite 3.53.4 (better-sqlite3 13.0.3)`: a versão real, lida com
  `select sqlite_version()`, e a do pacote, que é outro número.
- **Versão do Sprout:** o hash curto do commit medido
  (`git rev-parse --short HEAD`), não uma tag nem a versão do `package.json`:
  a tag da versão ainda não existe na hora da medição. Se a árvore tiver
  mudanças não commitadas, fora o próprio `docs/resultados.md`, um aviso sai no
  arquivo, junto do hash, e no stderr.
- **Ressalvas**, entre elas a de que o repositório central do git roda na
  mesma máquina, sem rede, o que favorece o git no tempo.

O Markdown passa pela API do prettier, com a configuração do repositório, antes
de ser gravado; por isso o arquivo gerado passa no `pnpm format:check` sem
edição.

### Progresso e falha

O stdout mostra uma linha por execução, sem cor, e no fim onde o arquivo foi
gravado. A medição que gerou o `docs/resultados.md` commitado (18/11/2026)
começou assim:

```
[1/30] git A: 137 ms
[2/30] sprout A: 30 ms
[3/30] git B: 1080 ms
[4/30] sprout B: 88 ms
[5/30] git C: 112 ms
[6/30] sprout C: 20 ms
```

e, depois das rodadas 2 a 5, terminou assim:

```
[30/30] sprout C: 20 ms
resultados gravados em docs/resultados.md
```

O arquivo só é gravado depois das 30 execuções e da formatação. O primeiro erro
do harness (§3, §4, §6) interrompe a medição, e o `docs/resultados.md` não é
tocado: nunca há resultado parcial. Os códigos de saída estão no §8.

### Quem gera o número oficial

O `docs/resultados.md` commitado é gerado no terminal da apresentação (zsh do
omarchy, sem `mise exec`, com `node --version` conferido ali), na branch da
issue, depois do último commit de código e no mesmo PR. Com o squash merge, o
hash gravado deixa de existir na `main`; ele continua acessível pela página do
PR.

Validada com git 2.56.0 e node 26.12.0 no omarchy (Fase 1 da #14).

### A curva N × tentativas

`pnpm demo:curva` (#50) roda o cenário B com N = 2, 5, 10 e 20 agentes, cada
um no seu próprio arquivo, nos dois sistemas, no modo rápido, **3 vezes** cada
(24 execuções), e grava [`docs/curva.md`](curva.md). Para cada N, o arquivo
traz o teto do git, as tentativas nos dois sistemas e o tempo total (média e
mínimo–máximo).

- **Como a medição.** Um processo só, execuções sequenciais, sem aquecimento,
  Markdown pela API do prettier e aviso de árvore suja, fora o próprio
  `docs/curva.md`, como no `demo:medir`.
- **Ordem intercalada.** Em cada uma das 3 rodadas, N crescente e, para cada
  N, git e depois Sprout: git 2, sprout 2, git 5, sprout 5, git 10, sprout 10,
  git 20, sprout 20.
- **Pastas próprias.** Os repositórios do git, os dados do servidor e os JSON
  ficam em `demo/tmp/curva/`. Nada fora dessa pasta muda em `demo/tmp`: nem a
  última demo, nem a `demo/tmp/medicao/`.
- **O teto é o pior caso.** O teto N(N+1)/2 (§3) é o pior caso legítimo do git,
  não o valor esperado: uma execução pode ficar abaixo dele.
- **Tentativas estruturais no Sprout.** Cada agente faz um único pedido, sem
  nova tentativa; o valor é sempre N.

A curva que gerou o `docs/curva.md` commitado (18/11/2026) começou assim:

```
[1/24] git N=2: 137 ms
[2/24] sprout N=2: 29 ms
[3/24] git N=5: 432 ms
[4/24] sprout N=5: 49 ms
[5/24] git N=10: 1014 ms
[6/24] sprout N=10: 107 ms
[7/24] git N=20: 3145 ms
[8/24] sprout N=20: 182 ms
```

e, depois das rodadas 2 e 3, terminou assim:

```
[24/24] sprout N=20: 180 ms
curva gravada em docs/curva.md
```

O arquivo só é gravado depois das 24 execuções; o primeiro erro do harness
interrompe a curva, e o `docs/curva.md` não é tocado. Os códigos de saída estão
no §8.

O `docs/curva.md` commitado é gerado como o `docs/resultados.md`: no terminal
da apresentação (zsh do omarchy), na branch da issue, depois do último commit
de código e no mesmo PR. A curva commitada rodou com node 26.13.0.

## 8. Comandos

Definidos:

```
pnpm demo:reset                         apaga demo/tmp; cada execução da demo recria o que precisa
pnpm demo:git --cenario A|B|C           um cenário no git, no modo rápido
pnpm demo:sprout --cenario A|B|C        um cenário no Sprout, no modo rápido
pnpm demo:git --cenario A|C --passo     um cenário no git, passo a passo (§5)
pnpm demo:sprout --cenario A|C --passo  um cenário no Sprout, passo a passo (§5)
pnpm demo:placar --cenario A|B|C        o placar do cenário, git e Sprout lado a lado
```

No modo rápido, o `demo:git` grava `demo/tmp/resultados/git-<cenário>.json`, e
o `demo:sprout`, `demo/tmp/resultados/sprout-<cenário>.json`. Com `--passo`,
os dois apagam o JSON do cenário e não gravam outro (§6); um `--passo` do C
apaga o JSON de uma execução rápida do C. Os dois saem com:

- **0:** a execução terminou; no modo rápido, o JSON foi gravado; no passo a
  passo, não há JSON;
- **1:** erro do harness (§3 no git, §4 no Sprout, §5 na conferência do passo
  a passo, §6 na conferência da `main` do modo rápido): mensagem no stderr,
  sem JSON. Com `--passo`, também o stdin fechado durante uma pausa
  (`stdin closed during a pause (Enter expected)`);
- **2:** cenário ausente ou inválido, ou `--passo` com o cenário B
  (`--passo só existe nos cenários A e C`): mensagem no stderr.

O Ctrl+C numa pausa do passo a passo mata a demo pelo SIGINT (§4). O código
que o shell mostra é o do sinal, não um código da demo: 130 no `$?`, ou seja,
128 + 2, em que 2 é o número do SIGINT.

O `demo:placar` lê esses dois arquivos e não grava nada (§6). Sai com:

- **0:** o placar foi impresso no stdout;
- **1:** um dos arquivos está ausente, ilegível ou inválido: uma linha no
  stderr para cada arquivo com problema, com o caminho e o comando que o gera
  (por exemplo, `rode pnpm demo:git --cenario B`), e nada no stdout;
- **2:** cenário ausente ou inválido: mensagem no stderr.

A medição do §7:

```
pnpm demo:medir
```

Não aceita argumentos, roda as 30 execuções e grava `docs/resultados.md`. Sai
com:

- **0:** as 30 execuções terminaram e o `docs/resultados.md` foi gravado;
- **1:** erro do harness numa execução (§3 no git, §4 no Sprout, §6 na
  conferência da `main`): a medição para ali, sem rodar as execuções seguintes,
  o stderr traz a mensagem original com a rodada, o sistema e o cenário, no
  formato `demo:medir: run 3/5, git B: <mensagem original>`, e o
  `docs/resultados.md` não é tocado. Também sai com 1, sem gravar, um erro do
  harness ao ler a máquina, as versões ou o estado da árvore depois das 30
  execuções (`demo:medir: <mensagem>`);
- **2:** qualquer argumento: uma linha de uso no stderr, nenhuma execução e
  nenhuma gravação.

Como no `demo:git` e no `demo:sprout`, um erro que não é do harness é bug: é
relançado, sem código de saída próprio, e o `docs/resultados.md` não é
gravado.

A curva do §7:

```
pnpm demo:curva
```

Não aceita argumentos, roda as 24 execuções e grava `docs/curva.md`. Sai com:

- **0:** as 24 execuções terminaram e o `docs/curva.md` foi gravado. Com
  mudanças não commitadas, fora o próprio `docs/curva.md`, um aviso sai no
  stderr e no arquivo, e o código continua 0;
- **1:** erro do harness numa execução (§3 no git, §4 no Sprout, §6 na
  conferência da `main`): a curva para ali, sem rodar as execuções seguintes,
  o stderr traz a mensagem original com a rodada, o sistema e o N, no formato
  `demo:curva: run 2/3, git N=20: <mensagem original>`, e o `docs/curva.md`
  não é tocado. Também sai com 1, sem gravar, um erro do harness ao ler a
  máquina, as versões ou o estado da árvore depois das 24 execuções
  (`demo:curva: <mensagem>`);
- **2:** qualquer argumento: a linha `usage: pnpm demo:curva (no arguments)`
  no stderr, nenhuma execução e nenhuma gravação.

Como no `demo:medir`, um erro que não é do harness é bug: é relançado, sem
código de saída próprio, e o `docs/curva.md` não é gravado.

**Cores.** Os três comandos (`demo:git`, `demo:sprout` e `demo:placar`) só
usam cores quando o stdout é um terminal e a variável `NO_COLOR` está ausente
ou vazia, como pede o [no-color.org](https://no-color.org). Com a saída
redirecionada para um arquivo ou um pipe, ou com `NO_COLOR` com qualquer
valor não vazio (até `0`), o texto sai sem códigos de cor. `NO_COLOR=` (vazio) não desliga as
cores:

```
NO_COLOR=1 pnpm demo:placar --cenario B
NO_COLOR= pnpm demo:placar --cenario B
pnpm demo:placar --cenario B | cat -v
```

- o primeiro sai sem cor;
- o segundo sai com cor;
- o terceiro sai sem cor: nenhum `^[` (o ESC) na saída.

Verificado à mão em 17/11/2026, num terminal com `tput cols` = 142:

- os sete comandos da demo (passo a passo do A e do C nos dois sistemas,
  modo rápido do B nos dois e o placar do B) com os textos dos exemplos do
  §5 e do §6, código de saída 0 nos três do cenário B, cores visíveis, o
  cinza legível e a linha de 111 colunas do Sprout C inteira;
- `NO_COLOR=1` sem cor e `NO_COLOR=` (vazio) com cor;
- com `command cat -v`, nenhum `^[` na saída do `demo:placar --cenario B`, na
  do `demo:git --cenario A --passo` nem na do `demo:sprout --cenario B`. O
  `command` é necessário quando `cat` é um alias de outro programa (como o
  `bat`, que não aceita `-v`);
- uma gravação com o VHS (114 colunas): as cores aparecem, a linha de 111
  cabe e o placar mostra o destaque do §6.
