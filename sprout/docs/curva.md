# Curva N × tentativas

<!-- Gerado por `pnpm demo:curva`; não edite à mão. Para refazer, rode `pnpm demo:curva`. -->

Medido em 18/11/2026 20:22.

Commit medido: `7799086`.

## Máquina e versões

- Processador: AMD Ryzen 7 5700U with Radeon Graphics (16 CPUs)
- Memória: 18,9 GiB
- Sistema: Omarchy, kernel 7.2.15-1-omarchy
- Node: v26.13.0
- Git: git version 2.56.0
- SQLite 3.53.4 (better-sqlite3 13.0.3)

## Curva

Cada N rodou 3 vezes em cada sistema, no modo rápido, com cada agente no seu próprio arquivo, em rodadas intercaladas. O tempo de cada execução, em ms, vai da liberação da barreira até o último agente terminar.

Tentativas: um valor quando as 3 execuções concordam; senão, o mínimo–máximo e os 3 valores, na ordem das rodadas.

|   N | Teto do git | Tentativas no git       | Tentativas no Sprout | Tempo do git (média) | Tempo do git (mínimo–máximo) | Tempo do Sprout (média) | Tempo do Sprout (mínimo–máximo) |
| --: | ----------: | ----------------------- | -------------------: | -------------------: | ---------------------------- | ----------------------: | ------------------------------- |
|   2 |           3 | 3                       |                    2 |                  138 | 135–143                      |                      25 | 20–29                           |
|   5 |          15 | 15                      |                    5 |                  433 | 426–442                      |                      49 | 47–51                           |
|  10 |          55 | 55                      |                   10 |                 1018 | 1000–1040                    |                      99 | 92–107                          |
|  20 |         210 | 175–189 (189, 179, 175) |                   20 |                 2822 | 2639–3145                    |                     180 | 177–182                         |

## Ressalvas

- O repositório central do git roda na mesma máquina, sem rede, o que favorece o git no tempo.
- Uma máquina só; os números não valem para outra máquina.
- Todas as execuções rodam num processo só. O servidor do Sprout é um processo filho novo a cada execução e o trabalho do git é feito por processos `git`; só o código do lado do cliente (os agentes) fica aquecido depois da primeira execução. Nenhuma execução foi descartada.
- O teto N(N+1)/2 é o pior caso legítimo do git, não o valor esperado: uma execução pode ficar abaixo dele (`docs/demo.md` §3).
- No Sprout, as tentativas são estruturais: cada agente faz um único pedido, sem nova tentativa, então o valor é sempre N.
- O tempo total vai da liberação da barreira até o último agente terminar. No git, inclui o lançamento dos processos `git push`, um depois do outro, cerca de 3,7 ms cada (`docs/demo.md` §3); no Sprout, o pedido HTTP local até o corpo da resposta lido.
- A preparação, o título e a conferência da `main` ficam fora do tempo medido.
- Com 3 amostras por N e sistema, a variação é dada pelo mínimo–máximo, sem desvio padrão; a média é arredondada só no fim.
- Cada execução é o cenário B com N agentes: por dentro, o título diz "cenário B" e os JSON se chamam `git-B.json` e `sprout-B.json`, em `demo/tmp/curva/resultados/`. Nada disso aparece na saída nem é lido.
- Os agentes são rotinas com roteiro fixo, não modelos de IA.
