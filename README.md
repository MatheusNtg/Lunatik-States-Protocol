

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
O gerenciamento de estados do Lunatik é feito em seus respectivos namespaces. Cada namespace possui seu próprio conjunto de estados isolados entre si. A figura abaixo ilustra em alto nível como tal isolamento é organizado. 

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/MCD45yy/diagramas-namespaces.png)

Vale ressaltar que os namespaces Lunatik são construídos sobre os net namespaces do linux. Essa escolha implica que a execução da API em um determinado namespace fará a API ter acesso somente aos estados que estão presente naquele namespace, assim como ilustrado na figura a seguir.

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/6NrcB6V/diagramas-namespace-kernel-user-space.png)

## Protocolo de comunicação

### Generic Netlink
Antes de entendermos o protocolo de comunicação, precisamos entender a estrutura dos pacotes do generic netlink. O generic netlink funciona como um [multiplexador de um barramento reservado do netlink](https://wiki.linuxfoundation.org/networking/generic_netlink_howto#architectural_overview). Tal multiplexação consegue ser obtida a partir do registro e gerenciamento de famílias que são organizadas utilizando strings únicas por famílias. O barramento do netlink usado pelo generic netlink é utilizado como camada de transporte, além disso, para realizar a troca de dados entre user space e o kernel, a [API de atributos do netlink é utilizada](https://elixir.bootlin.com/linux/latest/source/include/net/netlink.h#L315). 

As famílias do generic netlink são compostas por operações e atributos. As operações são do um conjunto de callbacks associados a valores únicos, enquanto que os atributos são os dados utilizados pela família para a realização de suas operações. Assim como os callbacks, os atributos também estão associados a um valor específico, desta forma temos uma estrutura conforme apresentada na figura a seguir:

![Figura](https://i.ibb.co/Rjxy99p/diagramas-Familia-generic-netlinl-1.png)

Dessa forma, para enviar uma mensagem do user space para o kernel é necessário preencher os campos da família a qual o pacote se destina, a operação a ser realizada na família e os atributos para aquela operação (associados aos seus respectivos valores em suas famílias). 

## Operações do protocolo
Esta seção visa descrever o funcionamento do protocolo das operações suportadas pela API do user space do Lunatik. Tais operações devem seguir a enumeração a seguir para serem associadas aos seus respectivos callbacks registrados na criação da família lunatik do generic netlink.
```c
enum lunatik_operations {
	LUNATIK_NEWSTATE = 1,
	LUNATIK_CLOSE,
	LUNATIK_DOSTRING,
	LUNATIK_GETSTATE,
	LUNATIK_PUTSTATE,
	LUNATIK_SENDDATA
};
```

Além das operações, os atributos também são uma enumeração necessária para a realização das operações no kernel e devem seguir a enumeração a seguir.

```c
enum lunatik_attrs {
	STATE_NAME,
	MAX_ALLOC,
	OPERATION_RESULT,
	ATTRS_COUNT
#define ATTR_MAX (ATTRS_COUNT - 1)
};
```

Toda operação esta associada a uma familía, dessa forma, antes da execução de qualquer operação descrita a seguir, é necessário o preenchimento de dois campos no header do pacote generic netlink, sendo eles: A família a qual o pacote se destina, e a operação em questão. O primeiro campo sempre será o valor definido na macro `LUNATIK_FAMILY` cujo o valor é "lunatik_string". Já o segundo valor pode ser qualquer valor presente em `enum lunatik_operations` desde que seguido as restrições definidas pelas operações.

O kernel notifica o user space sobre o resultado da solicitação da operação através do socket utilizado para receber a mensagem. Dessa forma, todas as operações descritas a seguir devem registrar um callback associado ao socket utilizado para enviar a mensagem para o kernel. Tal callback deve lidar com o resultado da operação solicitada, isto é, caso o kernel notifique o user space que a solicitação da operação não ocorreu da forma esperada, então a API deve retornar um erro para o usuário. Para maiores informações de como escrever tais funções utilizando a libnl visite este [link](https://www.infradead.org/~tgr/libnl/doc/core.html#core_attr_parse)

### Criação de estados

Representado pela enumeração `LUNATIK_NEWSTATE`, esta é a operação responsável pela criação de estados no kernel e também por oferecer ao usuário uma estrutura opaca que represente o estado presente no kernel com suas possíveis operações que são: Envio de códig

O usuário da API precisa informar um nome para o estado que se deseja criar, e,  opcionalmente, um valor de alocamento máximo de memória do estado no kernel. O valor de alocamento máximo será associado ao atributo `MAX_ALLOC` da mensagem a ser enviada ao generic netlink enquanto que o nome do estado estará associado ao atributo `STATE_NAME`. Caso o usuário não informe um valor de alocamento máximo, um valor padrão deve ser associado ao `MAX_ALLOC`.

Caso a operação de criação de estado seja realizada com sucesso, o kernel irá retornar o valor `0` no atributo `OPERATION_RESULT`, caso contrário, o valor neste campo será `1`.

Um exemplo de código para realizar tal operação utilizando a [libnl](https://www.infradead.org/~tgr/libnl/) é mostrado a seguir:

```c
...
#define LUNATIK_FAMILY "lunatik_family"
...
int func(struct nl_msg *msg, void *arg)
{
	struct nlmsghdr *nh = nlmsg_hdr(msg);
	struct genlmsghdr *gnlh = genlmsg_hdr(nh);
	struct nlattr * attrs_tb[ATTRS_COUNT + 1];
	
	/* Coloca os atributos recebidos do kernel na tabela "attrs_tb" */
	nla_parse(attrs_tb, ATTRS_COUNT, genlmsg_attrdata(gnlh, 0), genlmsg_attrlen(gnlh, 0), NULL))
	
	/* Verifica o resultado da operação no kernel */
	if (attrs_tb[OPERATION_RESULT] && nla_get_u8(attrs_tb[OPERATION_RESULT]) == 0) {
		*arg = 0;
	} else {
		*arg = 1;
	}

	return NL_OK;
}
...

/* Aloca um socket para realizar a comunicação */
struct nl_sock *socket = nl_socket_alloc();
/* Aloca um pacote para ser enviado para o kernel */
struct nlmsg *msg = nlmsg_alloc();
/* Variável que armazena o resultado da operação no kernel */

/* Conecta ao generic netlink */
genl_connect(socket); 

/* 
Registra a função "func" como callback das mensagens recebidas do kernel
*/
nl_socket_modify_cb(socket, NL_CB_MSG_IN, NL_CB_CUSTOM, func, err);

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

if(err) /* Retorne para o usuário um erro*/

/* 
Retorne para o usuário uma estrutura representando o estado no kernel
como descrito no inicio da seção
*/
```
