# Resolução de Problemas (Troubleshooting)

Esta seção reúne diagnósticos e soluções para os erros mais frequentes encontrados ao executar simulações com o LFT.

----

## 1. Erro: "ovs-vsctl: database connection failed (No such file or directory)"

### Causa
O contêiner do switch foi disparado pelo Docker, mas o script executou o comando `ovs-vsctl add-br` antes que o daemon `ovsdb-server` estivesse completamente inicializado dentro do contêiner.

### Solução
Certifique-se de que a versão mais recente do `profissa_lft/switch.py` está em uso. O método `Switch.instantiate()` inclui um loop de espera ativa:
```python
for _ in range(30):
    res = subprocess.run(f"docker exec {self.getNodeName()} ovs-vsctl show", shell=True, capture_output=True)
    if res.returncode == 0:
        break
    time.sleep(0.5)
```

----

## 2. Erro: "Cannot find device / Numerical result out of range" no ip link

### Causa
O nome atribuído a uma das pontas do par `veth` excedeu o limite máximo do kernel Linux de **15 caracteres** (constante `IFNAMSIZ`).

### Solução
Reduza os nomes das interfaces. Em vez de `veth-switch-internal-mprinter`, utilize abreviações como `mpr_br` e `br_mpr`.

----

## 3. Erro: "Error response from daemon: cannot remove container: tried to kill container, but did not receive an exit event"

### Causa
O contêiner Docker foi finalizado abruptamente e os processos emulados ou sockets remanescentes mantiveram o shim do containerd ocupado.

### Solução
Execute a limpeza dos shims e reinicie o serviço Docker do host:
```bash
sudo pkill -9 containerd-shim
sudo rm -rf /var/run/netns/*
sudo systemctl restart containerd docker
docker rm -f $(docker ps -aq)
```

----

## 4. Contêineres ou Interfaces "Zumbis" Travados

Se um script Python for abortado com `SIGKILL` ou falha crítica de hardware, interfaces virtuais e contêineres podem permanecer alocados no host.

### Script de Limpeza Geral
```bash
# 1. Forçar a remoção de todos os contêineres:
docker rm -f $(docker ps -aq)

# 2. Remover interfaces de host do LFT:
sudo ip link del h_brint 2>/dev/null
sudo ip link del h_brex 2>/dev/null
sudo ip link del hosts1 2>/dev/null

# 3. Limpar links de namespaces em /var/run/netns:
sudo rm -rf /var/run/netns/*

# 4. Limpar regras órfãs de NAT no iptables (se necessário):
sudo iptables -t nat -F POSTROUTING
```

----

## 5. Comandos Úteis para Diagnóstico de Rede

| Comando | Utilidade |
| --- | --- |
| ''docker ps'' | Lista todos os contêineres e topologias ativas. |
| ''ip link show'' | Lista todas as interfaces do host e pares veth criados. |
| ''ls -la /var/run/netns/'' | Mostra quais contêineres possuem namespaces acessíveis pelo host. |
| ''ip -n <nome_do_no> addr show'' | Exibe as interfaces e IPs de dentro do contêiner sem entrar nele. |
| ''docker exec <switch> ovs-vsctl show'' | Exibe a bridge, portas ativas e o status da conexão com o controlador SDN. |
| ''docker exec <switch> ovs-ofctl dump-flows <switch>'' | Exibe as regras de encaminhamento OpenFlow instaladas na bridge. |
| ''docker exec <no> ping -c 3 <ip_destino>'' | Testa a conectividade entre dois nós da topologia. |
