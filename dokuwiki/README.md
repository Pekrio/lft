# Documentação DokuWiki do LFT

Este diretório contém a documentação completa do framework **LFT (Lightweight Fog Testbed)** no formato nativo de marcação **DokuWiki** (`.txt`).

## Estrutura dos Arquivos

```
dokuwiki/
├── sidebar.txt                 # Menu lateral de navegação da wiki
├── start.txt                   # Página inicial e índice geral
├── instalacao.txt              # Requisitos de sistema e procedimentos de instalação
├── arquitetura.txt             # Arquitetura (namespaces, veth, OVS, roteamento e NAT)
├── api_referencia.txt          # Referência completa de classes, métodos e parâmetros
├── sdn_topologias.txt          # Guia de construção de topologias SDN programáveis
├── exemplos_topologias.txt     # Exemplos práticos de código comentados (examples/)
├── srsran_4g.txt               # Emulação de redes celulares 4G/LTE com srsRAN
├── docker_imagens.txt          # Catálogo e especificação das imagens Docker
├── cenario_seguranca.txt       # Documentação do cenário de segurança UNBCA / CIDDS
├── experimentos_benchmarks.txt # Guias de experimentos, benchmarks e métricas
├── troubleshooting.txt         # Guia de diagnóstico e resolução de problemas
├── LFT_MANUAL_COMPLETO.txt     # Manual mestre compilado em uma única página
└── README.md                   # Este arquivo de instruções
```

---

## Como Integrar com uma Instalação DokuWiki

Em um servidor rodando o DokuWiki (ex.: intranet do laboratório COMNET):

1. Copie ou sincronize os arquivos `.txt` deste diretório para a pasta de páginas (`data/pages/`) do DokuWiki:
   ```bash
   # Exemplo: publicando no namespace "lft"
   mkdir -p /var/www/dokuwiki/data/pages/lft/
   cp dokuwiki/*.txt /var/www/dokuwiki/data/pages/lft/
   ```
2. Ajuste as permissões para o servidor web (ex.: `chown -R www-data:www-data /var/www/dokuwiki/data/pages/lft/`).
3. Acesse via navegador: `http://seu-servidor/doku.php?id=lft:start`.

---

## Como Integrar com o GitHub Wiki

Se desejar sincronizar com o GitHub Wiki do repositório:
1. Clone o repositório de wiki do projeto:
   ```bash
   git clone https://github.com/UnB-COMNET/lft.wiki.git
   ```
2. Os arquivos podem ser convertidos para Markdown (`.md`) ou sincronizados diretamente com scripts auxiliares de conversão (ex.: `pandoc -f dokuwiki -t gfm`).
