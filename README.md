

# Lunatik States Management Protocol

## Introdução

Este documento tem por objetivo definir e documentar o funcionamento do protocolo de gerenciamento de estados do Lunatik (Lunatik States Management Protocol). Este protocolo por sua vez tem o propósito de oferecer uma interface entre o kernel e o user-space para o gerenciamento de estados do Lunatik.

## Estados Lunatik

Um estado Lunatik é uma estrutura contida no Lunatik que encapsula um estado Lua a fim de proporcionar um gerenciamento de isolamento de recursos, envio e recebimento de informações do user-space, comunicação entre o kernel e o user-space, armazenamento e execução de código Lua, controle de concorrência e gerenciamento de memória deste estado Lua. 

A estrutura do estado Lunatik pode ser vista a seguir:
```c
typedef struct lunatik_state {
	struct hlist_node node;
	struct lunatik_namespace *namespace;
	struct genl_info usr_space_info;
	struct lunatik_data data;
	lua_State *L;
	char *code_buffer;
	int buffer_offset;
	spinlock_t lock;
	refcount_t users;
	size_t maxalloc;
	size_t curralloc;
	size_t scriptsize;
	bool inuse;
	unsigned char name[LUNATIK_NAME_MAXSIZE];
} lunatik_State;
```
O significado de cada um dos campos dentro da estrutura `lunatik_State` é:
- `node`: Cabeça de uma lista ligada que é utilizada para armazenamento do estado Lunatik em uma tabela hash dentro do kernel.
- `namespace`: Namespace que o estado Lunatik pertence, para maiores informações consulte [namespaces Lunatik](#namespace).
- `usr_space_info`: Estrutura que armazena as informações necessárias para a comunicação com o espaço de usuário.
- `data`: Estrutura para armazenar os dados enviados do e para o espaço de usuário.
- `L`: Estado Lua a ser gerenciado.
- `code_buffer`: Buffer para armazenamento do código a ser executado no estado Lua.
- `buffer_offset`: Offset do buffer para sincronizar código que precisa ser enviado em múltiplas etapas.
- `lock`: Lock utilizado para controle de concorrência das operações internas do gerenciamento de estados.
- `users`: Contador de referências a este estado, é utilizado para auxiliar no controle de concorrência e evitar condições de exclusão de estados que estão sendo utilizados.
- `maxalloc`: Quantidade de memória máxima que o estado Lua pode alocar no Kernel.
- `curralloc`: Quantidade de memória atual sendo utilizada pelo estado Lua.
- `scriptsize`: Tamanho do script que está sendo armazenado na variável `code_buffer`.
- `inuse`: Valor que identifica se o estado está sendo utilizado no espaço de usuário. Serve para evitar que dois scripts Lua no espaço de usuário usem o mesmo estado ao mesmo tempo.
- `name`: String utilizada como identificador único ao estado Lua. As operações sobre um estado específico são realizadas baseadas neste valor.


## Estrutura alto nível do protocolo

A figura a seguir apresenta a estrutura em alto nível do gerenciamento de estados do Lunatik:

![enter image description here](https://i.ibb.co/p1CKwn0/diagramas-estrutura-geral.png)

Figura 1. Estrutura geral do funcionamento do gerenciamento de estados no Lunatik

Como mostrado na figura 1, existe um binding Lua para o Lunatik que oferece todas as operações disponíveis neste protocolo como um módulo Lua. Tal binding envia e recebe informações de controle como dados para o kernel e vice-versa. O uso de dois canais diferentes para o envio de dados e de controle facilita a implementação deste protocolo, evitando o conflito no envio e recebimento de mensagens. 

A utilização de um binding Lua para a realização das operações presentes neste protocolo unifica a ideia de utilização de Lua, uma vez que esta é a linguagem utilizada no Lunatik, no entanto, este protocolo pode ser adaptado para outras linguagens, desde que as definições estabelecidas neste documento sejam seguidas.

## <a name="namespace"></a> Namespace Lunatik
O gerenciamento de estados do Lunatik é feito em seus respectivos namespaces, cada namespace possui seu próprio conjunto de estados fazendo com estes sejam isolados entre si. A figura 2 dá uma noção em alto nível de como este isolamento é organizado. 

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/vxhZs8C/diagramas-namespaces.png)

Fig 2. Estrutura geral dos namespaces Lunatik

Como cada namespace possui seus próprios conjuntos de estados, isso implica que esses conjuntos são isolados e não se comunicam entre si. Isso faz com que a existência de dois estados Lunatik com o mesmo nome seja possível, desde que estes estados estejam associados a namespaces diferentes.

Vale ressaltar também que os namespaces Lunatik são construídos em cima dos net namespaces do linux, esta escolha facilita a implementação dos namespaces Lunatik uma vez que toda a execução de comunicação entre o espaço de usuário e o kernel está associado a um net namespace no linux. 

Como um namespace Lunatik sempre está associado ao net namespace da comunicação entre o kernel e o espaço de usuário temos uma estrutura para a realização das operações do Lunatik a partir do espaço de usuário da seguinte forma:

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/QnW5vtd/diagramas-namespace-kernel-user-space.png)

