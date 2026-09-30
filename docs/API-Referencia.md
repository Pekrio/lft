# Referência Completa da API (profissa_lft)

O pacote `profissa_lft` é o núcleo da biblioteca LFT. Todas as classes derivam direta ou indiretamente da classe base `Node`.

----

## 1. Classe Base: Node

Localização: `profissa_lft.node.Node`

Superclasse que define as operações fundamentais de gerenciamento de contêineres, interfaces virtuais, roteamento e execução de comandos.

### Construtor
```python
Node(nodeName: str)
```
  * **nodeName** (str): Nome identificador único do nó e do contêiner Docker.

### Métodos de Ciclo de Vida do Contêiner

| Método | Parâmetros | Descrição |
| --- | --- | --- |
| ''instantiate(...)'' | ''dockerImage="alexandremitsurukaihara/lst2.0:host"'', ''dockerCommand='' '', ''dns='8.8.8.8''', ''memory='' '', ''cpus='' '', ''runCommand='' '' | Cria e executa o contêiner com ''--network=none'' e privilégios. Vincula o namespace de rede em ''/var/run/netns/<nodeName>''. |
| ''delete()'' | Nenhum | Força a parada e remoção do contêiner Docker (''docker kill && docker rm''). |
| ''getNodeName()'' | Nenhum | Retorna a string com o nome do nó. |

### Métodos de Conexão e Rede

| Método | Parâmetros | Descrição |
| --- | --- | --- |
| ''connect(node, interfaceName, peerInterfaceName)'' | ''node: Node'', ''interfaceName: str'', ''peerInterfaceName: str'' | Cria um par ''veth'' interligando este nó ao nó de destino. Se algum dos nós for um ''Switch'', a interface correspondente é registrada como porta OVS. |
| ''setIp(ip, mask, interfaceName)'' | ''ip: str'', ''mask: int'', ''interfaceName: str'' | Atribui um endereço IP e máscara (ex.: ''setIp("10.0.0.1", 24, "h1s1")'') à interface indicada do nó. |
| ''setDefaultGateway(destinationIp, interfaceName)'' | ''destinationIp: str'', ''interfaceName: str'' | Adiciona rota para o IP de destino e configura-o como rota padrão (default gateway) do contêiner. |
| ''addRoute(ip, mask, interfaceName)'' | ''ip: str'', ''mask: int'', ''interfaceName: str'' | Adiciona uma rota estática na tabela de roteamento interna do contêiner. |
| ''addRouteOnHost(ip, mask, interfaceName, gateway="0.0.0.0")'' | ''ip: str'', ''mask: int'', ''interfaceName: str'', ''gateway: str'' | Adiciona uma rota estática na tabela de roteamento do **host**. |
| ''connectToInternet(hostIP, hostMask, interfaceName, hostInterfaceName)'' | ''hostIP: str'', ''hostMask: int'', ''interfaceName: str'', ''hostInterfaceName: str'' | Cria um par ''veth'' entre o nó (ou switch) e o host, configurando NAT MASQUERADE e zona de confiança no firewall. |
| ''connectToInternetWithoutNAT(hostIP, hostMask, interfaceName, hostInterfaceName)'' | ''hostIP: str'', ''hostMask: int'', ''interfaceName: str'', ''hostInterfaceName: str'' | Conecta a interface ao host sem criar regras de NAT/MASQUERADE. |
| ''enableForwarding(interfaceName, otherInterfaceName)'' | ''interfaceName: str'', ''otherInterfaceName: str'' | Habilita regras de forwarding e NAT entre duas interfaces do nó. |
| ''setMtuSize(interfaceName, mtu)'' | ''interfaceName: str'', ''mtu: int'' | Ajusta o tamanho da MTU na interface indicada. |
| ''acceptPacketsFromInterface(interfaceName)'' | ''interfaceName: str'' | Adiciona regra no ''iptables'' interno para aceitar todo tráfego de entrada na interface. |

