# Atividade 3.4 – Monitoramento e Análise de Tráfego

## Objetivo

Explorar ferramentas de monitoramento e análise de tráfego de rede — **EtherApe**, **TCPdump** e **Wireshark** — para capturar pacotes, mapear topologias, identificar padrões de comunicação e compreender vulnerabilidades de algoritmos de hash, com foco na colisão MD5.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### Análise de Tráfego de Rede

A análise de tráfego de rede (Network Traffic Analysis — NTA) é uma prática fundamental do Blue Team para detectar comportamentos anômalos, investigar incidentes e compreender a topologia e os fluxos de comunicação de um ambiente. As ferramentas utilizadas nesta atividade atuam em camadas complementares:

| Ferramenta | Camada | Descrição |
|------------|--------|-----------|
| **EtherApe** | Visualização | Exibe o tráfego de rede em tempo real em formato gráfico, mapeando nós e conexões por protocolo |
| **TCPdump** | Captura (CLI) | Captura e exibe pacotes diretamente no terminal com suporte a filtros e exportação para `.pcap` |
| **Wireshark** | Análise (GUI) | Analisa capturas em profundidade, com dissecção completa de protocolos e inspeção de payload |

### Formato PCAP

O formato `.pcap` (Packet Capture) é o padrão para armazenamento de capturas de rede. Arquivos `.pcap` podem ser gerados pelo TCPdump e abertos no Wireshark para análise offline — útil em investigações forenses e em cenários onde a análise em tempo real não é viável.

### Colisão em Função de Hash

Uma função de hash criptográfico mapeia uma entrada de tamanho arbitrário para uma saída de tamanho fixo (o digest). A propriedade de resistência a colisões exige que seja computacionalmente inviável encontrar duas entradas distintas que produzam o mesmo hash. O **MD5** (Message Digest 5), amplamente utilizado para verificação de integridade, teve essa propriedade comprometida em 2004, quando pesquisadores demonstraram a geração de colisões deliberadas — dois arquivos com conteúdos completamente diferentes e o mesmo valor MD5. Por esse motivo, o MD5 não é recomendado para aplicações criptográficas críticas, sendo substituído por algoritmos mais robustos como SHA-256 e SHA-3.

---

## Passo a Passo

### Ambiente

| VM | IP | Usuário | Função |
|----|----|---------|--------|
| linuxserver | [IP_LINUXSERVER] | aluno | Origem do tráfego simulado |
| linuxclient | [IP_LINUXCLIENT] | aluno / root | Captura e análise de tráfego |

---

### Atividade 1 – Mapeando a rede com o EtherApe

#### 1.1 – Verificando a instalação do EtherApe

No terminal do linuxclient como root:

```bash
which etherape
```

#### 1.2 – Verificando a instalação do TCPdump

```bash
which tcpdump
```

#### 1.3 – Capturando tráfego com o TCPdump

Identificação da interface de rede e armazenamento em variável:

```bash
INTERFACE=`ip a | grep -Ei "[IP_LINUXCLIENT]" | rev | cut -d " " -f1 | rev`
```

Captura de 10 pacotes e exportação para arquivo `.pcap`:

```bash
tcpdump -nni $INTERFACE -c 10 -w /root/capture.pcap
```

#### 1.4 – Gerando tráfego a partir do linuxserver

Em outro terminal, conectado ao linuxserver, envio de pacotes ICMP para o linuxclient:

```bash
ping [IP_LINUXCLIENT] -c 10
```

Após a conclusão do ping, o TCPdump finaliza automaticamente ao atingir 10 pacotes capturados.

#### 1.5 – Lendo o arquivo de captura com o TCPdump

```bash
tcpdump -r /root/capture.pcap
```

#### 1.6 – Visualizando a captura no EtherApe

Acesso ao linuxclient via RDP, abertura do EtherApe e carregamento do arquivo:

- Menu **File → Open**
- Navegação até o diretório home e seleção de `capture.pcap`

#### 1.7 – Analisando o tráfego capturado

Observação dos nós (dispositivos) e conexões exibidos graficamente. Uso das ferramentas de zoom e filtragem do EtherApe para explorar padrões de tráfego, fluxos intensos e possíveis anomalias.

#### 1.8 – Simulando tráfego suspeito

Com o EtherApe aberto e em execução em tempo real no linuxclient, envio de 15 pacotes ICMP a partir do linuxserver:

```bash
ping -c 15 [IP_LINUXCLIENT]
```

#### 1.9 – Avaliando a movimentação no EtherApe

Observação do gráfico em tempo real identificando os dispositivos presentes, suas conexões e a direção do tráfego. A "rajada" de pacotes ICMP origina no linuxserver e tem como destino o nó maior (linuxclient).

**Atividade 1** — Print da interface gráfica do EtherApe exibindo o mapa de rede com os dispositivos conectados e o tráfego ICMP simulado.

---

### Atividade 2 – Monitorando o tráfego com o TCPdump

#### 2.1 – Verificando o TCPdump

```bash
which tcpdump
```

#### 2.2 – Iniciando a captura

Fixação da interface em variável e início da captura filtrando por host:

```bash
INTERFACE=`ip a | grep -Ei "[IP_LINUXCLIENT]" | rev | cut -d " " -f1 | rev`
sudo tcpdump -i $INTERFACE host [IP_LINUXCLIENT]
```

