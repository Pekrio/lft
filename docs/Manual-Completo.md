# MANUAL COMPLETO: LIGHTWEIGHT FOG TESTBED (LFT)

*Documento unificado contendo toda a documentação, arquitetura, referência de API, catálogo de imagens e tutoriais da ferramenta LFT.*<br>
*Universidade de Brasília (UnB) - Laboratório COMNET*

----

# 1. VISÃO GERAL E INTRODUÇÃO

O **Lightweight Fog Testbed (LFT)** (evolução do *LST 2.0*) é um framework baseado em Python e contêineres Docker desenvolvido para simplificar a criação, emulação e teste de topologias de rede heterogêneas e leves para ambientes de **Fog Computing**, **Edge Computing** e **Segurança de Redes**.

Com o LFT, é possível orquestrar nós arbitrários em contêineres Docker, conectar dispositivos via pares virtuais Ethernet (`veth`) em *network namespaces* dedicados do Linux, gerenciar comutação programável via **Open vSwitch (OVS)**, controladores **SDN (ex.: Ryu, ONOS)**, emular enlaces sem fio **4G/LTE** utilizando **srsRAN** e coletar tráfego para análise com **CICFlowMeter** ou métricas de desempenho com **PerfSONAR**.

| Atributo | Detalhes |
| --- | --- |
| **Nome do Pacote** | ''profissa_lft'' |
| **Versão Atual** | 1.0.9 |
| **Linguagem** | Python >= 3.9 |
| **Licença** | GNU General Public License v3 (GPLv3) |
| **Instituição** | Universidade de Brasília (UnB) - Laboratório COMNET |
| **Repositório** | [[https://github.com/UnB-COMNET/lft | github.com/UnB-COMNET/lft]] |

----

# 2. REQUISITOS E INSTALAÇÃO

## 2.1 Requisitos do Sistema

O LFT interage diretamente com chamadas e subsistemas do **kernel Linux** (interfaces virtuais `veth`, manipulação de namespaces de rede em `/var/run/netns`, regras de firewall `iptables`/`firewalld`, e daemon `openvswitch-switch`).

| Plataforma | Suporte | Instruções |
| --- | --- | --- |
| **Ubuntu Desktop 24.04 LTS** | Nativo (Recomendado) | Instalação direta no host |
| **macOS (Apple Silicon / Intel)** | Via Máquina Linux (OrbStack / UTM) | Máquina virtual Ubuntu gerenciada via OrbStack |
| **Windows** | Via WSL 2 com suporte a systemd | Instalação na distribuição Ubuntu dentro do WSL 2 |

## 2.2 Instalação de Pacotes no Ubuntu

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y --no-install-recommends \
    docker.io \
    python3 \
    python3-pip \
    python3-pandas \
    iproute2 \
    iptables \
    net-tools \
    openvswitch-switch \
    firewalld \
    nfdump

sudo systemctl enable --now docker
sudo systemctl enable --now openvswitch-switch
sudo systemctl enable --now firewalld
sudo usermod -aG docker $USER
```

## 2.3 Instalação da Biblioteca Python

```bash
cd /caminho/para/lft
pip3 install --break-system-packages -e .
```

## 2.4 Execução no macOS (via OrbStack)

```bash
brew install --cask orbstack
orbctl create ubuntu:24.04 lft
orb -m lft -u root
cd /Users/<seu_usuario>/.../lft
pip3 install --break-system-packages -e .
```

----

# 3. ARQUITETURA INTERNA

## 3.1 Isolamento por Linux Network Namespaces

Cada nó do LFT é disparado no Docker com a opção `--network=none`, criando uma pilha de rede vazia contendo apenas a interface de loopback.

Na inicialização do nó, o framework localiza o PID do contêiner e cria um link simbólico no diretório global do Linux:
```bash
pid=$(docker inspect -f '{{.State.Pid}}' <nodeName>)
mkdir -p /var/run/netns/
ln -sfT /proc/$pid/ns/net /var/run/netns/<nodeName>
```
Isso viabiliza que comandos `ip -n <nodeName> ...` configurem a rede diretamente pelo kernel do host.

## 3.2 Pares Virtuais Ethernet (veth)

A interconexão entre dois nós é realizada por pares virtuais Ethernet:
```bash
ip link add <iface1> type veth peer name <iface2>
ip link set <iface1> netns <node1>
ip -n <node1> link set <iface1> up
ip link set <iface2> netns <node2>
ip -n <node2> link set <iface2> up
```

> **Atenção**: No kernel Linux, o comprimento máximo do nome de uma interface é de **15 caracteres** (`IFNAMSIZ`).

## 3.3 Open vSwitch (OVS)

Para nós da classe `Switch`, uma bridge homônima ao nó é criada dentro do contêiner OVS:
```bash
docker exec <switch> ovs-vsctl add-br <switch>
docker exec <switch> ip link set <switch> up
docker exec <switch> ovs-vsctl set-controller <switch> tcp:<controller_ip>:<port>
```

## 3.4 Conectividade com a Internet via NAT

O método `connectToInternet()` cria um par veth entre o switch e o host (ex.: `h_brint`), atribuindo o IP do gateway e habilitando as regras de encaminhamento:
```bash
iptables -t nat -I POSTROUTING -o <host_gw_interface> -j MASQUERADE
iptables -t nat -I POSTROUTING -o <host_iface> -j MASQUERADE
iptables -A FORWARD -i <host_iface> -o <host_gw_interface> -j ACCEPT
firewall-cmd --zone=trusted --add-interface=<host_iface>
```

----

# 4. REFERÊNCIA COMPLETA DA API

## 4.1 Classe Node (profissa_lft.node.Node)

Classe base de todos os elementos de rede.

| Método | Parâmetros | Descrição |
| --- | --- | --- |
| ''__init__(nodeName)'' | ''nodeName: str'' | Construtor do nó. |
| ''instantiate(...)'' | ''dockerImage'', ''dockerCommand'', ''dns'', ''memory'', ''cpus'', ''runCommand'' | Cria e dispara o contêiner com ''--network=none''. |
| ''delete()'' | Nenhum | Finaliza e remove o contêiner Docker. |
| ''connect(node, iface, peerIface)'' | ''node: Node'', ''iface: str'', ''peerIface: str'' | Interconecta nós criando um par ''veth''. |
| ''setIp(ip, mask, iface)'' | ''ip: str'', ''mask: int'', ''iface: str'' | Configura IP e máscara na interface informada. |
| ''setDefaultGateway(gwIp, iface)'' | ''gwIp: str'', ''iface: str'' | Define o gateway padrão para a interface. |
| ''addRoute(ip, mask, iface)'' | ''ip: str'', ''mask: int'', ''iface: str'' | Insere rota na tabela de roteamento interna do nó. |
| ''addRouteOnHost(ip, mask, iface, gateway)'' | ''ip: str'', ''mask: int'', ''iface: str'', ''gateway: str'' | Insere rota na tabela de roteamento do host. |
| ''connectToInternet(hostIP, hostMask, iface, hostIface)'' | ''hostIP: str'', ''hostMask: int'', ''iface: str'', ''hostIface: str'' | Habilita conectividade externa com NAT MASQUERADE. |
| ''setInterfaceProperties(iface, throughput, delay, jitter)'' | ''iface: str'', ''throughput: str'', ''delay: str'', ''jitter: str'' | Modela latência, jitter e largura de banda via ''tc netem''. |
| ''run(command: str)'' | ''command: str'' | Executa comando bash no nó (retorna ''subprocess.Popen''). |
| ''copyLocalToContainer(path, destPath)'' | ''path: str'', ''destPath: str'' | Copia arquivo do host para o contêiner. |
| ''copyContainerToLocal(path, destPath)'' | ''path: str'', ''destPath: str'' | Copia arquivo do contêiner para o host. |
| ''readConfigFile(containerPath)'' | ''containerPath: str'' | Lê arquivo INI e retorna ''ConfigParser''. |
| ''saveConfig(config, containerPath)'' | ''config: ConfigParser'', ''containerPath: str'' | Salva ''ConfigParser'' no contêiner. |

## 4.2 Classe Host (profissa_lft.host.Host)

Representa um host ou servidor convencional na rede. Herda todos os métodos de `Node`.

## 4.3 Classe Switch (profissa_lft.switch.Switch)

Representa um switch comutador Open vSwitch.
  * `instantiate(image='alexandremitsurukaihara/lst2.0:openvswitch', controllerIP=`, controllerPort=-1)''
  * `setController(ip: str, port: int)`: Conecta ao controlador OpenFlow.
  * `enableNetflow(bridgeName, destIp, destPort, activeTimeout=60)`: Exporta NetFlow.
  * `setFlow(rule: str)`: Insere regra OpenFlow na bridge.
  * `collectPackets(interfaceNames=[], path=`, rotateInterval=60)`: Inicia captura tshark rotativa (`path` deve terminar em `.pcap'').

## 4.4 Classe SwitchMeter (profissa_lft.switchmeter.SwitchMeter)

Herda de `Switch`. Integra captura com TCPDUMP e conversão direta para fluxos CSV via CICFlowMeter no próprio contêiner:
  * `collectPacketsCICFlowMeter(interfaceName, outputPath, rotateInterval=60)`
  * `convertPcapIntoFlows(pcapPath, destPath)`

## 4.5 Classe Controller (profissa_lft.controller.Controller)

Gerencia controladores SDN (Ryu ou ONOS).
  * `instantiate(dockerImage='alexandremitsurukaihara/lst2.0:ryucontroller')`
  * `initController(ip: str, port: int, command=[])`: Dispara o `ryu-manager` na porta 9001.

## 4.6 Classe CICFlowMeter (profissa_lft.cicflowmeter.CICFlowMeter)

Ferramenta para extração de atributos estatísticos de fluxos de rede para Machine Learning:
  * `instantiate(image='alexandremitsurukaihara/lst2.0:cicflowmeter')`
  * `convertPcapIntoFlows(pcapPath, destPath)`

## 4.7 Módulo Celular 4G/LTE (srsRAN)

  * **EPC** (`profissa_lft.epc.EPC`): `setEPCAddress()`, `setSgiInterfaceAddress()`, `addNewUE(name, ID, IP)`, `start()`, `stop()`.
  * **EnB** (`profissa_lft.enb.EnB`): `setEPCAddress()`, `setEnBAddress()`, `setMultiUEEnBAddr()`, `starGnuRadioMultiUE()`, `start()`, `stop()`.
  * **UE** (`profissa_lft.ue.UE`): `setUEID()`, `setTxGain()`, `setRxGain()`, `start()`, `stop()`.

## 4.8 Classe Perfsonar (profissa_lft.perfsonar.Perfsonar)

Permite configurar limites e exceções de rotas para medições de desempenho do pscheduler:
  * `readLimitFile(limitPath)`
  * `saveLimitFile(limitPath)`
  * `addRouteException(ip, netmask)`

----

# 5. EXEMPLOS PRÁTICOS

## 5.1 Topologia SDN Simples (2 Hosts + 1 Switch + 1 Controller)

```python
from profissa_lft.host import Host
from profissa_lft.switch import Switch
from profissa_lft.controller import Controller

h1 = Host('h1')
h2 = Host('h2')
s1 = Switch('s1')
c1 = Controller('c1')

h1.instantiate()
h2.instantiate()
s1.instantiate()
c1.instantiate()

h1.connect(s1, "h1s1", "s1h1")
h2.connect(s1, "h2s1", "s1h2")
c1.connect(s1, "c1s1", "s1c1")

h1.setIp('10.0.0.1', 24, "h1s1")
h2.setIp('10.0.0.2', 24, "h2s1")
s1.setIp('10.0.0.3', 24, 's1')
c1.setIp('10.0.0.4', 24, 'c1s1')

c1.initController('10.0.0.4', 9001)
s1.setController('10.0.0.4', 9001)

s1.connectToInternet('10.0.0.5', 24, "s1host", "hosts1")

h1.setDefaultGateway('10.0.0.5', "h1s1")
h2.setDefaultGateway('10.0.0.5', "h2s1")
s1.setDefaultGateway('10.0.0.5', "s1")
```

## 5.2 Emulação 4G/LTE com srsRAN e ZMQ

```python
from profissa_lft.epc import EPC
from profissa_lft.enb import EnB
from profissa_lft.ue import UE

epc = EPC('epc')
enb = EnB("enb")
ue1 = UE('ue1')

epc.instantiate()
enb.instantiate()
ue1.instantiate()

enb.connect(epc, "enbepc", "epcenb")
ue1.connect(enb, "ue1enb", "enbue1")

epc.setIp('10.0.0.1', 24, "epcenb")
enb.setIp('10.0.0.2', 24, "enbepc")
enb.setIp('11.0.0.1', 30, "enbue1")
ue1.setIp('11.0.0.2', 29, "ue1enb")

epc.setEPCAddress("10.0.0.1")
epc.addNewUE("ue1", "001010123456780", "172.16.0.2")

enb.setEPCAddress("10.0.0.1")
enb.setEnBAddress("10.0.0.2")
ue1.setUEID("001010123456780")

epc.start()
enb.start()
ue1.start()
```

----

# 6. CENÁRIO DE SEGURANÇA (UNBCA_ATTACK_ENVIRONMENT)

Localização: `scenario/UNBCA_ATTACK_ENVIRONMENT/`

Emula uma organização empresarial completa dividida em 4 subredes (Servidores, Gerenciamento, Escritório e Desenvolvimento) com duas bridges OpenFlow (`brint` interna e `brex` externa), simulando tráfego misto (benigno e malicioso) baseado no dataset **CIDDS**:

  * **Servidores**: Seafile, Mail, Web, Backup e Impressoras.
  * **Comportamentos**: Administração SSH, navegação empresarial, transferências Seafile, ataques de DoS (SYN Flood / HTTP Flood), Brute Force SSH e Port Scans.
  * **Execução**:
    * Modo Rápido (14 nós): `python3 cids.py`
    * Modo Completo (~30 nós + CICFlowMeter): `python3 cidds.py`
    * Limpeza: `./cleanup.sh`

----

# 7. RESOLUÇÃO DE PROBLEMAS (TROUBLESHOOTING)

## 7.1 Erro OVS: "database connection failed"
Aguarde a inicialização do daemon `ovsdb-server` antes de executar `ovs-vsctl add-br`. A versão mais recente do `profissa_lft/switch.py` implementa esse loop automaticamente.

## 7.2 Erro ip link: "Numerical result out of range"
O nome da interface virtual ultrapassou 15 caracteres (limite do Linux). Reduza o nome da interface para menos de 15 caracteres.

## 7.3 Limpeza Forçada de Topologias e Contêineres Travados
```bash
docker rm -f $(docker ps -aq)
sudo ip link del h_brint 2>/dev/null
sudo ip link del h_brex 2>/dev/null
sudo ip link del hosts1 2>/dev/null
sudo rm -rf /var/run/netns/*
sudo systemctl restart containerd docker
```
