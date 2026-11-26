# Sprout

O Sprout é um mini clone do Scion, o sistema de controle de versão interno da Anthropic. É uma demonstração, não um produto: compara o git com um servidor simples que segue uma ideia do Scion (aceitar commits concorrentes que não mexem nos mesmos arquivos, sem rejeitar o trabalho) quando vários agentes fazem commit ao mesmo tempo.

## Relação com o trabalho

O Sprout foi feito para a demonstração do Trabalho 2. Os números vêm de uma única máquina, com repositórios locais; servem para ilustrar a ideia, não para medir o Scion. A análise do problema do git está em [../conteudo/01-problema-central.md](../conteudo/01-problema-central.md) e nos slides ([../slides/apresentacao-final.pdf](../slides/apresentacao-final.pdf)).

## Vídeo

<!-- link do vídeo da demonstração entra aqui -->

## Código

Repositório: https://github.com/ArturCousseau/sprout (tag `v1.0.0`).
O repositório pode ficar privado depois do semestre; por isso a documentação
está copiada abaixo.

## Documentação

Cópia de docs/ do Sprout na tag v1.0.0 (commit 391109a).
Os documentos citam ADRs (docs/adr/) e arquivos de código (src/, demo/) que não foram copiados; eles estão no repositório do Sprout, na mesma tag.

- [`docs/escopo.md`](docs/escopo.md): escopo e decisões do projeto
- [`docs/demo.md`](docs/demo.md): como a demonstração funciona
- [`docs/resultados.md`](docs/resultados.md): resultados medidos
- [`docs/curva.md`](docs/curva.md): curva por número de agentes