#### 2.3 – Gerando tráfego HTTP

Em outro terminal do linuxclient, acesso a um site via cURL:

```bash
curl http://www.uol.com.br
```

#### 2.4 – Observando os pacotes capturados

O TCPdump exibe em tempo real os pacotes trocados durante o acesso HTTP, incluindo endereços IP de origem e destino, flags TCP e informações de sequência. Embora o TCPdump não resolva explicitamente o nome do site, os pacotes refletem o tráfego gerado pelo `curl` para `www.uol.com.br`.

**Atividade 2** — Print do terminal exibindo os pacotes capturados pelo TCPdump durante o acesso ao site.

---

### Atividade 3 – Análise de tráfego com o Wireshark

Acesso ao linuxclient via RDP e abertura do Wireshark. Seleção da interface **ens5** para captura.

Configurações e análises realizadas:

- **Filtro por host:** `ip.addr==192.168.98.11` para isolar apenas o tráfego do linuxclient
- **Análise de pacotes:** identificação de endereços IP de origem e destino, protocolos, portas e flags
- **Exploração de protocolos:** HTTP, DNS, TCP, UDP, ARP e suas interações
- **Investigação de anomalias:** identificação de pacotes perdidos, atrasos e possíveis comportamentos suspeitos

> Esta atividade não exige print para entrega.

---

### Atividade 4 – Estudando um pacote capturado pelo Wireshark

Com o Wireshark capturando na interface **ens5**, acesso ao site `https://esr.rnp.br/` pelo Firefox no linuxclient. Após o carregamento completo da página, captura encerrada via botão **Stop**.

Análise do pacote HTTP capturado respondendo às perguntas do laboratório:

| Pergunta | Resposta |
|----------|----------|
| Número do pacote | Obtido na coluna **No.** do Wireshark |
| Camadas de protocolo | Ethernet → IP → TCP → HTTP (4 camadas) |
| Protocolo de transporte | TCP |
| Endereços físicos (MAC) de origem e destino | Obtidos na camada Ethernet do painel de detalhes |
| Endereços lógicos (IP) de origem e destino | Obtidos na camada IP do painel de detalhes |
| Portas de origem e destino | Obtidas na camada TCP (porta de origem: efêmera; destino: 80) |
| Dados transferidos | Sim — tamanho em bytes obtido no campo **Length** da camada TCP |

> Esta atividade não exige print para entrega.

---

### Atividade 5 – Colisão em função de hash

No terminal do linuxclient, download dos arquivos de demonstração de colisão MD5:

```bash
wget --no-check-certificate "https://docs.google.com/uc?export=download&id=1iUmQP7LjdjQydhSTm3MOqT5xegd-0oQX" -O plane.jpeg
```

```bash
wget --no-check-certificate "https://docs.google.com/uc?export=download&id=1dQLmLnXegZD-lMWzJqHBYGtGSX-7UUGC" -O ship.jpeg
```

Verificação dos arquivos baixados:

```bash
ls -ls
```

Cálculo dos hashes MD5 de ambos os arquivos:

```bash
md5sum plane.jpeg
md5sum ship.jpeg
```

Apesar de serem imagens completamente distintas (avião e navio), os dois arquivos produzem o mesmo digest MD5 — demonstrando na prática o problema de colisão do algoritmo.

**Atividade 5** — Print do terminal exibindo a saída dos comandos `md5sum` com os hashes idênticos para `plane.jpeg` e `ship.jpeg`.

---

## Aprendizado

- O EtherApe oferece uma visão intuitiva da topologia de rede em tempo real, tornando imediatamente visíveis fluxos intensos e comunicações entre hosts. Em ambiente de SOC, essa visualização complementa ferramentas de alerta ao facilitar a correlação espacial de eventos — especialmente útil durante resposta a incidentes para identificar rapidamente quais dispositivos estão se comunicando de forma anômala.
- O TCPdump é uma ferramenta indispensável por sua leveza e flexibilidade: opera inteiramente via CLI, consome poucos recursos e pode ser executado em servidores sem interface gráfica. A combinação de captura com `-w` e leitura com `-r` permite separar o momento da coleta da análise, essencial em ambientes de produção onde a análise interativa prolongada não é viável.
- O Wireshark eleva a análise a outro nível ao dissectar completamente cada protocolo, exibindo campos individuais e relacionamentos entre pacotes. A capacidade de aplicar filtros de exibição (`ip.addr`, `http`, `tcp.flags.syn == 1`) transforma uma captura densa em evidência cirúrgica durante uma investigação.
- A colisão MD5 demonstrada na prática ilustra por que algoritmos com propriedades criptográficas comprometidas não devem ser usados para verificação de integridade em contextos de segurança. Um atacante capaz de produzir colisões pode substituir um arquivo legítimo por um malicioso mantendo o mesmo hash — tornando a verificação de integridade inútil. SHA-256 e SHA-3 são as alternativas recomendadas atualmente.
- A análise de um pacote HTTP no Wireshark evidencia a natureza em texto claro do protocolo: endereços, portas, cabeçalhos e payload são completamente visíveis. Esse exercício reforça a importância do HTTPS e da criptografia em trânsito para qualquer comunicação que envolva dados sensíveis.
