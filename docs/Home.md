<p align="center"><img src="../logos/lft-github.png" width="450" alt="LFT Logo"></p>

# Lightweight Fog Testbed (LFT)

O **Lightweight Fog Testbed (LFT)** (evolução do *LST 2.0*) é um framework baseado em Python e contêineres Docker desenvolvido para simplificar a criação, emulação e teste de topologias de rede heterogêneas e leves para ambientes de **Fog Computing**, **Edge Computing** e **Segurança de Redes**.

Com o LFT, é possível orquestrar nós arbitrários em contêineres Docker, conectar dispositivos via pares virtuais Ethernet (`veth`) em *network namespaces* dedicados do Linux, gerenciar comutação programável via **Open vSwitch (OVS)**, controladores **SDN (ex.: Ryu, ONOS)**, emular enlaces sem fio **4G/LTE** utilizando **srsRAN** e coletar tráfego para análise com **CICFlowMeter** ou métricas de desempenho com **PerfSONAR**.

Este wiki contém toda a documentação necessária para entender, instalar, operar e estender o LFT.

----

## Índice da Documentação

  * [1. Instalação e Requisitos](Instalacao.md)
    * Requisitos de sistema (Linux Ubuntu 24.04 LTS e macOS com OrbStack)
    * Instalação via `pip` e script de dependências
    * Imagens Docker utilizadas
  * [2. Arquitetura do LFT](Arquitetura.md)
    * Isolamento por Linux Network Namespaces
    * Pares veth e interconexão de contêineres
    * Integração com Open vSwitch e controladores SDN
    * Roteamento, NAT e conectividade externa
  * [3. Referência Completa da API (profissa_lft)](API-Referencia.md)
    * Classe base `Node` (métodos de ciclo de vida, rede, tráfego e arquivos)
    * Classe `Host`
    * Classe `Switch` e `SwitchMeter`
    * Classe `Controller`
    * Classe `CICFlowMeter`
    * Classes para 4G/LTE: `EPC`, `EnB`, `UE`
    * Classe `Perfsonar`
    * Constantes e Exceções
  * [4. Criação de Topologias SDN](Topologias-SDN.md)
    * Exemplo de topologia simples (Hosts + Switch + Controller)
    * Configuração de controle de tráfego (TC netem - delay, jitter, rate)
    * Conexão com a Internet e gateways
  * [5. Exemplos Práticos de Código (examples/)](Exemplos-de-Codigo.md)
    * Scripts comentados de ponta a ponta
    * Topologia direta, SDN, multi-subrede, multi-controlador, NetFlow
  * [6. Emulação Sem Fio 4G/LTE (srsRAN)](Emulacao-4G-LTE.md)
    * Configuração do EPC (MME, SPGW, banco de usuários)
    * Configuração da eNodeB (ZMQ / GNU Radio)
    * Conexão de UEs e alocação de IPs
  * [7. Catálogo de Imagens Docker](Catalogo-de-Imagens-Docker.md)
    * Especificação de cada contêiner, portas, serviços e propósito
  * [8. Cenário de Segurança e Ataques (UNBCA / CIDDS)](Cenario-Seguranca-UNBCA.md)
    * Topologia corporativa multi-subrede
    * Comportamentos de clientes benignos e atacantes
    * Captura de pacotes e conversão para datasets de intrusão
  * [9. Experimentos e Benchmarks](Experimentos-e-Benchmarks.md)
    * Procedimentos experimentais, métricas e análise de desempenho
  * [10. Resolução de Problemas](Resolucao-de-Problemas.md)
    * Dúvidas comuns, diagnóstico de conectividade e comandos de limpeza
  * [11. Manual Completo (Tudo em 1 Página)](Manual-Completo.md)
    * Compilado unificado de toda a documentação para consulta rápida ou impressão

----

## Informações do Projeto

| Atributo | Detalhes |
| --- | --- |
| **Nome do Pacote** | ''profissa_lft'' |
| **Versão Atual** | 1.0.9 |
| **Linguagem** | Python >= 3.9 |
| **Licença** | GNU General Public License v3 (GPLv3) |
| **Instituição** | Universidade de Brasília (UnB) - Laboratório COMNET |
| **Repositório** | [[https://github.com/UnB-COMNET/lft | github.com/UnB-COMNET/lft]] |
| **Autor Original** | Alexandre Mitsuru Kaihara |
