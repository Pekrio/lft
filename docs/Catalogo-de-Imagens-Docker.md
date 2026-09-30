# Catálogo de Imagens Docker do LFT

O LFT fornece receitas Dockerfile completas no diretório `docker/` para todos os tipos de nós suportados. Cada imagem foi desenvolvida com foco em baixo consumo de recursos, ferramentas essenciais de diagnóstico e scripts de boot (`onboot.sh`) que mantêm o contêiner ativo em topologias `--network=none`.

----

## 1. Resumo das Imagens

| Diretório | Imagem / Tag | Base | Serviços e Ferramentas Embutidas |
| --- | --- | --- | --- |
| ''docker/host'' | ''alexandremitsurukaihara/lst2.0:host'' | Ubuntu 20.04 | net-tools, iproute2, ping, iptables, nano, firewalld, tcpdump. |
| ''docker/openswitch'' | ''alexandremitsurukaihara/lst2.0:openvswitch'' | Ubuntu 20.04 | openvswitch-switch, tshark, tcpdump, firewalld, iptables. |
| ''docker/openswitchmeter'' | ''alexandremitsurukaihara/lft:openvswitchmeter'' | Ubuntu 20.04 | Open vSwitch + scripts automáticos de captura TCPDUMP e CICFlowMeter. |
| ''docker/controller'' | ''alexandremitsurukaihara/lst2.0:ryucontroller'' | Ubuntu 20.04 | Python 3, Ryu SDN Framework, eventlet 0.30.2, pandas. |
| ''docker/CICFlowMeter'' | ''alexandremitsurukaihara/lst2.0:cicflowmeter'' | Ubuntu 20.04 | Java runtime, suíte TCPDUMP_and_CICFlowMeter para conversão PCAP -> CSV. |
| ''docker/linux_client'' | ''alexandremitsurukaihara/lst2.0:linuxclient'' | Debian | Selenium, geckodriver, suíte de automação (navegação, e-mail, cópia, ataques DoS/SSH). |
| ''docker/web_server'' | ''alexandremitsurukaihara/lst2.0:web'' | Ubuntu 20.04 | Servidor Web Apache2 / Nginx com suporte a páginas estáticas e dinâmicas. |
| ''docker/mail_server'' | ''alexandremitsurukaihara/lst2.0:mail'' | Ubuntu 20.04 | Servidor de e-mails Postfix/Dovecot (SMTP porta 25, IMAP/POP3). |
| ''docker/file_server'' | ''alexandremitsurukaihara/lst2.0:file'' | Ubuntu 20.04 | Servidor de compartilhamento Samba / FTP. |
| ''docker/backup_server'' | ''alexandremitsurukaihara/lst2.0:backup'' | Ubuntu 20.04 | Servidor de armazenamento e backup remoto CIFS / Samba. |
| ''docker/print_server'' | ''alexandremitsurukaihara/lst2.0:printer'' | Ubuntu 20.04 | Servidor de impressão CUPS virtual com suporte a filas IPP/Raw. |
| ''docker/seafile_server'' | ''alexandremitsurukaihara/lst2.0:seafile'' | Ubuntu 20.04 | Servidor de nuvem privada Seafile Server com sincronização HTTP. |
| ''docker/srsRAN'' | ''alexandremitsurukaihara/lft:srsran'' | Ubuntu 20.04 | Binários compilados do srsEPC, srsENB, srsUE, ZeroMQ e biblioteca UHD. |
| ''docker/gnuradio'' | ''alexandremitsurukaihara/lft:gnuradio'' | Ubuntu 20.04 | GNU Radio 3.8+ e gr-zeromq para modelagem de canal sem fio físico. |
| ''docker/perfsonar'' | ''alexandremitsurukaihara/lft:perfsonar'' | CentOS / Rocky | PerfSONAR Testpoint completo com pscheduler para medições de QoS. |
| ''docker/perfsonar_srsRAN'' | ''alexandremitsurukaihara/lft:perfsonar_srsran'' | Ubuntu 20.04 | Imagem híbrida combinando srsRAN com agentes de teste perfSONAR. |

----

## 2. Estrutura Padrão dos Dockerfiles

Cada contêiner do LFT segue o padrão estrutural:

```docker
FROM ubuntu:20.04

# 1. Instalação não interativa de utilitários de rede essenciais
RUN apt update \
&& RUNLEVEL=1 apt install -y --no-install-recommends \
   sudo net-tools iproute2 iputils-ping iptables nano firewalld tcpdump

# 2. Inclusão e permissão de execução do script de inicialização
COPY onboot.sh /home
RUN chmod +x /home/onboot.sh

# 3. Ponto de entrada padrão
CMD ["/home/onboot.sh"]

# 4. Portas expostas
EXPOSE 22
```

----

## 3. Como Construir Localmente uma Imagem

Se você alterar qualquer Dockerfile ou script dentro de `docker/<nome_do_servico>`:

```bash
cd docker/<nome_do_servico>
docker build -t alexandremitsurukaihara/lst2.0:<tag_desejada> .
```
