
# Lunatik States Management Protocol

## Introdução

Este documento tem por objetivo definir e documentar o funcionamento do protocolo de gerenciamento de estados do Lunatik (Lunatik States Management Protocol). Este protocolo por sua vez tem o propósito de oferecer uma interface para as operações relacionadas a estados no ambiente do Lunatik, tanto para o Kernel por meio de uma **API** quanto para o espaço de usuário por meio de um **binding** Lua. 

## Estrutura básica

A figura a seguir apresenta a estrutura geral do gerenciamento de estados do Lunatik:

![enter image description here](https://i.ibb.co/dmtTbVQ/estrutura-geral-1.png)

Figura 1. Estrutura geral do funcionamento do gerenciamento de estados no Lunatik

Como mostrado na figura 1, o gerenciamento de estados do Lunatik é oferecido tanto no espaço de usuário, com um binding para Lua, quanto para o Kernel, por meio de API que pode ser consumida pelos outros módulos do Kernel. Como indicado nas orientações das setas, tanto o binding quanto a API podem enviar e receber informações do módulo do Lunatik. Tais informações podem ser circunstanciais, isto é: trazem informações relacionadas os estados presentes no Lunatik. Como  podem ser de controle, requisitando que uma determinada operação seja realizada sobre a lista de estados do Lunatik ou sobre um determinado estado em específico.

Com o binding Lua o usuário pode realizar operações no kernel mesmo estando no espaço de usuário, isto é feito graças ao Generic Netlink, que permite, além de outras coisas, a comunicação entre processos do espaço de usuário e Kernel.

Já a API de gerenciamento de estados provê uma interface para o controle dos estados presentes no Lunatik. Isso facilita a vida do desenvolvedor dos módulos do Kernel que fazem o uso do Lunatik, uma vez que estes desenvolvedores não precisam se preocupar com o gerenciamento dos estados Lua que estão utilizando.

## Ambientes Lunatik
O gerenciamento de estados do Lunatik é feito em seus respectivos ambientes, cada ambiente possui seu próprio conjunto de estados fazendo com que os ambientes sejam isolados entre si, a figura 2 dá uma noção em alto nível como é feito esse isolamento. Para maiores informações sobre ambientes Lunatik consulte o glossário [aqui](#ambientes_lunatik).

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/pbDs7vJ/linux.png)
Fig 2. Estrutura geral dos ambientes Lunatik

Como cada ambiente possui seus próprios conjuntos de estados, isso implica que esses conjuntos são isolados e não se comunicam entre si. Isso faz com que a existência de dois estados Lunatik com o mesmo nome seja possível, desde que estes estados estejam associados a ambientes diferentes.

Vale ressaltar também que os ambientes estão sempre associados aos namespaces do Linux, fazendo com que o acesso a eles sejam feitos por meio de tais namespaces. Assim, caso o usuário deseje realizar qualquer operação em um determinado ambiente, ao invés de informar o ambiente em si, ele deve informar o namespace que o ambiente em questão está associado. [DISCUTIR ISSO]


## Operações oferecidas pela API

Assim como descrito na seção acima, o Lunatik possui o conceito de ambientes, provendo assim um isolamento de recursos. Além disso, as operações oferecidas pelo gerenciamento de estados podem estar associadas a um determinado ambiente ou não. De fato, mesmo que o usuário não cite explicitamente um determinado ambiente, o ambiente padrão será escolhido, fazendo com que todas as operações estejam associadas a um ambiente, mesmo que implicitamente. Isso faz com que todas as operações no gerenciamento de estados possuam duas variantes, uma envolvendo um ambiente explícito, indicado pelo usuário através de um `net namespace` e outra que de forma abstrata não envolve ambientes, e consequentemente o ambiente não precisa ser informado pelo usuário.

- **Criação de estados:** Cria um estado Lunatik no kernel a partir de um nome e um alocamento máximo de memória no Kernel (Veja [aqui](#estado_lunatik) maiores informaçõe sobre estados Lunatik). O nome servirá como identificador único do estado e o alocamento máximo de memória representa a quantidade máxima de memória que o estado poderá utilizar. Retorna um ponteiro para o estado lunatik em caso de sucesso e NULL caso contrário.
  - Assinatura da função sem ambiente associado: `lunatik_State *lunatik_newstate(const char *name, size_t maxalloc)`
  - Assinatura da função com ambiente associado: `lunatik_State *lunatik_netnewstate(const char *name, size_t maxalloc, struct net *env)`
- **Exclusão de estados:** Exclui, a partir de um nome, um estado Lunatik. Essa exclusão é feita removendo o estado da estrutura de armazenamento de estados do ambiente Lunatik e só pode ser realizada caso só exista uma única refêrencia ao estado. Caso contrário, mesmo que a operação seja solicitada pelo usuário, nenhuma alteração na estrutura de armazenamento de estados do Lunatik ocorrerá. Retorna 0 caso o estado tenha sido excluído e um número negativo caso contrário.
  - Assinatura da função sem ambiente associado: `int lunatik_close(const char *name)`
  - Assinatura da função com ambiente associado: `int lunatik_netclosestate(const char *name, struct net *env)`
 - **Busca de estados:** Realiza uma busca, em um determinado ambiente Lunatik, a partir de um nome. Se tal estado for encontrado então um ponteiro para ele é retornado, se o estado não existir então o valor NULL é retornado.
  - Assinatura da função sem ambiente associado: `lunatik_State *lunatik_statelookup(const char *name)`
  - Assinatura da função com ambiente associado: `lunatik_State *lunatik_netstatelookup(const char *name, struct net *env)`
- **Obtenção de referência para o estado:** Incrementa o contador de referência de um determinado estado. Isso é importante pois um determinado estado pode ter sido criado em algum ponto do tempo e então você pode decidir realizar uma ação sobre esse estado (digamos executar um código em Lua), caso você não obtenha a referência ao estado e decida realizar a sua ação, pode ser que enquanto sua ação está sendo realizada alguém solicite que o estado que você está manipulando seja destruído, e caso a contagem de refência para ele for igual a 1 este estado será apagado. fazendo com que o seu acesso a memória seja inválido. Por conta disso essa é uma operação importante e evita tais situações, sendo assim, toda vez que for necessário fazer alguma operação no estado é necessário o incremento da contagem de referência para este estado. Retorna verdadeiro caso o incremento de referência tenha sido incrementado com sucesso e falso caso contrário (situação de overflow). Vale ressaltar que esta é uma operação associada ao ponteiro do estado em si, portanto o ambiente não é necessário nessa operação, afinal de contas se um determinado estado existe este já está associado a um ambiente.
  - Assinatura da função: `bool lunatik_getstate(lunatik_State *state)`
 - **Devolução de referência:** Decrementa o contador de referência de um estado. Assim que uma determinada operação com o estado Lunatik for concluída, a referência obtida por meio da função `lunatik_getstate()` deve ser devolvida a fim de evitar inconsistências. Assim como a operação anterior, esta operação tem o ambiente implicitamente passado, pois ele está ligado ao estado passado à função, tornando desnecessário a passagem deste parâmetro para a função.
	 - Assinatura da função: `void lunatik_putstate(lunatik_State *state)`

## <a name="glossary"></a>Glossário

- <a name="estado_lunatik"></a>**Estado Lunatik:** Um estado Lunatik é uma estrutura de dados opaca que serve como uma capsula para os estados Lua e tem o propósito de abstrair todas as operações relacionadas ao gerenciamento de estados em Lua.
- <a name="ambientes_lunatik"></a> **Ambiente Lunatik:** Um ambiente Lunatik é uma instância de armazenamento de estados do Lunatik. Em outras palavras, a estrutura usada pelo Lunatik para armazenar seus estados estará sempre associada a um net namespace, mesmo que não especificada pelo usuário. Neste caso, estará associada ao net namespace do processo `init` do Linux. Isso faz com que o Lunatik possa prover aos seus usuários um isolamento de recursos.
