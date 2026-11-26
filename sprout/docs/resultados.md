# Resultados da medição

<!-- Gerado por `pnpm demo:medir`; não edite à mão. Para refazer, rode `pnpm demo:medir`. -->

Medido em 18/11/2026 16:14.

Commit medido: `5bce0e3`.

## Máquina e versões

- Processador: AMD Ryzen 7 5700U with Radeon Graphics (16 CPUs)
- Memória: 18,9 GiB
- Sistema: Omarchy, kernel 7.2.15-1-omarchy
- Node: v26.13.0
- Git: git version 2.56.0
- SQLite 3.53.4 (better-sqlite3 13.0.3)

## Tempo total

Cada cenário rodou 5 vezes em cada sistema, no modo rápido, em rodadas intercaladas. O tempo de cada execução, em ms, vai da liberação da barreira até o último agente terminar.

Média e mediana são calculadas sobre os valores brutos e arredondadas para ms inteiros só no fim; os valores de cada execução e o mínimo–máximo aparecem arredondados um a um.

| Cenário | Sistema | Execuções (ms)               | Média | Mediana | Mínimo–máximo |
| ------- | ------- | ---------------------------- | ----: | ------: | ------------- |
| A       | git     | 137, 140, 138, 139, 141      |   139 |     139 | 137–141       |
| A       | sprout  | 30, 25, 25, 21, 26           |    26 |      25 | 21–30         |
| B       | git     | 1080, 1040, 1017, 1069, 1167 |  1075 |    1069 | 1017–1167     |
| B       | sprout  | 88, 88, 90, 94, 125          |    97 |      90 | 88–125        |
| C       | git     | 112, 110, 109, 115, 122      |   114 |     112 | 109–122       |
| C       | sprout  | 20, 19, 19, 21, 20           |    20 |      20 | 19–21         |

## Contagens

Um valor quando as 5 execuções concordam; senão, o mínimo–máximo e os 5 valores, na ordem das rodadas.

| Cenário | Sistema | aceitos | rejeições | tentativas | trocas de pai | conflitos reais |
| ------- | ------- | ------: | --------: | ---------: | ------------: | --------------: |
| A       | git     |       2 |         1 |          3 |             - |               0 |
| A       | sprout  |       2 |         0 |          2 |             1 |               0 |
| B       | git     |      10 |        45 |         55 |             - |               0 |
| B       | sprout  |      10 |         0 |         10 |             9 |               0 |
| C       | git     |       1 |         1 |          2 |             - |               1 |
| C       | sprout  |       1 |         0 |          2 |             0 |               1 |

## Ressalvas

- O repositório central do git roda na mesma máquina, sem rede, o que favorece o git no tempo.
- Uma máquina só; os números não valem para outra máquina.
- As 30 execuções rodam num processo só. O servidor do Sprout é um processo filho novo a cada execução e o trabalho do git é feito por processos `git`; só o código do lado do cliente (os agentes) fica aquecido depois da primeira execução. Nenhuma execução foi descartada.
- O tempo total vai da liberação da barreira até o último agente terminar. No git, inclui o lançamento dos processos `git push`, que se espalha por cerca de 30 ms (`docs/demo.md` §3); no Sprout, o pedido HTTP local até o corpo da resposta lido.
- A preparação, o título e a conferência da `main` ficam fora do tempo medido.
- Com 5 amostras, a variação é dada pelo mínimo–máximo, sem desvio padrão; média e mediana são arredondadas só no fim.
- Os agentes são rotinas com roteiro fixo, não modelos de IA.