### Modelagem de Tráfego (QoS / Traffic Control)

```python
setInterfaceProperties(interfaceName: str, throughput: str, delay: str, jitter: str) -> None
```
Aplica modelagem via Linux Traffic Control (`tc netem`) na interface indicada.
  * **interfaceName**: Nome da interface do nó.
  * **throughput**: Taxa de transmissão (ex.: `"10mbit"`, `"100kbit"`).
  * **delay**: Atraso médio introduzido (ex.: `"20ms"`).
  * **jitter**: Variação aleatória do atraso (ex.: `"5ms"`).

### Execução de Comandos e Manipulação de Arquivos

| Método | Retorno | Descrição |
| --- | --- | --- |
| ''run(command: str)'' | ''subprocess.Popen'' | Executa comando bash dentro do contêiner via ''docker exec''. |
| ''runs(commands: list)'' | ''list[subprocess.Popen]'' | Executa múltiplos comandos dentro do contêiner. |
| ''copyLocalToContainer(path, destPath)'' | None | Copia arquivo do sistema de arquivos local/host para dentro do contêiner. |
| ''copyContainerToLocal(path, destPath)'' | None | Copia arquivo de dentro do contêiner para o sistema local/host. |
| ''readConfigFile(containerPath)'' | ''ConfigParser'' | Lê arquivo de configuração INI de dentro do contêiner e retorna objeto ''ConfigParser''. |
| ''saveConfig(config, containerPath)'' | None | Salva objeto ''ConfigParser'' no contêiner. |
| ''setHost(ip: str)'' | None | Atualiza o ''/etc/hosts'' interno com a resolução do hostname atual para o IP informado. |

----

## 2. Classe: Host

Localização: `profissa_lft.host.Host` (Herda de `Node`)

Representa um host ou servidor final na rede.

```python
from profissa_lft.host import Host

h1 = Host("h1")
h1.instantiate(dockerImage="alexandremitsurukaihara/lst2.0:host")
```

----

## 3. Classe: Switch

Localização: `profissa_lft.switch.Switch` (Herda de `Node`)

Representa um switch comutador Open vSwitch (OVS).

### Construtor
```python
Switch(name: str, hostPath='', containerPath='')
```
  * Permite mapear volumes locais para captura ou persistência de logs/PCAPs.

### Métodos Específicos

| Método | Parâmetros | Descrição |
| --- | --- | --- |
| ''instantiate(image='alexandremitsurukaihara/lst2.0:openvswitch', controllerIP='', controllerPort=-1)'' | ''image: str'', ''controllerIP: str'', ''controllerPort: int'' | Inicializa o contêiner OVS, cria a bridge homônima e conecta opcionalmente ao controlador. |
| ''setController(ip: str, port: int)'' | ''ip: str'', ''port: int'' | Associa a bridge do switch ao controlador OpenFlow via TCP (''ovs-vsctl set-controller''). |
| ''enableNetflow(bridgeName, destIp, destPort, activeTimeout=60)'' | ''bridgeName: str'', ''destIp: str'', ''destPort: int'', ''activeTimeout: int'' | Ativa a exportação de fluxos NetFlow para um coletor externo. |
| ''disableNetflow(bridgeName: str)'' | ''bridgeName: str'' | Desativa a exportação de fluxos NetFlow. |
| ''setFlow(rule: str)'' | ''rule: str'' | Adiciona uma regra de fluxo estática OpenFlow na bridge (''ovs-ofctl add-flow''). |
| ''collectPackets(interfaceNames=[], path='', rotateInterval=60)'' | ''interfaceNames: list'', ''path: str'', ''rotateInterval: int'' | Inicia o ''tshark'' em background capturando o tráfego nas interfaces do switch. O parâmetro ''path'' deve terminar em ''.pcap''. |

----

## 4. Classe: Controller

Localização: `profissa_lft.controller.Controller` (Herda de `Node`)

