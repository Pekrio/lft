# Instalação e Requisitos do LFT

## 1. Requisitos do Sistema Operacional

O LFT interage diretamente com chamadas e subsistemas do **kernel Linux** (interfaces virtuais `veth`, manipulação de namespaces de rede em `/var/run/netns`, regras de firewall `iptables`/`firewalld`, e daemon `openvswitch-switch`).

| Plataforma | Suporte | Instruções |
| --- | --- | --- |
| **Ubuntu Desktop 24.04 LTS** | Nativo (Recomendado) | Instalação direta no host |
| **macOS (Apple Silicon / Intel)** | Via Máquina Linux (OrbStack / UTM) | Máquina virtual Ubuntu gerenciada via OrbStack |
| **Windows** | Via WSL 2 com suporte a systemd | Instalação na distribuição Ubuntu dentro do WSL 2 |

> **Nota para usuários macOS**: O kernel Darwin/XNU do macOS não possui namespaces de rede Linux nem interfaces virtuais `veth`. Portanto, o LFT não roda de forma "bare-metal" no macOS. A execução é realizada de maneira transparente usando o **OrbStack** para criar uma máquina Linux Ubuntu 24.04 integrada.

----

## 2. Dependências do Sistema (Linux)

Os seguintes pacotes de sistema são obrigatórios para a execução do framework:

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
```

Certifique-se de que os serviços essenciais estão ativos:

```bash
sudo systemctl enable --now docker
sudo systemctl enable --now openvswitch-switch
sudo systemctl enable --now firewalld
sudo usermod -aG docker $USER
```

----

## 3. Instalação do Pacote LFT (profissa_lft)

### Opção A: Instalação via repositório local (Modo Editável / Desenvolvimento)

No diretório raiz do repositório:

```bash
# No Ubuntu 24.04 (com PEP 668):
pip3 install --break-system-packages -e .

# Ou utilizando um ambiente virtual (venv):
python3 -m venv venv
source venv/bin/activate
pip install -e .
```

### Opção B: Instalação via script de dependências

```bash
chmod +x dependencies.sh
sudo ./dependencies.sh
```

----

## 4. Configuração no macOS via OrbStack

Se você estiver em um Mac:

  - Certifique-se de que o OrbStack está instalado:
```bash
brew install --cask orbstack
```
  - Crie uma máquina Ubuntu 24.04:
```bash
orbctl create ubuntu:24.04 lft
```
  - Acesse o terminal da máquina como root:
```bash
orb -m lft -u root
```
  - Dentro da máquina, os mesmos arquivos do seu Mac estarão montados em `/Users/<seu_usuario>/...`. Basta navegar até o diretório do projeto e instalar as dependências.

----

## 5. Catálogo de Imagens Docker Utilizadas

O LFT utiliza contêineres Docker leves para representar nós, switches, servidores e aplicações. As principais imagens hospedadas no Docker Hub são:

| Imagem Docker | Finalidade |
| --- | --- |
| ''alexandremitsurukaihara/lst2.0:host'' | Host Linux genérico com utilitários de rede (iproute2, ping, tcpdump) |
| ''alexandremitsurukaihara/lst2.0:openvswitch'' | Switch virtual baseado em Open vSwitch 2.13 com suporte a OpenFlow |
| ''alexandremitsurukaihara/lst2.0:ryucontroller'' | Controlador SDN baseado no framework Ryu |
| ''alexandremitsurukaihara/lst2.0:linuxclient'' | Cliente Debian com suíte de automação de tráfego e scripts de ataque |
| ''alexandremitsurukaihara/lst2.0:web'' | Servidor Web HTTP/HTTPS |
| ''alexandremitsurukaihara/lst2.0:mail'' | Servidor de Correio Eletrônico (SMTP/IMAP) |
| ''alexandremitsurukaihara/lst2.0:file'' | Servidor de Arquivos (Samba / FTP) |
| ''alexandremitsurukaihara/lst2.0:backup'' | Servidor de Backup em rede (CIFS / NFS) |
| ''alexandremitsurukaihara/lst2.0:printer'' | Emulador de Impressora de rede (CUPS) |
| ''alexandremitsurukaihara/lst2.0:seafile'' | Servidor de Nuvem Privada Seafile |
| ''alexandremitsurukaihara/lst2.0:cicflowmeter'' | Ferramenta de análise de pacotes PCAP e extração de características de fluxos |
| ''alexandremitsurukaihara/lft:srsran'' | Suíte completa srsRAN (srsEPC, srsENB, srsUE) com suporte ZMQ |
