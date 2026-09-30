# Arquitetura do LFT

O LFT adota uma arquitetura orientada a **contêineres desacoplados de rede**, onde cada nó é instanciado sem a pilha de rede padrão do Docker (`--network=none`) e suas interfaces são criadas e conectadas pelo próprio framework no nível do kernel Linux.

----

## 1. Isolamento por Network Namespaces do Linux

Quando um contêiner é iniciado no Docker com a opção `--network=none`, o daemon cria um namespace de rede isolado contendo apenas a interface de loopback (`lo`).

Para permitir que o framework manipule as interfaces diretamente do host através de utilitários como `ip link` e `ip netns`, o LFT realiza o seguinte procedimento na inicialização de qualquer nó (método `__enableNamespace` da classe `Node`):

  - Obtém o PID do contêiner no host:
    <code>pid=$(docker inspect -f '{{.State.Pid}}' <nome_do_no>)</code>
  - Cria um link simbólico no diretório global de namespaces do Linux:
    <code>mkdir -p /var/run/netns/
ln -sfT /proc/$pid/ns/net /var/run/netns/<nome_do_no></code>

A partir desse momento, comandos como `ip -n <nome_do_no> ...` operam diretamente na pilha de rede do contêiner sem a sobrecarga de um `docker exec`.

----

## 2. Pares de Interfaces Virtuais (veth pairs)

A interconexão física entre dois nós é emulada por meio de um par de interfaces virtuais Ethernet (`veth`):

  * Cada ponta de um par `veth` age como um conector de cabo de rede Ethernet cruzado. O que entra por uma ponta sai imediatamente na outra.
  * O LFT cria o par no host:
    <code>ip link add <iface1> type veth peer name <iface2></code>
  * Move a interface `<iface1>` para o namespace do nó A:
    <code>ip link set <iface1> netns <nome_do_no_A>
ip -n <nome_do_no_A> link set <iface1> up</code>
  * Move a interface `<iface2>` para o namespace do nó B:
    <code>ip link set <iface2> netns <nome_do_no_B>
ip -n <nome_do_no_B> link set <iface2> up</code>

> **Importante**: No kernel Linux, nomes de interface de rede possuem um limite estrito de **15 caracteres** (definido pela constante `IFNAMSIZ`). Ao escolher nomes para as interfaces (ex.: `h1s1`, `s1h1`), garanta que o comprimento não ultrapasse 15 caracteres.

----

## 3. Integração com Open vSwitch (OVS)

Para nós do tipo `Switch`, o LFT utiliza o Open vSwitch em um contêiner com privilégios de kernel (`--privileged`):

  - Na inicialização do switch, é criada uma bridge OVS com o mesmo nome do nó:
    <code>docker exec <switch> ovs-vsctl add-br <switch>
docker exec <switch> ip link set <switch> up</code>
  - Quando um outro nó é conectado ao switch através do método `connect()`, a interface que entra no switch é automaticamente adicionada como porta na bridge OVS:
    <code>docker exec <switch> ovs-vsctl add-port <switch> <iface_do_switch></code>
  - O switch pode ser associado a um controlador OpenFlow externo via TCP:
    <code>docker exec <switch> ovs-vsctl set-controller <switch> tcp:<controller_ip>:<port></code>

----

## 4. Conectividade Externa, Roteamento e NAT

O LFT permite que a topologia emulada acesse a Internet através do host físico ou VM hospedeira:

  - Um par `veth` conecta a bridge do switch ao host hospedeiro (método `connectToInternet`).
  - Uma ponta permanece no namespace do host (ex.: `h_brint`) e recebe o IP do gateway (ex.: `192.168.100.101/24`).
  - Regras de **NAT / MASQUERADE** são aplicadas no `iptables` do host:
    <code>iptables -t nat -I POSTROUTING -o <default_gw_interface> -j MASQUERADE
iptables -t nat -I POSTROUTING -o <host_interface> -j MASQUERADE
iptables -A FORWARD -i <host_interface> -o <default_gw_interface> -j ACCEPT
iptables -A FORWARD -i <default_gw_interface> -o <host_interface> -j ACCEPT</code>
  - A interface do host é inserida na zona de confiança do firewall (`firewall-cmd --zone=trusted --add-interface=<host_interface>`).
  - Os nós internos recebem rotas padrão apontando para o IP do gateway configurado no host.

----

## 5. Controle e Modelação de Tráfego (Linux TC / Netem)

Cada interface de contêiner pode ter suas propriedades de enlace físicas controladas em tempo de execução via **Traffic Control (TC)** com o qdisc **netem**:

  * **Throughput (Largura de banda)**: ex.: `10mbit`, `1gbit`
  * **Latência (Delay)**: ex.: `20ms`, `100ms`
  * **Jitter (Variação de atraso)**: ex.: `5ms`
  * **Perda de pacotes (Packet Loss)**: percentual configurável

Esse controle é invocado através do método `setInterfaceProperties()` da classe `Node`.