Gerencia um controlador SDN baseado no framework Ryu ou personalizado.

| Método | Parâmetros | Descrição |
| --- | --- | --- |
| ''instantiate(dockerImage='alexandremitsurukaihara/lst2.0:ryucontroller', dockerCommand='')'' | ''dockerImage: str'', ''dockerCommand: str'' | Inicializa o contêiner do controlador. |
| ''initController(ip: str, port: int, command=[])'' | ''ip: str'', ''port: int'', ''command: list'' | Inicia o daemon ''ryu-manager'' escutando no IP e porta especificados (padrão: 9001). |

----

## 5. Classe: CICFlowMeter

Localização: `profissa_lft.cicflowmeter.CICFlowMeter` (Herda de `Node`)

Orquestra a ferramenta CICFlowMeter para processamento de arquivos PCAP e extração de métricas estatísticas de fluxos de rede (como tamanho de pacotes, IAT, flags, bytes por segundo), muito utilizada para datasets de segurança e aprendizado de máquina.

| Método | Parâmetros | Descrição |
| --- | --- | --- |
| ''instantiate(image='alexandremitsurukaihara/lst2.0:cicflowmeter')'' | ''image: str'' | Inicializa o contêiner CICFlowMeter com montagem do diretório de fluxos. |
| ''convertPcapIntoFlows(pcapPath: str, destPath: str)'' | ''pcapPath: str'', ''destPath: str'' | Converte o arquivo PCAP indicado gerando o arquivo CSV de características no diretório de destino. |

----

## 6. Módulo Móvel 4G/LTE: EPC, EnB e UE

Localizações:
  * `profissa_lft.epc.EPC`
  * `profissa_lft.enb.EnB`
  * `profissa_lft.ue.UE`

Utilizam a suíte **srsRAN 4G** com camada RF emulada via sockets ZeroMQ (ZMQ) ou GNU Radio.

### EPC (Evolved Packet Core)
  * `instantiate(dockerImage='alexandremitsurukaihara/lft:srsran', runCommand=`)''
  * `setEPCAddress(ip='127.0.1.100')`: Define IP de bind do MME e SPGW.
  * `setSgiInterfaceAddress(ip='172.16.0.1')`: Define IP da interface SGi (saída para internet).
  * `addNewUE(name, ID, IP="dynamic")`: Adiciona um dispositivo móvel (IMSI) no banco de usuários (`user_db.csv`).
  * `start()`: Dispara o processo `srsepc`.
  * `stop()`: Encerra o processo `srsepc`.

### EnB (eNodeB / Estação Rádio-Base)
  * `instantiate(dockerImage='alexandremitsurukaihara/lft:srsran', ...)`
  * `start(transmitterIp="*", transmitterPort=2000, receiverIp="localhost", receiverPort=2001)`: Inicia a estação base com portas de rádio ZMQ.
  * `stop()`: Encerra o processo `srsenb`.
  * `starGnuRadioSingleUE()` / `starGnuRadioMultiUE()`: Executa emulação via canal GNU Radio.

### UE (User Equipment / Terminal Móvel)
  * `instantiate(dockerImage='alexandremitsurukaihara/lft:srsran', ...)`
  * `start(deviceArgs=`)`: Inicia o terminal móvel `srsue'' conectando ao ZMQ da eNodeB.
  * `stop()`: Encerra o processo `srsue`.
  * `setTxGain(txGain: int)`, `setRxGain(rxGain: int)`, `setIMSI(imsi: str)`: Configura parâmetros de rádio e identidade do chip SIM.

----

## 7. Exceções

Localização: `profissa_lft.exceptions`

  * `NodeInstantiationFailed`: Lançada quando a criação ou inicialização de um nó/contêiner falha.
  * `NodeConnectInterfaceFailed`: Lançada em caso de erro na criação de interfaces virtuais ou adição de portas no switch.
