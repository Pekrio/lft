# Cenário de Segurança e Ataques (UNBCA / CIDDS)

Localização no repositório: `scenario/UNBCA_ATTACK_ENVIRONMENT/`

O **UNBCA_ATTACK_ENVIRONMENT** é um ambiente avançado de emulação de redes e ataques cibernéticos, projetado com base na metodologia do dataset **CIDDS (Coburg Intrusion Detection Data Set)**. Ele replica o ambiente de uma organização corporativa completa com múltiplos departamentos, servidores de infraestrutura, impressoras e tráfego misto (benigno e malicioso).

----

## 1. Arquitetura da Rede Corporativa

A topologia divide a organização em subredes lógicas conectadas por duas bridges SDN programáveis:

```text
                     [ Rede Externa (Internet) ]
                                  |
                              (h_brex)
                                  |
                              [ brex ] <--- [ c2 ] (Ryu Controller)
                             /        \
                    (192.168.50.100)   \
                     /                  \
              [ eweb ]                   [ e1, e2 ]
           (Web Server)              (External Attackers)
                 |
                 +======================= (brex_brint <-> brint_brex)
                 |
             [ brint ] <--- [ c1 ] (Ryu Controller)
          (192.168.100.100)
                 |
    +------------+------------+---------------+----------------+
    |                         |               |                |
[ Servidores ]           [ Management ]    [ Office ]     [ Developer ]
192.168.100.0/24        192.168.200.0/24  192.168.210.0/24 192.168.220.0/24
- mail (100.1)           - mprinter (200.1)- oprinter(210.1)- dprinter(220.1)
- file (100.2)           - m1 (200.2)      - o1 (210.2)     - d1..d2 (Admins)
- web (100.3)            - m2 (200.3)      - o2 (210.3)     - d3..d11 (Devs)
- backup (100.4)         - m3 (200.4)                       - d12..d13 (Attackers)
- seafile (50.1)         - m4 (200.5)
```

----

## 2. Perfil dos Clientes e Automação de Comportamento

Cada contêiner cliente (`LinuxClient`) executa o script de automação `readIni.py` com base em perfis comportamentais definidos em arquivos INI (`client_behaviour/*.ini`):

| Perfil | Comportamento Emulado | Ações |
| --- | --- | --- |
| **Administrator** (''d1, d2'') | Administração de infraestrutura | Acesso SSH aos servidores, operações de backup, gerenciamento de arquivos. |
| **Management** (''m1..m4'') | Usuários de negócios | Envio de e-mails, download/upload no Seafile, navegação web empresarial, impressão de documentos. |
| **Office** (''o1, o2'') | Usuários de escritório | Consultas web rotineiras, e-mails com anexos, tarefas de impressão. |
| **Developer** (''d3..d11'') | Desenvolvedores de software | Conexões SSH, transferências contínuas de arquivos, testes em servidores internos. |
| **Internal Attacker** (''d12, d13'') | Ameaça interna (Insider Threat) | Escaneamento de portas internas (Port Scan), DoS (SYN Flood / HTTP Flood), força bruta SSH. |
| **External Attacker** (''e1, e2'') | Atacante externo vindo da Internet | Ataques direcionados contra o servidor web externo (''eweb'') e tentativas de invasão na rede interna. |

----

## 3. Execução dos Scripts do Cenário

### Modo Rápido / Simplificado (cids.py)
Instancia a infraestrutura central de servidores e a subrede de gerenciamento (14 contêineres). Ideal para validação rápida e depuração:

```bash
orb -m lft -u root bash -c "cd /Users/marotta/GDrive/Desenvolvimento/lft/scenario/UNBCA_ATTACK_ENVIRONMENT && python3 cids.py"
```

### Modo Completo com Geração de Dataset (cidds.py)
Instancia a topologia corporativa completa com todos os departamentos (~30 nós). Ao finalizar a captura com `CTRL+C`, o script aciona automaticamente o **CICFlowMeter** para processar os arquivos PCAP e gerar o arquivo `final_report.csv` contendo os fluxos rotulados:

```bash
orb -m lft -u root bash -c "cd /Users/marotta/GDrive/Desenvolvimento/lft/scenario/UNBCA_ATTACK_ENVIRONMENT && python3 cidds.py"
```

----

## 4. Script de Limpeza do Cenário

Para garantir que todos os contêineres, interfaces de rede e links de namespace sejam destruídos com segurança:

```bash
orb -m lft -u root bash -c "cd /Users/marotta/GDrive/Desenvolvimento/lft/scenario/UNBCA_ATTACK_ENVIRONMENT && ./cleanup.sh"
```
