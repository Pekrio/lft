# Topologias e Exemplos de Código no LFT

O diretório `examples/` do LFT reúne cenários que demonstram cada recurso da biblioteca. Abaixo estão detalhados os scripts de exemplo, o que cada linha faz e como executá-los.

----

## 1. Topologia Host-a-Host Direta (SimpleTopologyExample.py)

Demonstra a menor topologia possível no LFT: dois hosts conectados diretamente por um par de interfaces virtuais `veth`, sem a presença de switches ou controladores.

### Código Comentado
```python
from profissa_lft.host import Host

# 1. Instancia dois hosts
h1 = Host('h1')
h2 = Host('h2')

h1.instantiate()
h2.instantiate()

# 2. Conecta diretamente h1 e h2 via par veth (h1h2 <-> h2h1)
h1.connect(h2, "h1h2", "h2h1")

# 3. Atribui endereços IP na mesma sub-rede /24
h1.setIp('10.0.0.1', 24, "h1h2")
h2.setIp('10.0.0.2', 24, "h2h1")

# 4. Testa a conectividade com ping direto
h1.run("ping -c 3 10.0.0.2")
```

### Como Executar
```bash
python3 examples/SimpleTopologyExample.py
```

----

## 2. Topologia SDN com Controlador Ryu (simpleSDNTopology.py)

Conecta dois hosts a um switch Open vSwitch, gerenciado remotamente por um controlador Ryu via OpenFlow (porta 9001), com rota padrão para a Internet via gateway no host.

### Código Comentado
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

# Interconexões: cada host tem uma ponta veth conectada a s1
h1.connect(s1, "h1s1", "s1h1")
h2.connect(s1, "h2s1", "s1h2")
c1.connect(s1, "c1s1", "s1c1")

h1.setIp('10.0.0.1', 24, "h1s1")
h2.setIp('10.0.0.2', 24, "h2s1")
s1.setIp('10.0.0.3', 24, 's1')
c1.setIp('10.0.0.4', 24, 'c1s1')

# Inicializa o ryu-manager e conecta o switch
c1.initController('10.0.0.4', 9001)
s1.setController('10.0.0.4', 9001)

# Conecta o switch ao gateway do host físico (10.0.0.5) com NAT
s1.connectToInternet('10.0.0.5', 24, "s1host", "hosts1")

# Define 10.0.0.5 como rota padrão para saída externa
h1.setDefaultGateway('10.0.0.5', "h1s1")
h2.setDefaultGateway('10.0.0.5', "h2s1")
s1.setDefaultGateway('10.0.0.5', "s1")
```

----

## 3. Duas Subredes com Roteamento SDN (TwoSubNetSDNExample.py)

Interconecta dois switches (`s1` na rede 10.0.0.0/24 e `s2` na rede 11.0.0.0/24) sob o controle de um único controlador Ryu compartilhado na rede 12.0.0.0/24.

### Destaques do Código
  * **Interconexão entre switches**: `s1.connect(s2, "s1s2", "s2s1")` cria um enlace tronco entre as duas bridges OVS.
  * **Múltiplos IPs na bridge**: O switch recebe múltiplos IPs (um para falar com o controlador e outro para sua subrede).
  * **Rotas inter-subredes**:
    <code python>
    h1.addRoute("11.0.0.0", 24, "h1s1") # h1 alcança a subrede de h2
    h2.addRoute("10.0.0.0", 24, "h2s2") # h2 alcança a subrede de h1
```

----

## 4. Topologia com Múltiplos Controladores (MultiControllerTwoSubNetExample.py)

Demonstra controle distribuído onde cada switch SDN é subordinado a seu próprio controlador dedicado:
  * Switch `s1` gerenciado pelo controlador `c1` (12.0.0.1:9001).
  * Switch `s2` gerenciado pelo controlador `c2` (13.0.0.1:9001).
  * Os switches comunicam-se entre si através do enlace `s1s2 <-> s2s1`.

----

## 5. Captura de Pacotes PCAP no Switch (collectPacketsSDNTopology.py)

Inicia o sniffer de pacotes `tshark` em segundo plano dentro do contêiner do switch e copia os arquivos PCAP gerados para a máquina hospedeira.

