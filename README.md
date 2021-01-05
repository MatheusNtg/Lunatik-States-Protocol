
# Lunatik States Management Protocol

## Introdução

Este documento tem por objetivo definir e documentar o funcionamento do protocolo de gerenciamento de estados do Lunatik (Lunatik States Management Protocol). Este protocolo por sua vez tem o propósito de oferecer uma interface para as operações relacionadas a estados no ambiente do Lunatik, tanto para o Kernel por meio de uma **API** quanto para o espaço de usuário por meio de um **binding** Lua. 

## Estrutura básica

A figura a seguir apresenta a estrutura geral do gerenciamento de estados do Lunatik:

![enter image description here](https://i.ibb.co/dmtTbVQ/estrutura-geral-1.png)

Figura 1. Estrutura geral do funcionamento do gerenciamento de estados no Lunatik

Como mostrado na figura 1, o gerenciamento de estados do Lunatik é oferecido tanto no espaço de usuário, com um binding para Lua, quanto para o Kernel, por meio de API que pode ser consumida pelos outros módulos do Kernel. 

## Ambientes Lunatik
O gerenciamento de estados do Lunatik é feito em seus respectivos ambientes, cada ambiente possui seu próprio conjunto de estados fazendo com que os ambientes sejam isolados entre si, a figura 2 dá uma noção em alto nível como é feito esse isolamento.

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/pbDs7vJ/linux.png)

Fig 2. Estrutura geral dos ambientes Lunatik

## Operações oferecidas pela API

- **Criação de estados:** Cria um estado Lunatik no kernel a partir de um nome e um alocamento máximo de memória no Kernel (Veja [aqui](#estado_lunatik) maiores informaçõe sobre estados Lunatik). O nome servirá como identificador único do estado e o alocamento máximo de memória representa a quantidade máxima de memória que o estado poderá utilizar. Retorna um ponteiro para o estado lunatik em caso de sucesso e NULL caso contrário.
  - Assinatura da função: `lunatik_State *lunatik_newstate(const char *name, size_t maxalloc)`
- **Exclusão de estados:** Exclui, a partir de um nome, um estado Lunatik. Essa exclusão é feita removendo o estado da estrutura de armazenamento de estados do Lunatik e só pode ser realizada caso só exista uma única refêrencia ao estado. Caso contrário, mesmo que a operação seja solicitada pelo usuário, nenhuma alteração na estrutura de armazenamento de estados do Lunatik ocorrerá. Retorna 0 caso o estado tenha sido excluído e um número negativo caso contrário.
  - Assinatura da função: `int lunatik_close(const char *name)`
- **Busca de estados:** Realiza uma busca, a partir de um nome, na lista de estados do Lunatik. Se tal estado for encontrado então um ponteiro para ele é retornado, se o estado não existir então o valor NULL é retornado.
  - Assinatura da função: `lunatik_State *lunatik_statelookup(const char *name)`
- **Obtenção de referência para o estado:** Incrementa o contador de referência de um determinado estado. Isso é importante pois um determinado estado pode ter sido criado em algum ponto do tempo e então você pode decidir realizar uma ação sobre esse estado (digamos executar um código em Lua), caso você não obtenha a referência ao estado e decida realizar a sua ação, pode ser que enquanto sua ação está sendo realizada alguém solicite que o estado que você está manipulando seja destruído, e caso a contagem de refência para ele for igual a 1, este estado será apagado fazendo com que o seu acesso a memória seja inválido. Por conta disso essa é uma operação importante e evita tais situações, sendo assim, toda vez que for necessário fazer alguma operação no estado é necessário o incremento da contagem de referência para este estado. Retorna verdadeiro caso o incremento de referência pôde ser incrementado com sucesso e falso caso contrário (situação de overflow).
  - Assinatura da função: `bool lunatik_getstate(lunatik_State *state)`
 - **Devolução de referência:** Decrementa o contador de referência de um estado. Assim que uma determinada operação com o estado Lunatik for concluída, a referência obtida por meio da função `lunatik_getstate()` deve ser devolvida a fim de evitar inconsistências.
	 - Assinatura da função: `void lunatik_putstate(lunatik_State *state)`

## <a name="glossary"></a>Glossário

- <a name="estado_lunatik"></a>**Estado Lunatik:** Um estado Lunatik é uma estrutura de dados opaca que serve como uma capsula para os estados Lua e tem o propósito de abstrair todas as operações relacionadas ao gerenciamento de estados em Lua.
- <a name="ambientes_lunatik"></a> **Ambiente Lunatik:** Um ambiente Lunatik é uma instância de armazenamento de estados do Lunatik. Em outras palavras, a estrutura usada pelo Lunatik para armazenar seus estados estará sempre associada a um net namespace, mesmo que não especificada pelo usuário, neste caso, estará associada ao net namespace do processo `init` do Linux. Isso faz com que o Lunatik possa provêr aos seus usuários um isolamento de recursos.