Podemos ter N namespaces como mostado na figura anterior, cada namespace é isolado e não compartilham recursos, dessa forma, no espaço de usuário, temos processos do Binding Lua para o Lunatik separados e isolados um dos outros, vale ressaltar que todos estes processos podem coexistir. Já no Kernel, a API de namespaces do Linux permite o armazenamento de qualquer estrutura dentro dos namespaces criados. Como dito anteriormente, um namespace Lunatik é um conjunto de estados isolados que podem coexistir pois estão lógicamente separados. Sendo assim, toda vez que um determinado net namespace é criado no Linux, um namespace Lunatik também é criado e é associado a este namespace, isto é, a estrutura que armazena os estados Lunatik é colocada dentro do net namespace criado, por isso a cor igual tanto para o net namespace quanto para o namespace Lunatik. Isso faz com que o gerenciamento de estados esteja contido em seu próprio namespace, em outras palavras, um Binding Lua para o Lunatik só irá se comunicar com o seu respectivo namespace Lunatik. Por fim, perceba que o Lunatik existe em todos os namespaces, pois não é ele que é isolado e sim os seus namespaces.

## Protocolo de comunicação

### Estrutura do datagrama
Antes de entendermos o protocolo de comunicação precisamos entender a estrutura dos pacotes utilizados. Uma vez que uma das formas de realizar a comunicação entre o kernel e o user space se dá por meio da utilização do generic netlink e tal ferramenta serializa os dados, acabamos por ter um pacote da seguinte forma:

