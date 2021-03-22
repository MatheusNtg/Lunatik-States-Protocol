

# Lunatik States Management Protocol

## Introdução

Este documento tem por objetivo definir e documentar o funcionamento do protocolo da API de gerenciamento de estados do Lunatik (Lunatik States Management Protocol) no espaço de usuário. Algumas suposições devem ser feitas a fim de estabelecer o funcionamento correto do protocolo, tais suposições são:
1. O módulo Lunatik deve estar carregado no kernel;
2. O módulo Lunatik faz o uso do generic netlink para se comunicar com o user space;
3. A família do Generic Netlink está criada e configurada com todos os atributos e operações necessárias para o funcionamento deste protocolo (para maiores informações sobre as famílias do generic netlink acesse este [link](https://wiki.linuxfoundation.org/networking/generic_netlink_howto#the_genl_family_structure)).
4.  Os processos de criação e conexão do socket netlink não serão especificados neste protocolo mas devem realizados em cada uma das operações.


## Estrutura alto nível do protocolo

A figura a seguir apresenta a estrutura em alto nível do gerenciamento de estados do Lunatik:

![enter image description here](https://i.ibb.co/s30v6Sc/diagramas-estrutura-geral.png)

Como mostrado acima, existe uma API para o Lunatik no user space que oferece todas as operações disponíveis neste protocolo. Tal API envia e recebe informações de controle e dados para o kernel e vice-versa. O uso de dois canais diferentes para o envio de dados e controle facilita a implementação deste protocolo.

## <a name="namespace"></a> Namespace Lunatik
O gerenciamento de estados do Lunatik é feito em seus respectivos namespaces. A figura abaixo ilustra em alto nível como tal isolamento é organizado. 

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/MCD45yy/diagramas-namespaces.png)

Vale ressaltar que os namespaces Lunatik são construídos sobre os net namespaces do linux. Essa escolha implica que a execução da API em um determinado namespace fará a API ter acesso somente aos estados que estão presente naquele namespace, assim como ilustrado na figura a seguir.

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/6NrcB6V/diagramas-namespace-kernel-user-space.png)

## Protocolo de comunicação

### Generic Netlink
Antes de entendermos o protocolo de comunicação, precisamos entender a estrutura dos pacotes do generic netlink. O generic netlink funciona como um [multiplexador de um barramento reservado do netlink](https://wiki.linuxfoundation.org/networking/generic_netlink_howto#architectural_overview). Essa multiplexação consegue ser obtida a partir do registro e gerenciamento de famílias que são organizadas utilizando strings únicas por famílias. O barramento do netlink usado pelo generic netlink é utilizado como camada de transporte, além disso, para realizar a troca de dados entre user space e o kernel, a [API de atributos do netlink é utilizada](https://elixir.bootlin.com/linux/latest/source/include/net/netlink.h#L315). 

As famílias do generic netlink são compostas por operações e atributos. As operações são um conjunto de funções callbacks associadas a valores chaves únicos, enquanto que os atributos são os dados utilizados pelas operações. Assim como as funções callbacks, os atributos também estão associados a um valor chave específico. Os valores associados as operações são utilizados pelo generic netlink para decidir qual será a função callback a ser executada quando um determinado pacote chega para uma família. Já os valores associados aos atributos são utilizados para a serialização e deserialização de dados enviados nos pacotes.  Desta forma, a fim de enviar uma mensagem do user space para o kernel é necessário preencher os campos da **família** a qual o pacote se destina, a **operação** a ser realizada e os **atributos** para aquela operação. 

## Operações do protocolo
Esta seção visa descrever o funcionamento do protocolo das operações suportadas pela API do user space do Lunatik. Tais operações devem seguir a enumeração a seguir para serem associadas aos seus respectivos callbacks registrados na criação da família lunatik do generic netlink.
```c
enum lunatik_operations {
	LUNATIK_NEWSTATE = 1,
	LUNATIK_CLOSE    = 2,
	LUNATIK_DOSTRING = 3,
	LUNATIK_GETSTATE = 4,
	LUNATIK_SENDDATA = 5
};
```

Além das operações, os atributos também são uma enumeração necessária para a realização das operações no kernel e devem seguir a enumeração a seguir.

```c
enum lunatik_attrs {
	STATE_NAME       = 0,
	MAX_ALLOC        = 1,
	OPERATION_RESULT = 2,
	JSON_DATA	 = 3
};
```

Toda operação esta associada a uma família, dessa forma, antes da execução de qualquer operação descrita a seguir, é necessário o preenchimento de dois campos no header do pacote generic netlink, sendo eles: A família a qual o pacote se destina, e a operação em questão. O primeiro campo sempre será o valor definido na macro `LUNATIK_FAMILY` cujo valor é "lunatik_string". Já o segundo valor pode ser qualquer valor presente em `enum lunatik_operations` desde que seguindo as restrições definidas pelas operações.

O kernel notifica o user space sobre o resultado da solicitação da operação através do socket utilizado para receber a mensagem. Desta forma, todas as operações descritas a seguir devem registrar uma função callback associada ao socket utilizado para enviar a mensagem para o kernel. Essa função deve lidar com o resultado da operação solicitada, isto é, caso o kernel notifique o user space que a solicitação da operação não ocorreu da forma esperada, então a API deve retornar um erro para o usuário. Para maiores informações sobre a escrita dessas funções utilizando a libnl visite este [link](https://www.infradead.org/~tgr/libnl/doc/core.html#core_attr_parse)

### Criação de estados

Operação responsável pela criação de estados no kernel. Os seguintes atributos devem ser preenchidos no pacote a ser enviado para o kernel:

- **Operação**:
	- `LUNATIK_NEWSTATE`
- **Atributos**:
	- `STATE_NAME`: String terminada em `'\0'`, de tamanho máximo 64 bytes representando o nome do estado a ser criado.
	- `MAX_ALLOC`: Número inteiro de 32 bits que representa o alocamento máximo de memória permitido pelo estado no kernel.
	
O kernel informa o espaço usuário sobre o resultado da solicitação da operação enviando uma mensagem pelo mesmo socket utilizado para receber a mensagem:

- **Retorno**: O atributo `OPERATION_RESULT` é preenchido com o valor `0` caso a operação seja realizada com sucesso e em caso de erro o valor `1` estará presente neste atributo.

Um exemplo de código para realizar esta operação utilizando a [libnl](https://www.infradead.org/~tgr/libnl/) é mostrado a seguir:

```c
...
#define LUNATIK_FAMILY "lunatik_family"
...
/* 
* Processa a resposta enviada pelo kernel 
* @param msg Mensagem recebida pelo kernel
* @param arg Variável para a manipulação de dados dentro da função
* @return Inteiro representando o status do recebimento da mensagem
*/
int process_response(struct nl_msg *msg, void *arg)
{
	struct nlmsghdr *nh = nlmsg_hdr(msg);
	struct genlmsghdr *gnlh = genlmsg_hdr(nh);
	struct nlattr * attrs_tb[4];
	
	/* Coloca os atributos recebidos do kernel na tabela "attrs_tb" */
	nla_parse(attrs_tb, 3, genlmsg_attrdata(gnlh, 0), genlmsg_attrlen(gnlh, 0), NULL))
	
	/* Verifica o resultado da operação no kernel */
	if (attrs_tb[OPERATION_RESULT] && nla_get_u8(attrs_tb[OPERATION_RESULT]) == 0) {
		*arg = 0;
	} else {
		*arg = 1;
	}

	return NL_OK;
}
...
/*
* Envia a mensagem de criação de estado para o kernel
* @param nome_do_estado Nome do estado a ser criado
* @param alocamento_maximo Alocamento máximo do memória permitido pelo estado no kernel
*/
void send_new_state_msg(char *nome_do_estado, uint32_t alocamento_maximo)
{
	/* Aloca um socket para realizar a comunicação */
	struct nl_sock *socket = nl_socket_alloc();
	/* Aloca um pacote para ser enviado para o kernel */
	struct nlmsg *msg = nlmsg_alloc();
	/* Variável que armazena o resultado da operação no kernel */

	/* Conecta ao generic netlink */
	genl_connect(socket); 

	/* 
	Registra a função "process_response" como callback das mensagens recebidas do kernel
	*/
	nl_socket_modify_cb(socket, NL_CB_MSG_IN, NL_CB_CUSTOM, process_response, err);

	/* 
	Coloca as informações relativas a família a qual a mensagem
	se destina e a operação a ser realizada nesta família no header
	do pacote "msg"
	*/
	genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, LUNATIK_FAMILY, 0, 0, LUNATIK_NEWSTATE, 1);

	/* Preenche os atributos necessários para criação de estado */
	NLA_PUT_STRING(msg, STATE_NAME, nome_do_estado);
	NLA_PUT_U32(msg, MAX_ALLOC, alocamento_maximo);

	/* Envia a mensagem */
	nl_send_auto(socket, msg);

	/* 
	Recebe a mensagem do kernel informando o resultado da operação
	neste momento a função "func" é chamada
	*/
	nl_recvmsgs_default(socket);
	/* Espera o ACK */
	nl_wait_for_ack(socket);

	nl_socket_free(socket);
}
```
### Encerramento de estados

Esta é a operação responsável por informar ao kernel que um estado não está sendo mais utilizado pelo espaço de usuário. . Assim que um estado é encerrado ele não deve ser mais acessível pelo espaço de usuário a não se que o mesmo seja criado novamente ou, caso continue existindo, seja obtido pela operação de obtenção de estado. Um estado continuará existindo no kernel caso exista outra referência além do espaço do usuário a este estado.
- **Operação**:
	- `LUNATIK_CLOSE`
- **Atributos**:
	- `STATE_NAME`: String terminada em `'\0'`, de tamanho máximo 64 bytes representando o nome do estado a ser encerrado.
- **Retorno**:
	- Retorna `0` no atributo `OPERATION_RESULT` caso o estado tenha sido excluído no kernel e `1` caso contrário.

O envio de mensagem para o kernel segue o mesmo padrão da operação de criação de estados, somente a operação na função `genlmsg_put` e os atributos a serem colocados na mensagem são alterados, desta forma temos:
```c
...
/*
* Envia a mensagem de encerramento do estado
* @param nome_do_estado Nome do estado a ser encerrado
*/
void send_close_state_msg(char *nome_do_estado)
{
...
	genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, LUNATIK_FAMILY, 0, 0, LUNATIK_CLOSE, 1);
	NLA_PUT_STRING(msg, STATE_NAME, nome_do_estado);
...
}
...
```

### Obtenção de estado

Operação responsável por enviar as informações relativas a um estado existente no kernel e travar este estado para uso pela API do espaço do usuário.

- **Operação**:
	- `LUNATIK_GETSTATE`
- **Atributos**:
	- `STATE_NAME`: String terminada em `'\0'`, de tamanho máximo 64 bytes representando o nome do estado a ser obtido.
- **Retorno**:
	- Retorna um objeto `JSON` com as informações relativas ao estado no atributo `JSON_DATA`, caso o estado não exista no kernel então um objeto vazio é retornado. Um exemplo de objeto retornado é mostrado a seguir:
	```json
	{
		"nome_do_estado": "Um exemplo de nome",
		"max_alloc"     : 32768,
		"curr_alloc"    : 16423
	}
	```

Nesta operação, a função de processamento da resposta do kernel deve lidar com o retorno do kernel de forma apropriada. Isto é, deve criar uma representação no espaço de  usuário do estado presente no kernel e retornar essa representação para o usuário. Uma função exemplo que realiza tal processamento é mostrada a seguir:
```c
/* Representa o estado no espaço de usuário */
struct lunatik_us_state {
	char *name;
	uint32_t max_alloc;
	uint32_t curr_alloc;
}
...
/* 
* Processa a resposta enviada pelo kernel 
* @param msg Mensagem recebida pelo kernel
* @param arg Variável para a manipulação de dados dentro da função
* @return Inteiro representando o status do processamento da mensagem
*/
int process_response(struct nl_msg *msg, void *arg)
{
	struct nlmsghdr *nh = nlmsg_hdr(msg);
	struct genlmsghdr *gnlh = genlmsg_hdr(nh);
	struct nlattr * attrs_tb[4];
	struct lunatik_us_state *user_state = (struct lunatik_us_state*) arg;
	
	/* Coloca os atributos recebidos do kernel na tabela "attrs_tb" */
	nla_parse(attrs_tb, 3, genlmsg_attrdata(gnlh, 0), genlmsg_attrlen(gnlh, 0), NULL))
	
	/* Verifica o resultado da operação no kernel */
	if (attrs_tb[OPERATION_RESULT] && nla_get_u8(attrs_tb[OPERATION_RESULT]) == 0) {
		*arg = 0;
	} else {
		*arg = 1;
	}

	return NL_OK;
}
...
/* 
* Envia a mensagem de obtenção do estado para o kernel
* @param nome_do_estado Nome do estado a ser obtido no kernel */
void send_get_state_msg(char *nome_do_estado)
{
	struct nl_sock *socket = nl_socket_alloc();
	struct nlmsg *msg = nlmsg_alloc();
	struct lunatik_us_state *user_state; // Parei aqui

	genl_connect(socket); 

	nl_socket_modify_cb(socket, NL_CB_MSG_IN, NL_CB_CUSTOM, process_get_state_response, user_state);

	genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, LUNATIK_FAMILY, 0, 0, LUNATIK_GETSTATE, 1);
	NLA_PUT_STRING(msg, STATE_NAME, nome_do_estado);

	nl_send_auto(socket, msg);

	nl_recvmsgs_default(socket);
	nl_wait_for_ack(socket);

	nl_socket_free(socket);
}
```

