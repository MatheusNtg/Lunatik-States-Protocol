
# Lunatik States Management Protocol

## Introdução

Este documento tem por objetivo definir e documentar o funcionamento do protocolo de gerenciamento de estados do Lunatik (Lunatik States Management Protocol). Este protocolo por sua vez tem o propósito de oferecer uma interface para as operações relacionadas a estados no ambiente do Lunatik, tanto para o Kernel por meio de uma **API** quanto para o espaço de usuário por meio de um **binding** Lua. 

## Motivação
Para entendermos a motivação por trás do gerenciamento de estados Lunatik primeiro precisamos entender alguns conceitos, como: O que é o Lunatik e o que são estados Lua.

### Lunatik
Segundo ["Scriptable Operating Systems with Lua"](https://www.netbsd.org/~lneto/dls14.pdf) o Lunatik é:
> "A small subsystem that provides a programming and execution environment for OS kernel scripting based on the Lua programming language"

Podemos pensar no Lunatik como a implementação da linguagem Lua para ser executada no kernel, oferecendo um interpretador Lua que difere da implementação original apenas pelas limitações que o kernel impõe e que estão descritas [aqui](https://github.com/luainkernel/lunatik/tree/master/doc).

### Estados Lua
Para a realização de bindings, Lua utiliza o conceito de estados Lua, que nada mais são do que instâncias do ambiente de execução de um determinado programa Lua: [Programming in Lua : 24.1](http://www.lua.org/pil/24.1.html). A partir destes estados é possível acessar o contexto em que um determinado programa Lua está rodando e com isso realizar todas as alterações desejadas.  Assim como a implementação padrão de Lua, o Lunatik segue a mesma lógica de estados Lua para realizar as suas operações.

### Gerenciamento de estados

Imagine que você queira desenvolvedor um módulo para o kernel composto por vários componentes independentes e que cada componente precise acessar o mesmo estado Lua. Uma forma de fazer isso é encapsular o estado Lua em uma estrutura global relativa ao seu programa onde cada componente poderá ter acesso a este estado. Um problema claro desta abordagem é que o estado em questão pode facilmente sofrer de condição de corrida, forçando você a desenvolver um esquema de gerenciamento de concorrência para este estado. Este protocolo é desenvolvido com o intuito de sanar este tipo de problema, além de facilitar a comunicação entre o kernel e o espaço de usuário para o carregamento de código lua nos estados presentes no Lunatik.


## Estrutura básica

A figura a seguir apresenta a estrutura geral do gerenciamento de estados do Lunatik:

![enter image description here](https://svgshare.com/i/UEi.svg)

Figura 1. Estrutura geral do funcionamento do gerenciamento de estados no Lunatik

Como mostrado na figura 1, o gerenciamento de estados do Lunatik é oferecido tanto no espaço de usuário, com um binding para Lua, quanto para o Kernel, por meio de API que pode ser consumida pelos outros módulos do Kernel. Como indicado nas orientações das setas, tanto o binding quanto a API podem enviar e receber informações do módulo do Lunatik. Tais informações podem ser de dados como podem ser de controle, requisitando que uma determinada operação seja realizada sobre a lista de estados do Lunatik ou sobre um determinado estado em específico.

Com o binding Lua, o usuário pode realizar operações no kernel mesmo estando no espaço de usuário, isto é feito graças ao Generic Netlink, que permite, além de outras coisas, a comunicação entre processos do espaço de usuário e Kernel.

Já a API de gerenciamento de estados provê uma interface para o controle dos estados presentes no Lunatik.

## Ambientes Lunatik
O gerenciamento de estados do Lunatik é feito em seus respectivos ambientes, cada ambiente possui seu próprio conjunto de estados fazendo com que os ambientes sejam isolados entre si, a figura 2 dá uma noção em alto nível como é feito esse isolamento. Para maiores informações sobre ambientes Lunatik consulte o glossário [aqui](#ambientes_lunatik).

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/pbDs7vJ/linux.png)

Fig 2. Estrutura geral dos ambientes Lunatik

Como cada ambiente possui seus próprios conjuntos de estados, isso implica que esses conjuntos são isolados e não se comunicam entre si. Isso faz com que a existência de dois estados Lunatik com o mesmo nome seja possível, desde que estes estados estejam associados a ambientes diferentes.

Vale ressaltar também que os ambientes estão sempre associados aos namespaces do Linux, fazendo com que o acesso a eles sejam feitos por meio de tais namespaces. Assim, caso o usuário deseje realizar qualquer operação em um determinado ambiente, ao invés de informar o ambiente em si, ele deve informar o namespace que o ambiente em questão está associado. [DISCUTIR ISSO]


## Operações oferecidas pela API

Assim como descrito na seção acima, o Lunatik possui o conceito de ambientes, provendo assim um isolamento de recursos. Além disso, as operações oferecidas pelo gerenciamento de estados podem estar associadas a um determinado ambiente ou não. De fato, mesmo que o usuário não cite explicitamente um determinado ambiente, o ambiente padrão será escolhido, fazendo com que todas as operações estejam associadas a um ambiente, mesmo que implicitamente. Isso faz com que todas as operações no gerenciamento de estados possuam duas variantes, uma envolvendo um ambiente explícito, indicado pelo usuário através de um `net namespace` e outra que de forma abstrata não envolve ambientes, e consequentemente o ambiente não precisa ser informado pelo usuário. Para maiores informações relacionadas à assinaturas de funções consultar [API](#api)

- **Criação de estados:** Cria um estado Lunatik no kernel a partir de um nome e um alocamento máximo de memória no Kernel (Veja [aqui](#estado_lunatik) maiores informaçõe sobre estados Lunatik). O nome servirá como identificador único do estado e o alocamento máximo de memória representa a quantidade máxima de memória que o estado poderá utilizar. Retorna um ponteiro para o estado lunatik em caso de sucesso e NULL caso contrário.
- **Exclusão de estados:** Exclui, a partir de um nome, um estado Lunatik. Essa exclusão é feita removendo o estado da estrutura de armazenamento de estados do ambiente Lunatik e só pode ser realizada caso só exista uma única refêrencia ao estado. Caso contrário, mesmo que a operação seja solicitada pelo usuário, nenhuma alteração na estrutura de armazenamento de estados do Lunatik ocorrerá. Retorna 0 caso o estado tenha sido excluído e um número negativo caso contrário.
- **Busca de estados:** Realiza uma busca, em um determinado ambiente Lunatik, a partir de um nome. Se tal estado for encontrado então um ponteiro para ele é retornado, se o estado não existir então o valor NULL é retornado.
- **Obtenção de referência para o estado:** Incrementa o contador de referência de um determinado estado. Isso é importante pois um determinado estado pode ter sido criado em algum ponto do tempo e então você pode decidir realizar uma ação sobre esse estado (digamos executar um código em Lua), caso você não obtenha a referência ao estado e decida realizar a sua ação, pode ser que enquanto sua ação está sendo realizada alguém solicite que o estado que você está manipulando seja destruído, e caso a contagem de refência para ele for igual a 1 este estado será apagado. Fazendo com que o seu acesso a memória seja inválido. Por conta disso essa é uma operação importante e evita tais situações, sendo assim, toda vez que for necessário fazer alguma operação no estado é necessário o incremento da contagem de referência para este estado. Retorna verdadeiro caso o incremento de referência tenha sido incrementado com sucesso e falso caso contrário (situação de overflow). Vale ressaltar que esta é uma operação associada ao ponteiro do estado em si, portanto o ambiente não é necessário nessa operação, afinal de contas se um determinado estado existe este já está associado a um ambiente.
 - **Devolução de referência:** Decrementa o contador de referência de um estado. Assim que uma determinada operação com o estado Lunatik for concluída, a referência obtida por meio da função `lunatik_getstate()` deve ser devolvida a fim de evitar inconsistências. Assim como a operação anterior, esta operação tem o ambiente implicitamente passado, pois ele está ligado ao estado passado à função, tornando desnecessário a passagem deste parâmetro para a função.

## Protocolo de comunicação user space-kernel

Como mostrado na figura 1, um binding Lua é oferecido no espaço de usuário para realizar operações de gerenciamento de estados no kernel. Para que isso seja possível um processo de comunicação entre processos deve ser adotado, e no nosso caso o Generic Netlink foi o mecanismo escolhido, juntamente com a biblioteca [libnl](https://www.infradead.org/~tgr/libnl/) no espaço de usuário, que já implementa várias funcionalidades comumente usadas.  A estrutura geral da comunicação entre o espaço de usuário e o kernel é mostrada na figura a seguir:

![enter image description here](https://i.ibb.co/pWZ4qWG/estrutura-geral-protocolo.png)

Figura 3. Estrutura geral da comunicação entre o binding Lua e o Lunatik
### Pacotes Lunatik
Uma comunicação através de pacotes de controle e de dados é realizada a fim de obter as funcionalidades providas pelo gerenciador de estados no espaço de usuário, a estrutura lógica destes pacotes é a seguinte: Um pacote lunatik é uma classe abstrata de pacotes, sendo as suas respectivas instâncias os pacotes de controle e os de dados, diferindo um do outro somente pelo campo tipo presente na estrutura de pacotes. Fisicamente (i.e implementado), a estrutura de um pacote lunatik é a seguinte:

```c
struct lunatik_packet {
	char state_name[LUNATIK_NAME_MAXSIZE];
	enum lunatik_packet_type type;
	enum lunatik_operations operation;
	size_t len;
	void *associated_value;
};
```
Cujos os campos têm os significados:
- `state_name`: O nome do estado cujo a operação será realizada;
- `type`: O tipo do pacote lunatik;
- `operation`: A operação a ser realizada no gerenciador de estados;
- `len`: O tamanho do pacote;
- `associated_value`: Ponteiro para valores associados a determinadas operações que precisam de dados extras aos fornecidos na estrutura.
## Tipos de pacotes:
Como dito na seção anterior, só existem dois tipos de pacotes, sendo eles os de controle e os de dados, dessa forma, o campo `type` da estrutura `struct lunatik_packet` tem o seguinte formato:
```c
enum lunatik_packet_type {
	LUNATIK_CONTROL,
	LUNATIK_DATA
};
``` 
## Operações
Nesta seção será discutido os detalhes de implementação dos pacotes das operações realizadas no espaço de usuário. Para isso precisamos entender quais são os tipos de operações disponíveis no gerenciamento de estado, que está definido na enumeração `enum lunatik_operations`   e que possui seu representante na `struct lunatik_packet` chamado de `operation`.
```c
enum lunatik_operations {
	LUNATIK_NEWSTATE = 1,
	LUNATIK_CLOSE,
	LUNATIK_DOSTRING,
	LUNATIK_GETSTATE,
	LUNATIK_PUTSTATE,
	LUNATIK_DATA
};
```
Uma vez definido os tipos de operações presentes nos pacotes podemos especificar os detalhes de cada uma delas.
### `LUNATIK_NEWSTATE`
Operação responsável pela criação de estados, e assim como definido na seção de operações oferecidas pelo gerenciamento de estados, essa operação necessita do nome do estado que servirá como ID para o estado e de um alocamento máximo de memória que será usado para determinar qual será o valor máximo de memória que este estado poderá utilizar no kernel.  Para realizar tal funcionalidade, o campo `associated_value` desta operação aponta para o alocamento máximo utilizado pelo estado.
### `LUNATIK_CLOSE`
Operação responsável pela deleção de estados existentes, essa operação não necessita de nenhum valor associado além dos presentes na estrutura `struct lunatik_packet`. 
### `LUNATIK_DOSTRING`
Operação utilizada para execução de códigos em Lua nos estados Lunatik. Para que tal operação possa ser realizada é necessário informar o nome do estado tal que o código será executado, juntamente com o nome, o código em si também precisa ser enviado para o estado, e tal informação é passada por meio do campo `associated_value`. 
### `LUNATIK_GETSTATE`
Operação responsável pela obtenção dos estados já existentes no kernel. Essa operação necessita somente do nome do estado para ser realizada.
### `LUNATIK_PUTSTATE`
Operação utilizada para devolução de referências de estados adquiridos por meio da operação `LUNATIK_GETSTATE`, também necessita somente da informação do nome do estado a ser devolvido a referência.
### `LUNATIK_DATA`
Operação responsável pelo envio de dados binários para os estados no kernel, é necessário enviar o nome do estado a se enviar o dado e o dado em si será enviado no campo `associated_value`.

## Binding Lunatik

Definição das constantes e funções do binding, seus parâmetros e retornos.

### Constantes

- `lunatik.datamaxsize`: Inteiro que representa a quantidade máxima em bytes de dados que um estado pode receber por mensagem.
- `lunatik.defaultmaxallocbytes`: Inteiro que representa o alocamento padrão, em bytes, de um estado.
- `lunatik.maxstates`: Inteiro que representa a quantidade máxima de estados que o Lunatik pode ter por ambiente.
- `lunatik.statenamemaxsize`: Inteiro que representa o tamanho máximo permitido da string utilizada para a criação do estado. 

### Sessão Lunatik
Para realizar as operações presentes no binding Lunatik é necessário primeiro estabelecer uma sessão de comunicação com o Lunatik. A função responsável por estabelecer tal sessão é a `lunatik.session()` e um exemplo de seu uso é mostrado a seguir.
- Retorno: Userdata utilizado para acessar os métodos de controle do gerenciados de estados.
```lua
local lunatik = require'lunatik'
session = lunatik.session()
```
Aqui estão descritas as operações disponíveis neste userdata:
- **Criação de estados:**
	- Parâmetros:
	`name`: Nome do estado a ser criado.
	`maxalloc`: Total de memória, em bytes, que o estado pode alocar no kernel. Caso nenhum valor seja informado o valor padrão `lunatik.defaultmaxallocbytes` será associado.
	- Retorno: Um userdata representando o estado criado em caso de sucesso e `nil` caso contrário.
	- Assinatura: `session:newstate(name [, maxalloc])`
 - **Fecha a sessão**:
	 - Parâmetros:
	 Nenhum
	 - Retorno: Retorna `true` caso a sessão seja fechada com sucesso e `nil` caso contrário.
	 - Assinatura: `session:close()`
 - **Obtém um estado no kernel**:
	 - Parâmetros:
		 - `name`: Nome do estado a se obter
	 - Retorno: Um userdata para realizar as operações disponíveis para o estado requisitado ou `nil` caso tal estado não exista no kernel.
	 - Assinatura: `session:getstate(name)`
- **Listagem de estados**:
	- Parâmetros:
	Nenhum
	- Retorno: Uma tabela com cada elemento representando um estado presente no ambiente padrão do Lunatik. Cada elemento tem o seguinte formato:
	```lua
	{
		name = "Nome do estado",
		current_alloc = 43215,
		max_alloc = 50000
	}
	```
	- Assinatura: `session:list()`
## API
Definição das funções, seus parâmetros e retornos.

### Funções com ambientes implícitos:
- **Criação de estado:** 
	- Parâmetros:
		`name`: Nome do estado, serve como identificador único do estado.
		`maxalloc`: Número que representa, em bytes, a quantidade máxima de memória que o estado a ser criado pode alocar no kernel.
	- Retorno: Em caso de sucesso, retorna um ponteiro para o estado criado. Retorna `NULL` caso o estado não possa ser criado.
	- Assinatura: `lunatik_State *lunatik_newstate(const char *name, size_t maxalloc)`
- **Exclusão de estado:**
	- Parâmetros: 
		`name`: Nome do estado a ser excluído
	- Retorno: Retorna 0 em caso de sucesso e um número diferente de 0 caso contrário.
	- Assinatura: `int lunatik_close(const char *name)`
- **Busca de estado:**
	- Parâmetros:
		`name`: Nome do estado a ser procurado.
	- Retorno: Caso o estado exista, retorna um ponteiro para este. Retorna `NULL` caso contrário.
	- Assinatura: `lunatik_State *lunatik_statelookup(const char *name)`

### Funções com ambientes explícitos:
- **Criação de estado:**
	- Parâmetros:
		`name`: Nome do estado a ser criado, utilizado como identificador único para o estado naquele ambiente.
		`maxalloc`: Número que representa, em bytes, a quantidade máxima de memória que o estado a ser criado pode alocar no kernel.
		`env`: Ambiente no qual o estado será criado.
	- Retorno: Caso o estado seja criado com sucesso, retorna um ponteiro para o estado. Retorna `NULL` caso contrário.
	- Assinatura: `lunatik_State *lunatik_netnewstate(const char *name, size_t maxalloc, struct net *env)`
- **Exclusão de estado:**
	- Parâmetros:
		`name`: Nome do estado a ser fechado.
		`env`: Ambiente a se procurar o estado a ser fechado pelo `name`.
	- Retorno: Retorna 0 caso o estado seja fechado com sucesso e um número diferente de 0 caso contrário.
	- Assinatura: `int lunatik_netclosestate(const char *name, struct net *env)`
- **Busca de estado:**
	- Parâmetros:
		`name`: Nome do estado a ser procurado.
		`env`: Ambiente a se procurar o estado pelo `name`.
	- Retorno: Retorna um ponteiro para o estado caso este exista no ambiente `env` ou `NULL` caso contrário.
	- Assinatura: `lunatik_State *lunatik_netstatelookup(const char *name, struct net *env)`

### Funções independentes de ambientes:

- **Obtenção de referência de um estado:**
	- Parâmetros:
		`state`: Ponteiro para o estado cuja a referência quer se obter.
	- Retorno: `true` caso a referência seja obtida com sucesso e `false` caso contrário.
	- Assinatura: `bool lunatik_getstate(lunatik_State *state)`
- **Devolução de referência de um estado:**
	- Parâmetros:
		`state`: Ponteiro para o estado cuja a referência quer se devolver.
	- Retorno: `true` caso a referência seja devolvida com sucesso e `false` caso contrário.
	- Assinatura: `bool lunatik_getstate(lunatik_State *state)`são mostradas a seguir:




## <a name="glossary"></a>Glossário

- <a name="estado_lunatik"></a>**Estado Lunatik:** Um estado Lunatik é uma estrutura de dados opaca que serve como uma capsula para os estados Lua e tem o propósito de abstrair todas as operações relacionadas ao gerenciamento de estados em Lua.
- <a name="ambientes_lunatik"></a> **Ambiente Lunatik:** Um ambiente Lunatik é uma instância de armazenamento de estados do Lunatik. Em outras palavras, a estrutura usada pelo Lunatik para armazenar seus estados estará sempre associada a um net namespace, mesmo que não especificada pelo usuário. Neste caso, estará associada ao net namespace do processo `init` do Linux. Isso faz com que o Lunatik possa prover aos seus usuários um isolamento de recursos.
