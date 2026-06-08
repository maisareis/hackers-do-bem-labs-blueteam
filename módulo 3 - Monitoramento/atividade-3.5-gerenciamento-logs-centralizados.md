# Atividade 3.5 – Gerenciamento de Logs Centralizados

## Objetivo

Implementar uma infraestrutura de gerenciamento de logs centralizado utilizando o **Graylog**, configurando a sincronização de tempo com **NTP**, integrando fontes de log via **rsyslog** e aplicando pipelines de processamento com padrões **GROK** para normalização e análise eficiente de eventos de segurança.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### Gerenciamento de Logs Centralizado

O gerenciamento centralizado de logs consiste em coletar, armazenar e analisar eventos gerados por diferentes ativos de uma infraestrutura em um único ponto. Isso permite correlacionar eventos, identificar atividades suspeitas e responder a incidentes com maior eficiência.

| Componente | Função |
|------------|--------|
| **Graylog** | Plataforma de coleta, indexação e análise de logs |
| **ElasticSearch** | Backend de indexação e armazenamento distribuído de logs |
| **MongoDB** | Armazenamento de metadados e configurações do Graylog |
| **rsyslog** | Serviço de encaminhamento de logs do sistema operacional |
| **NTP** | Sincronização de tempo entre os hosts para consistência dos logs |
| **GROK** | Padrões baseados em expressões regulares para extração de campos de logs |

### Arquitetura de Alta Disponibilidade

Uma arquitetura escalável de gerenciamento de logs organiza os componentes em camadas:

- **Geradores de log:** servidores Linux, servidores Windows, firewalls, roteadores, switches e aplicativos
- **Balanceador de carga:** distribui o tráfego de entrada entre os nós de processamento
- **Cluster de processamento:** nós Graylog em paralelo com MongoDB Replica Set para alta disponibilidade
- **Backend distribuído:** múltiplos nós ElasticSearch com armazenamento local
- **Softwares de suporte:** HIDS, ferramentas de visualização, rotacionamento de logs e Archive NFS

### NTP e Confiabilidade dos Logs

A sincronização de tempo é fundamental para a integridade dos logs. Timestamps inconsistentes entre hosts dificultam a correlação de eventos e podem comprometer investigações de incidentes. O NTP garante que todos os sistemas da infraestrutura utilizem a mesma referência de tempo.

### Pipelines e GROK no Graylog

Os pipelines permitem transformar e enriquecer mensagens de log antes do armazenamento. Cada pipeline é composto por estágios contendo regras que definem condições e ações. O GROK utiliza padrões de expressão regular para extrair campos estruturados de mensagens de texto livre, como hostname, status, usuário e IP de origem a partir de logs do SSH.

---

## Passo a Passo

### Ambiente

| VM | Usuário | Função |
|----|---------|--------|
| linuxserver | root | Servidor Graylog, NTP e rsyslog |
| linuxclient | root | Cliente rsyslog |

---

### Atividade 1 – Proposta de Arquitetura

Elaboração do diagrama de arquitetura escalável e de alta disponibilidade no **Draw.io** (`https://app.diagrams.net`), contemplando todos os componentes da infraestrutura de gerenciamento de logs centralizado organizados em camadas.

**Atividade 1** — Diagrama de arquitetura escalável e de alta disponibilidade para gerenciamento de logs centralizado, elaborado no Draw.io, contemplando geradores de log (servidores Linux, Windows, firewall, roteador, switch, aplicativos e estações), balanceador de carga BigIP F5, cluster de processamento com dois nós Graylog e MongoDB Replica Set, backend de armazenamento distribuído com três nós ElasticSearch e softwares de suporte (HIDS, visualização de dados, rotacionamento de logs e Archive NFS).

---

### Atividade 2 – Analisando Logs com o LogPad

Análise exploratória de arquivo de logs de firewall utilizando a ferramenta web **LogPad** (`https://cloudvyzor.com/`). A atividade demonstra como filtrar eventos específicos em arquivos de log de grande volume, identificando tentativas de acesso SSH, ataques de força bruta contra o usuário root e tentativas de login com usuários inválidos.

> Esta atividade não exige print para entrega.

---

### Atividade 3 – Configurando o Servidor NTP

Acesso ao linuxserver como root e verificação do ambiente:

```bash
whoami
which ntpd
```

Verificação da timezone atual:

```bash
date
```

Correção da timezone para America/Sao_Paulo (GMT-3):

```bash
dpkg-reconfigure tzdata
```

Atualização do espelho Debian para o espelho brasileiro:

```bash
sed -i 's/http:\/\/deb.debian.org/http:\/\/deb.debian.org.br/g' /etc/apt/sources.list
```

Atualização da lista de pacotes:

```bash
apt update
```

Substituição dos servidores NTP pelo pool brasileiro:

```bash
sed -i 's/pool 0.debian.pool.ntp.org iburst/\
server 0.br.pool.ntp.org iburst\
server 1.br.pool.ntp.org iburst\
server 2.br.pool.ntp.org iburst/g' /etc/ntpsec/ntp.conf
```

Parada do serviço, sincronização forçada com o pool remoto e reinicialização:

```bash
service ntpsec stop
ntpd -gq
service ntpsec start
```

Verificação da data e hora com timezone corrigida:

```bash
date
```

