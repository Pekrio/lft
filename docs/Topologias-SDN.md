# Criação de Topologias SDN no LFT

Este guia prático ensina a construir topologias SDN programáveis utilizando os componentes do LFT.

----

## 1. Exemplo de Topologia Básica

Vamos analisar o exemplo `examples/simpleSDNTopology.py`, que interliga dois hosts (`h1` e `h2`) a um switch OpenFlow (`s1`) gerenciado por um controlador Ryu (`c1`), com saída para a Internet.

### Diagrama da Topologia

```text
       [ Internet / Host Gateway (10.0.0.5) ]
                         |
                   (hosts1 / s1host)
                         |
                      [ s1 ] (10.0.0.3)
                     /  |  \
         (s1h1)     /   |   \  (s1h2)
           (h1s1)  /    |    \  (h2s1)
                  /     |     \
               [ h1 ]   |    [ h2 ]
            10.0.0.1    |   10.0.0.2
                        |
                     (s1c1)
                     (c1s1)
                        |
                     [ c1 ] (Controller - 10.0.0.4:9001)
```

### Código Python Completo

```python
from profissa_lft.host import Host
from profissa_lft.switch import Switch
from profissa_lft.controller import Controller

# 1. Definir os nós
h1 = Host('h1')
h2 = Host('h2')
s1 = Switch('s1')
c1 = Controller('c1')

# 2. Instanciar os contêineres Docker
h1.instantiate()
h2.instantiate()
s1.instantiate()
c1.instantiate()

# 3. Interconectar nós criando pares veth
# Parâmetros: connect(node_destino, iface_local, iface_remota)
h1.connect(s1, "h1s1", "s1h1")
h2.connect(s1, "h2s1", "s1h2")
c1.connect(s1, "c1s1", "s1c1")

# 4. Atribuir endereçamento IP às interfaces
h1.setIp('10.0.0.1', 24, "h1s1")
h2.setIp('10.0.0.2', 24, "h2s1")
s1.setIp('10.0.0.3', 24, 's1')
c1.setIp('10.0.0.4', 24, 'c1s1')

# 5. Inicializar o Ryu Controller e conectar o Switch
c1.initController('10.0.0.4', 9001)
s1.setController('10.0.0.4', 9001)

# 6. Conectar o Switch à Internet através do Host
s1.connectToInternet('10.0.0.5', 24, "s1host", "hosts1")

# 7. Configurar Default Gateway em todos os nós
h1.setDefaultGateway('10.0.0.5', "h1s1")
h2.setDefaultGateway('10.0.0.5', "h2s1")
s1.setDefaultGateway('10.0.0.5', "s1")

print("Topologia SDN pronta!")
```

----

## 2. Regras Essenciais ao Criar Topologias

  - **Nomenclatura de Interfaces Virtuais (veth)**:
    No Linux, nomes de interface **não podem exceder 15 caracteres**. Adote padrões concisos:
    * `<origem>_<destino>` (ex.: `h1_s1` e `s1_h1`)
    * `<no>_h` para a ponta do switch e `h_<no>` para a ponta do host
  - **Ordem de Operações**:
    1. Executar sempre `instantiate()` antes de `connect()`.
    2. Executar `connect()` antes de `setIp()`.
    3. Executar `initController()` antes de `setController()`.
    4. Configurar gateways e rotas por último.

----

## 3. Aplicando Modelagem de Tráfego (Emulação de Enlace)

Para emular links realistas com latência, instabilidade (jitter) e restrição de taxa (throughput), use `setInterfaceProperties`:

```python
# Aplicar 50ms de latência, 5ms de jitter e limite de 10 Mbps na interface de h1
h1.setInterfaceProperties(
    interfaceName="h1s1",
    throughput="10mbit",
    delay="50ms",
    jitter="5ms"
)
```

Isso programa o agendador `tc netem` no kernel Linux dentro do namespace de `h1`.

----

## 4. Testes e Validação da Topologia

Com a topologia ativa, valide a comunicação diretamente via terminal:

```bash
# Ping direto entre hosts:
docker exec h1 ping -c 3 10.0.0.2

# Ping para o gateway do switch:
docker exec h1 ping -c 3 10.0.0.5

# Teste de acesso à Internet (DNS público):
docker exec h1 ping -c 3 8.8.8.8

# Visualizar o status das portas Open vSwitch na bridge:
docker exec s1 ovs-vsctl show
docker exec s1 ovs-ofctl dump-flows s1
```
