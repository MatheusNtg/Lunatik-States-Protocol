# Lunatik States Management Protocol

## Introdução

Este documento tem por objetivo definir e documentar o funcionamento do protocolo da API de gerenciamento de estados do Lunatik (Lunatik States Management Protocol) no espaço de usuário. Algumas suposições devem ser feitas a fim de estabelecer o funcionamento correto do protocolo, tais suposições são:
- O módulo Lunatik deve estar carregado no kernel;
- O módulo Lunatik faz o uso do generic netlink para se comunicar com o user space;
- O identificador da família do Lunatik no generic netlink é a string "lunatik_family", neste protocolo sendo definada pela macro `LUNATIK_FAMILY`;
- Uma estrutura que encapsula os estados Lua no kernel existe e esta é opaca para o usuário do módulo;
- A família do Generic Netlink está criada e configurada com todos os atributos e operações necessárias para o funcionamento deste protocolo (para maiores informações sobre as famílias do generic netlink acesse este [link](https://wiki.linuxfoundation.org/networking/generic_netlink_howto#the_genl_family_structure)).


## Estrutura alto nível do protocolo

A figura a seguir apresenta a estrutura em alto nível do gerenciamento de estados do Lunatik:

![enter image description here](https://i.ibb.co/s30v6Sc/diagramas-estrutura-geral.png)

Figura 1. Estrutura geral do funcionamento do gerenciamento de estados no Lunatik

Como mostrado na figura 1, existe uma API Lua para o Lunatik no user space que oferece todas as operações disponíveis neste protocolo. Tal API envia e recebe informações de controle como dados para o kernel e vice-versa. O uso de dois canais diferentes para o envio de dados e de controle facilita a implementação deste protocolo, evitando o conflito no envio e recebimento de mensagens. 

## <a name="namespace"></a> Namespace Lunatik
O gerenciamento de estados do Lunatik é feito em seus respectivos namespaces, cada namespace possui seu próprio conjunto de estados fazendo com estes sejam isolados entre si. A figura 2 dá uma noção em alto nível de como este isolamento é organizado. 

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/MCD45yy/diagramas-namespaces.png)

Fig 2. Estrutura geral dos namespaces Lunatik

Vale ressaltar também que os namespaces Lunatik são construídos em cima dos net namespaces do linux, esta escolha implica que a execução da API em um determinado namespace fará a API ter acesso somente aos estados que estão presente naquele namespace, assim como ilustrado na figura a seguir.

![Fig2. Estrutura geral dos ambientes Lunatik](https://i.ibb.co/ykwy2J4/diagramas-namespace-kernel-user-space.png)
Fig. 3: Acesso da API em cada namespace

## Protocolo de comunicação

### Generic Netlink
Antes de entendermos o protocolo de comunicação precisamos entender a estrutura dos pacotes do generic netlink. O generic netlink funciona como um [multiplexador de um barramento reservado do netlink](https://wiki.linuxfoundation.org/networking/generic_netlink_howto#architectural_overview), tal multiplexação consegue ser obtida a partir do registro e gerenciamento de famílias que são registradas utilizando strings únicas por famílias. O barramento do netlink usado pelo generic netlink é utilizado como camada de transporte, além disso, para realizar a troca de dados entre o kernel e o user space, a [API de atributos do netlink é utilizada](https://elixir.bootlin.com/linux/latest/source/include/net/netlink.h#L315). Dessa forma, para enviar uma mensagem para o kernel a partir do user space é necessário preencher os campos da família a ser enviado o pacote, a operação a ser realizada na família e os atributos para aquela operação. Um exemplo de código que realiza tal operação utilizando a [libnl](https://www.infradead.org/~tgr/libnl/) é mostrado a seguir:

```c
...
/* 
Aloca o pacote a ser enviado para o kernel
*/
struct nl_msg *msg = nlmsg_alloc(); 

/*
Coloca dentro do pacote "msg" a família para qual o pacote se destina
identificado nos argumentos da função pela macro LUNATIK_FAMILY
e identifica a operação como o valor presente na macro "OPERATION"
*/
genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, LUNATIK_FAMILY, 0, 0, OPERATION, 1);

/*
Coloca um atributo unsigned int de 8 bits de valor 5 no pacote "msg"
associado a chave presente na macro ATTRIBUTE
*/
NLA_PUT_U8(msg, ATTRIBUTE, 5);

...
```

Os atributos têm que estar associados a algum valor específico como mostrado na macro `NLA_PUT_U8` para que a serialização e desserialização possa ocorrer.  Vale ressaltar que o código mostrado anteriormente não trata erros numa implementação real tais tratamentos devem ocorrer, a fim de tratar tais erros consulte a documentação da libnl disponível [aqui](https://www.infradead.org/~tgr/libnl/doc/api/group__genl.html).


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

Além das operações, os atributos também são uma enumeração necessária para a realização das operações no kernel e devem seguir a enumeração a seguir pois os atributos estão relacionados a um valor que serve como chave para aquele atributo, tal enumeração é: (esta enumeração ainda está incompleta e somente os valores utilizados pela operação de criação de estado é que foram preenchidos.)

```c
enum lunatik_attrs {
	STATE_NAME,
	MAX_ALLOC,
};
```

### Criação de estados

Representado pela enumeração `LUNATIK_NEWSTATE` esta é a operação responsável pela criação de estados no kernel a partir do user space e também por oferecer ao usuário uma estrutura opaca que represente o estado presente no kernel com suas possíveis operações que são: Envio de código para este estado, envio de dados para este estado e encerramento de conexão com este estado entre o kernel e o user space.

A fim de realizar tal operação primeiro precisamos criar um socket netlink que irá realizar a comunicação entre o kernel e o user space, com a libnl podemos fazer isso da seguinte forma:
```c
struct nl_sock *socket = nl_socket_alloc();
```

Uma vez que o socket foi alocado, precisamos conectá-lo ao generic netlink e registrar um callback para o recebimento das mensagens enviadas pelo kernel. Este callback deve lidar com o resultado da operação no kernel, pois este irá enviar uma mensagem informando se tal operação ocorreu com sucesso ou não. Na libnl fazemos isso da seguinte forma:
```c
/* Conecta ao generic netlink */
genl_connect(socket); 

/* 
Registra a função "func" como callback das mensagens recebidas do kernel
*/
nl_socket_modify_cb(socket, NL_CB_MSG_IN, NL_CB_CUSTOM, func, err);
```

Uma vez alocado e preparado o socket que será utilizado para a comunicação entre o kernel e o user space, deve-se preparar a mensagem a ser enviada.

```c
struct nlmsg *msg = nlmsg_alloc();
```

Após o alocamento da mensagem o header deve ser preenchido com as informações respectivas à criação de estado.

```c
genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, LUNATIK_FAMILY, 0, 0, LUNATIK_NEWSTATE, 1);
```
Após isso, é necessário informar os atributos para a realização da criação do estado no kernel, tal informação é recebida através do usuário da API que informa o nome do estado e um número inteiro que representa o alocamento máximo de memória do estado Lua no kernel. Lembrando que este último parâmetro é opcional para o usuário e caso não seja informado, um valor padrão deve ser preenchido nos atributos da mensagem. 
```c
/* A variável nome_do_estado armazena o nome do estado informado pelo usuário */
NLA_PUT_STRING(msg, STATE_NAME, nome_do_estado);

/* A variável alocamento máximo armazena o alocamento máximo informado pelo usuário */
NLA_PUT_U32(msg, MAX_ALLOC, alocamento_maximo);
```
Assim que a mensagem estiver finalizada podemos enviá-la ao kernel:

```c
nl_send_auto(socket, msg);
```
No caso da libnl devemos esperar primeiro pelo retorno da mensagem pelo kernel e depois pelo `ACK` da mensagem.
```c
/* Neste momento lidaremos com o retorno da operação pelo kernel */
nl_recvmsgs_default(socket);

/* ACK */
nl_wait_for_ack(socket);
```

Por fim, podemos liberar o socket e retornar ao usuário uma estrutura que representa o estado presente no kernel como informado no início desta seção.
```c
nl_socket_free(socket);
```