![Pacote](https://i.ibb.co/7kvJh5k/diagramas-pacote-1.png)

A descrição dos campos do pacote é a seguinte:

- `state_name`: Uma sequência de caracteres que representa o nome do estado a ser realizada a operação.
- `type`: Tipo do pacote (controle ou dados).
- `operation`: Operação a ser realizada no estado descrito no campo `state_name`. 
- `arg`: Ponteiro para argumentos extras em operações que necessitam de campos além dos que estão presentes no pacote.
- `arg_len`: Tamanho do dado passado no campo `arg`.

## Diagrama de sequência do protocolo
A figura a seguir ilustra o processo geral de troca de mensagens do protocolo.

![Diagrama de sequência](https://i.ibb.co/wBHH1Y0/diagramas-diagrama-sequencia-1.png)

## Descrição das Operações
Esta seção explora de forma mais detalhada como devem ser preenchido os pacotes enviados para o kernel a fim de implementar as operações oferecidas pelo binding Lua para o Lunatik. Tais operações são:

- Criação de sessão de comunicação com o Kernel;
- Encerramento de sessão de comunicação com o Kernel;
- Listagem dos estados presentes no Kernel a partir de uma sessão;
- Criação de estados Lunatik não existentes;
- Encerramento da comunicação com o estado no Kernel;
- Envio e execução de códigos para os estados Lunatik;
- Obtenção dos estados Lunatik existentes;
- Devolução de referência para os estados Lunatik obtidos;
- Envio de dados do kernel para o user space e vice-versa.

Como podemos observar na listagem anterior, temos operações relacionadas à sessão de comunicação com o kernel e operações de manipulações de estados, que são realizadas a partir de uma sessão de comunicação entre o user space e o kernel. Sendo assim, primeiro precisamos definir como tal sessão é realizada.

Uma sessão de comunicação entre o kernel e o user space é feita utilizando a conexão com a família do Generic Netlink definida no kernel a partir do espaço de usuário. A família do Generic Netlink é o que define os atributos utilizados pela família bem como as funções a serem chamadas assim que uma mensagem de um determinado tipo chega no kernel.


- **Criação de estados:** Para realizar a criação dos estados é necessário preencher o campo `state_name` do pacote com o nome do estado informado pelo usuário no momento do solicitação da operação. O campo `arg` apontará para o tamanho máximo de alocamento do estado. Na visão do usuário este valor é opcional, em outras palavras, ao solicitar esta operação, o usuário não precisa o informar e caso isto ocorra um valor padrão deverá ser atribuído.
- **Exclusão de estados:** Exclui, um estado Lunatik. Essa exclusão é feita removendo o estado da estrutura de armazenamento de estados do ambiente Lunatik e só pode ser realizada caso só exista uma única refêrencia ao estado. Caso contrário, mesmo que a operação seja solicitada pelo usuário, nenhuma alteração na estrutura de armazenamento de estados do Lunatik ocorrerá. Retorna 0 caso o estado tenha sido excluído e um número negativo caso contrário.
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
		- `name`: Nome do estado a ser criado.
		- `maxalloc`: Total de memória, em bytes, que o estado pode alocar no kernel. Caso nenhum valor seja informado o valor padrão `lunatik.defaultmaxallocbytes` será associado.
	- Retorno: Um userdata representando o estado criado em caso de sucesso e `nil` caso contrário.
	- Assinatura: `session:newstate(name [, maxalloc])`
 - **Fecha a sessão**:
	 - Parâmetros:
	 	- Nenhum
	 - Retorno: Retorna `true` caso a sessão seja fechada com sucesso e `nil` caso contrário.
	 - Assinatura: `session:close()`
 - **Obtém um estado no kernel**:
	 - Parâmetros:
		 - `name`: Nome do estado a se obter
	 - Retorno: Um userdata para realizar as operações disponíveis para o estado requisitado ou `nil` caso tal estado não exista no kernel.
	 - Assinatura: `session:getstate(name)`
- **Listagem de estados**:
	- Parâmetros:
		- Nenhum
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
		- `name`: Nome do estado, serve como identificador único do estado.
		- `maxalloc`: Número que representa, em bytes, a quantidade máxima de memória que o estado a ser criado pode alocar no kernel.
	- Retorno: Em caso de sucesso, retorna um ponteiro para o estado criado. Retorna `NULL` caso o estado não possa ser criado.
	- Assinatura: `lunatik_State *lunatik_newstate(const char *name, size_t maxalloc)`
- **Exclusão de estado:**
	- Parâmetros: 
		- `name`: Nome do estado a ser excluído
	- Retorno: Retorna 0 em caso de sucesso e um número diferente de 0 caso contrário.
	- Assinatura: `int lunatik_close(const char *name)`
- **Busca de estado:**
	- Parâmetros:
		- `name`: Nome do estado a ser procurado.
	- Retorno: Caso o estado exista, retorna um ponteiro para este. Retorna `NULL` caso contrário.
	- Assinatura: `lunatik_State *lunatik_statelookup(const char *name)`

### Funções com ambientes explícitos:
- **Criação de estado:**
	- Parâmetros:
		- `name`: Nome do estado a ser criado, utilizado como identificador único para o estado naquele ambiente.
		- `maxalloc`: Número que representa, em bytes, a quantidade máxima de memória que o estado a ser criado pode alocar no kernel.
		- `env`: Ambiente no qual o estado será criado.
	- Retorno: Caso o estado seja criado com sucesso, retorna um ponteiro para o estado. Retorna `NULL` caso contrário.
	- Assinatura: `lunatik_State *lunatik_netnewstate(const char *name, size_t maxalloc, struct net *env)`
- **Exclusão de estado:**
	- Parâmetros:
		- `name`: Nome do estado a ser fechado.
		- `env`: Ambiente a se procurar o estado a ser fechado pelo `name`.
	- Retorno: Retorna 0 caso o estado seja fechado com sucesso e um número diferente de 0 caso contrário.
	- Assinatura: `int lunatik_netclosestate(const char *name, struct net *env)`
- **Busca de estado:**
	- Parâmetros:
		- `name`: Nome do estado a ser procurado.
		- `env`: Ambiente a se procurar o estado pelo `name`.
	- Retorno: Retorna um ponteiro para o estado caso este exista no ambiente `env` ou `NULL` caso contrário.
	- Assinatura: `lunatik_State *lunatik_netstatelookup(const char *name, struct net *env)`

### Funções independentes de ambientes:

- **Obtenção de referência de um estado:**
	- Parâmetros:
		- `state`: Ponteiro para o estado cuja a referência quer se obter.
	- Retorno: `true` caso a referência seja obtida com sucesso e `false` caso contrário.
	- Assinatura: `bool lunatik_getstate(lunatik_State *state)`
- **Devolução de referência de um estado:**
	- Parâmetros:
		- `state`: Ponteiro para o estado cuja a referência quer se devolver.
	- Retorno: `true` caso a referência seja devolvida com sucesso e `false` caso contrário.
	- Assinatura: `bool lunatik_getstate(lunatik_State *state)`são mostradas a seguir:




## <a name="glossary"></a>Glossário

- <a name="estado_lunatik"></a>**Estado Lunatik:** Um estado Lunatik é uma estrutura de dados opaca que serve como uma capsula para os estados Lua e tem o propósito de abstrair todas as operações relacionadas ao gerenciamento de estados em Lua.
- <a name="ambientes_lunatik"></a> **Ambiente Lunatik:** Um ambiente Lunatik é uma instância de armazenamento de estados do Lunatik. Em outras palavras, a estrutura usada pelo Lunatik para armazenar seus estados estará sempre associada a um net namespace, mesmo que não especificada pelo usuário. Neste caso, estará associada ao net namespace do processo `init` do Linux. Isso faz com que o Lunatik possa prover aos seus usuários um isolamento de recursos.
