Entendendo a proposta de distribuição do Git:

O Git adota uma organização em que cada clone é um repositório funcional.

Esse clone contém, por exemplo:

Commits
Histórico
Branches e referências
Versionamento

Isso fornece autonomia ao desenvolvedor no gerenciamento desse repositório local, ao remover a necessidade de conexão com o servidor remoto.

Muitas empresas utilizam um repositório como cópia oficial, o qual os colaboradores clonam, modificam e atualizam remotamente. Isso pode fazer parecer que o Git é uma tecnologia centralizada, mas, na verdade, esse cenário representa apenas uma organização centralizada que foi construída sobre uma tecnologia distribuída.

Como o caso da Anthropic se encaixa nisso:

A Anthropic organizava seu desenvolvimento em um monorepo, um único repositório que contém vários projetos, serviços ou bibliotecas. Esse monorepo contava com:

* 30.000 alterações de código integradas por dia no monorepo.
* 100 GB de histórico da branch principal.
* 25 milhões de checkouts por dia para CI.

Aproximadamente 600 PB de tráfego Git por dia (sem otimizações).

A solução imediata para reduzir esse gargalo era organizar um sistema de distribuição de carga.

Quais soluções o Git proporcionava para isso?

* Git mirrors

É uma cópia sincronizada de um repositório Git. Torna-se uma via alternativa para a clonagem do repositório principal.

Utilização: em vez de clonar o repositório por meio do servidor Git principal, o clone é feito pelo mirror, que apenas se sincroniza com o servidor principal.

* Seed

É uma cópia compactada e dessincronizada do repositório (snapshot). Torna-se um meio de acesso ao repositório alternativo à clonagem.

Utilização: Baixar do serviço de hospedagem. Descompactar. Atualizar o repositório pelo servidor Git apenas com as mudanças.

* Clones parciais

É uma maneira alternativa de clonagem, na qual é possível clonar apenas determinadas partes do repositório, o que permite, por exemplo, excluir inicialmente o conteúdo dos arquivos, selecionar apenas determinada branch etc.

Utilização: O comando git clone é executado recebendo os parâmetros que selecionam determinada parte do repositório, por exemplo:

git clone --filter=blob:none URL — Não clona o conteúdo de nenhum arquivo.

Durante uma operação do Git que precise do conteúdo do arquivo, por exemplo, git show main:main.c, o Git executa um demand fetch, que é o processo de trazer o objeto do servidor para o repositório local.

Por que essas soluções não resolveram o gargalo da Anthropic?

* Centenas de servidores mirrors sendo mantidos.
* Repositórios ultrapassando os limites de disco e memória dos mirrors, consequentemente perdendo a escalabilidade.
* Seeds cada vez mais desatualizados por conta de distâncias cada vez maiores entre a ponta da branch principal do seed e a do repositório.
* Era necessário cada vez mais cuidado para manter as operações dependentes do histórico funcionando, pois, como foi visto, o clone parcial não elimina objetos, apenas posterga sua transferência até que uma operação precise deles.

Referências:

ANTHROPIC. Redesigning source control at Anthropic. Anthropic Engineering, 2026. Disponível em: https://www.anthropic.com/research/redesigning-source-control. Acesso em: 22 nov. 2026.

GIT. A short history of Git. Pro Git. Disponível em: https://git-scm.com/book/en/v2/Getting-Started-A-Short-History-of-Git. Acesso em: 22 nov. 2026.

GIT. Distributed workflows. Pro Git. Disponível em: https://git-scm.com/book/en/v2/Distributed-Git-Distributed-Workflows.html. Acesso em: 22 nov. 2026.

GIT. Partial clone. Git Documentation, 2026. Disponível em: https://git-scm.com/docs/partial-clone.html. Acesso em: 22 nov. 2026.

GITHUB. Introducing the GHES repository cache. GitHub Blog, 22 mar. 2022. Disponível em: https://github.blog/news-insights/product-news/introducing-the-ghes-repository-cache/. Acesso em: 22 nov. 2026.