**Atividade 3** — Configuração do servidor NTP concluída com sucesso no linuxserver, exibindo o resultado do comando `date` com timezone corrigida para o fuso horário de Brasília (GMT-3), confirmando a sincronização do horário com o pool de servidores NTP brasileiros.

---

### Atividade 4 – Primeiros Passos com o Graylog

Liberação das portas necessárias no firewall UFW:

```bash
ufw allow 9000
ufw allow 5140
```

Verificação do status do serviço Graylog:

```bash
systemctl status graylog-server.service
```

#### Configuração do rsyslog no linuxserver

Adição da linha de encaminhamento de logs ao arquivo `/etc/rsyslog.conf`:

```
*.* @@127.0.0.1:5140
```

Reinicialização do rsyslog:

```bash
systemctl restart rsyslog.service
```

#### Configuração do rsyslog no linuxclient

Adição da linha de encaminhamento de logs ao arquivo `/etc/rsyslog.conf`:

```
*.* @@[IP_LINUXSERVER]:5140
```

Reinicialização do rsyslog:

```bash
systemctl restart rsyslog.service
```

#### Acesso à Interface Web

Acesso ao Graylog via navegador: `http://[IP_LINUXSERVER]:9000`

Credenciais de administrador configuradas durante a instalação.

#### Configuração da Fonte de Logs (Input)

No painel do Graylog: **System → Inputs → Select Input → Syslog TCP → Launch new input**

Parâmetros configurados:

| Campo | Valor |
|-------|-------|
| Title | Syslog TCP |
| Bind address | 0.0.0.0 |
| Port | 5140 |

#### Teste de Recebimento de Logs

Geração de log de teste no linuxserver:

```bash
logger "teste graylog"
```

Busca no painel **Search** do Graylog para confirmar o recebimento.

#### Criação do Dashboard

No painel do Graylog: **Dashboards → Create new dashboard**

Adição de widget do tipo **Message Count** ao painel criado.

**Atividade 4** — Painel de controle criado no Graylog exibindo widget de contagem de mensagens (Message Count), confirmando o recebimento e centralização dos logs enviados pelo linuxserver via Syslog TCP na porta 5140.

---

### Atividade 5 – Uso de Pipelines e GROK

> Esta atividade não exige print para entrega.

#### Criação da Pipeline

No painel do Graylog: **System → Pipelines → Add new pipeline**

| Campo | Valor |
|-------|-------|
| Title | SSH login |
| Description | SSH login |

Conexão da pipeline à **Default Stream** via **Edit Connections**.

#### Criação das Regras GROK

Acesso ao menu **Manage Rules → Create rule → Use Source Code Editor**.

Regra para logins aceitos via SSH:

```
rule "ssh login"
when
  ! contains(
  value: to_string($message."message"),
  search: "Accepted password"
)
then
  let gl2_fragment_grok_results = grok(
  pattern: "%{HOSTNAME:hostname} sshd\[%{NUMBER}\]\: %{WORD:status} password
for %{USER:user} from %{IPV4:source_ip} port %{NUMBER} ssh2",
  value: to_string($message."message")
);
set_fields(
  fields: gl2_fragment_grok_results
);
end
```

Regra para logins com falha via SSH:

```
rule "ssh failed"
when
  ! contains(
  value: to_string($message."message"),
  search: "Failed password"
)
then
  let gl2_fragment_grok_results = grok(
  pattern: "%{HOSTNAME:hostname} sshd\[%{NUMBER}\]\: %{WORD:status} password
for %{USER:user} from %{IPV4:source_ip} port %{NUMBER} ssh2",
  value: to_string($message."message")
);
set_fields(
  fields: gl2_fragment_grok_results
);
end
```

#### Adição das Regras ao Stage 0 da Pipeline

**Manage Pipelines → Edit → Stage 0 → Edit → Stage rules:** adicionar `ssh login` e `ssh failed` → **Update Stage**.

Após acessar o linuxserver e linuxclient via SSH, os logs passam a exibir os campos normalizados:

| Campo | Descrição |
|-------|-----------|
| `hostname` | Nome do host de origem |
| `status` | Status da autenticação (Accepted / Failed) |
| `user` | Usuário que tentou autenticar |
| `source_ip` | IP de origem da tentativa |

---

## Aprendizado

- A sincronização de tempo via NTP é um pré-requisito crítico para qualquer infraestrutura de logs. Timestamps inconsistentes entre hosts inviabilizam a correlação de eventos e comprometem investigações de incidentes.
- O Graylog centraliza logs de múltiplas fontes em uma única interface, permitindo buscas, filtros e visualizações que facilitam a identificação de padrões suspeitos — como os ataques de força bruta evidenciados na análise do arquivo de logs de firewall.
- A análise do arquivo de logs no LogPad demonstrou na prática como um ataque de força bruta se manifesta nos registros do sistema: alto volume de entradas `Failed password for root` e `Failed password for invalid user` provenientes de múltiplos IPs em curto intervalo de tempo.
- O uso de padrões GROK transforma mensagens de texto livre em campos estruturados e pesquisáveis, aumentando significativamente a capacidade de filtragem e correlação. Campos como `source_ip` e `status` extraídos dos logs SSH permitem criar alertas e dashboards precisos.
- Uma arquitetura de alta disponibilidade com balanceador de carga, cluster Graylog e múltiplos nós ElasticSearch garante que a infraestrutura de logs não se torne um ponto único de falha — o que seria crítico em um ambiente de segurança onde a perda de logs pode comprometer uma investigação.