### Código Comentado
```python
filePath = "/home/packets"
fileName = "test.pcap"

# 1. Cria a pasta interna no contêiner
s1.run(f"mkdir {filePath}")

# 2. Inicia a captura em rotação a cada 10 segundos
s1.collectPackets(["s1"], f"{filePath}/{fileName}", rotateInterval=10)

# 3. Gera tráfego de teste
h1.run("ping -c 5 10.0.0.2")
h2.run("ping -c 5 10.0.0.1")
sleep(12)

# 4. Copia os arquivos PCAP capturados para o diretório local do host
s1.copyContainerToLocal(filePath, "./")
```

----

## 6. SwitchMeter com CICFlowMeter Integrado (swtichmeterCollectPackets.py)

Utiliza a classe `SwitchMeter`, que combina um switch Open vSwitch com ferramentas embutidas de captura e conversão direta para fluxos CSV via CICFlowMeter:

```python
from profissa_lft.switchmeter import SwitchMeter

s1 = SwitchMeter('s1')
s1.instantiate()

# Captura diretamente usando TCPDUMP e CICFlowMeter embutidos no contêiner
s1.collectPacketsCICFlowMeter(interfaceName="s1", outputPath="/home/packets", rotateInterval=10)
```

----

## 7. Exportação e Coleta de NetFlow (netflowEnablementExample.py)

Configura o Open vSwitch para gerar e exportar registros **NetFlow v5** para um coletor `nfcapd` (parte da suíte `nfdump`) rodando no host:

### Código Comentado
```python
# 1. Inicia o coletor nfcapd no host na porta UDP 2055
subprocess.run("mkdir -p netflow && nfcapd -w -D -l ./netflow -p 2055", shell=True)

# 2. Habilita o envio de NetFlow no switch s1 apontando para o IP do gateway do host
s1.enableNetflow("s1", "10.0.0.5", "2055")

# 3. Gera tráfego de rede para criar fluxos
h1.run("ping -c 10 10.0.0.2")
h1.run("apt update")
sleep(20)

# 4. Analisa os fluxos capturados com o nfdump:
returnedExecution = subprocess.run(
    "nfdump -r netflow/nfcapd.* -s ip/bytes -n 10", 
    shell=True, stdout=subprocess.PIPE, text=True
)
print(returnedExecution.stdout)
```

----

## 8. Topologia Celular 4G com Múltiplos UEs (simple4GTopology.py)

Demonstra uma rede 4G completa com 1 EPC (núcleo), 1 eNodeB (estação base) e 2 UEs (terminais móveis), comunicando-se via canais ZMQ e GNU Radio:

```python
from profissa_lft.epc import EPC
from profissa_lft.enb import EnB
from profissa_lft.ue import UE

epc = EPC('epc')
enb = EnB("enb")
ue1 = UE('ue1')
ue2 = UE('ue2')

epc.instantiate()
enb.instantiate()
ue1.instantiate()
ue2.instantiate()

enb.connect(epc, "enbepc", "epcenb")
ue1.connect(enb, "ue1enb", "enbue1")
ue2.connect(enb, "ue2enb", "enbue2")

# Atribuição de IPs nas interfaces S1 e rádio
epc.setIp('10.0.0.1', 24, "epcenb")
enb.setIp('10.0.0.2', 24, "enbepc")
enb.setIp('11.0.0.1', 30, "enbue1")
enb.setIp('11.0.0.5', 30, "enbue2")
ue1.setIp('11.0.0.2', 29, "ue1enb")
ue2.setIp('11.0.0.6', 29, "ue2enb")

# Cadastro dos UEs no banco do EPC
epc.setEPCAddress("10.0.0.1")
epc.addNewUE(ue1.getNodeName(), "001010123456780", "172.16.0.2")
epc.addNewUE(ue2.getNodeName(), "001010123456789", "172.16.0.3")

# Configuração dos endereços RF da eNodeB e UEs
enb.setEPCAddress("10.0.0.1")
enb.setEnBAddress("10.0.0.2")
enb.setMultiUEEnBAddr("11.0.0.1", 2101, '11.0.0.1', 2100)
enb.setMultiUEUE1Addr("11.0.0.2", 2001, "11.0.0.1", 2000)
enb.setMultiUEUE2Addr("11.0.0.6", 2011, "11.0.0.5", 2010)

ue1.setUEID("001010123456780")
ue2.setUEID("001010123456789")

# Disparo dos daemons
epc.start()
enb.starGnuRadioMultiUE()
enb.start("11.0.0.1", 2101, "11.0.0.1", 2100)
ue1.start("11.0.0.2", 2001, "11.0.0.1", 2000)
ue2.start("11.0.0.6", 2011, "11.0.0.5", 2010)
```
