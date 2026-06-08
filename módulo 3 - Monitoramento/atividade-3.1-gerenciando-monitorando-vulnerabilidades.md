# Atividade 3.1 – Gerenciando e Monitorando Vulnerabilidades

## Objetivo

Explorar técnicas de coleta de informações via **OSINT**, reconhecimento de rede com **Nmap** e redução da superfície de ataque por meio de **hardening** em sistemas Windows Server e Linux Debian, incluindo desativação de serviços desnecessários, configuração de firewall, políticas de senha e hardening de aplicações DNS.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### OSINT (Open-Source Intelligence)

OSINT é a prática de coletar informações publicamente disponíveis sobre um alvo utilizando fontes abertas — mecanismos de busca, redes sociais, bases de dados de vazamentos e ferramentas especializadas. No contexto do Blue Team, OSINT é utilizado para mapear a exposição de informações de uma organização ou usuário na internet, permitindo identificar dados que poderiam ser explorados por atacantes.

| Técnica | Descrição |
|---------|-----------|
| **Google Dorking** | Uso de operadores avançados (`site:`, `inurl:`, `intitle:`) para filtrar resultados específicos |
| **Redes Sociais** | Levantamento de informações públicas em perfis de LinkedIn, Twitter, Instagram, etc. |
| **Webmii** | Ferramenta de agregação de perfis públicos na internet |
| **Have I Been Pwned** | Verificação de e-mails em bases de dados de vazamentos |
| **Busca reversa de imagens** | Identificação de outras contas ou sites associados a uma imagem |

### Nmap

O **Nmap** é uma ferramenta de mapeamento de rede amplamente utilizada por equipes de segurança para descoberta de hosts, enumeração de portas abertas, detecção de versões de serviços e identificação de sistemas operacionais (OS fingerprinting).

| Técnica | Flag | Descrição |
|---------|------|-----------|
| Escaneamento de portas | `-p` | Define o intervalo de portas a escanear |
| Detecção de versão | `-sV` | Identifica versões dos serviços em execução |
| Descoberta de hosts | `-sn` | Mapeia hosts ativos na rede sem escanear portas |
| TCP Connect | `-sT` | Realiza conexão TCP completa (three-way handshake) |
| TCP Null | `-sN` | Envia pacotes sem flags; portas fechadas respondem com RST |
| TCP FIN | `-sF` | Stealth scan que atravessa filtros de pacotes SYN |
| TCP Xmas | `-sX` | Envia pacotes com flags FIN, PSH e URG simultâneas |
| OS Fingerprinting | `-O` | Tenta identificar o sistema operacional do alvo |

### Hardening

Hardening é o conjunto de práticas que reduz a superfície de ataque de um sistema por meio da desativação de serviços desnecessários, aplicação de políticas de controle de acesso, configuração de firewalls e restrição de funcionalidades não utilizadas. O princípio fundamental é o do **menor privilégio**: cada componente deve ter acesso apenas ao estritamente necessário para sua função.

---

## Passo a Passo

### Ambiente

| VM | Usuário | Função |
|----|---------|--------|
| linuxserver | root | Alvo dos scans e hardening Linux |
| linuxclient | root | Origem dos scans Nmap |
| windowsserver | Administrator | Hardening Windows |

---

### Atividade 1 – OSINT

Pesquisa de informações públicas sobre um usuário-alvo utilizando o próprio computador, sem acesso às VMs do laboratório.

Ferramentas e técnicas utilizadas:

- **Google:** busca pelo nome do usuário com aspas para pesquisa exata e operadores avançados (`site:linkedin.com`, `inurl:`, `intitle:`)
- **Redes sociais:** levantamento de informações públicas em perfis de LinkedIn, Twitter, Instagram e Facebook
- **Webmii** (`https://webmii.com/`): agregação de perfis públicos e links a redes sociais
- **Have I Been Pwned:** verificação de e-mails em bases de dados de vazamentos
- **Google Imagens / TinEye:** busca reversa de imagens para identificar outros perfis associados

**Atividade 1** — Print da tela exibindo os resultados da pesquisa OSINT, incluindo informações coletadas via Webmii e demais fontes abertas consultadas.

---

### Atividade 2 – Utilizando o Nmap

Realizado a partir da VM linuxclient.

#### Escaneamento de Portas

```bash
nmap -p 1-1000 [IP_LINUXSERVER]
```

#### Detecção de Versão de Serviços

```bash
nmap -sV [IP_LINUXSERVER]
```

#### Escaneamento de Rede

```bash
nmap -sn [REDE]/24
```

> Esta atividade não exige print para entrega.

---

### Atividade 3 – Comandos Avançados no Nmap

Realizado a partir da VM linuxclient. Os comandos abaixo requerem privilégios de root.

#### TCP Connect Scan

```bash
nmap -sT [IP_LINUXSERVER]
```

#### TCP Null Scan

```bash
nmap -sN [IP_LINUXSERVER]
```

#### TCP FIN Scan

```bash
nmap -sF [IP_LINUXSERVER]
```

> O TCP FIN (Stealth FIN) atravessa filtros que bloqueiam pacotes SYN. Portas fechadas respondem com RST; portas abertas ignoram o pacote. O comportamento pode variar conforme o sistema operacional alvo.

#### TCP Xmas Scan

```bash
nmap -sX [IP_LINUXSERVER]
```

**Atividade 3** — Print da tela exibindo o resultado do último escaneamento avançado executado no Nmap (TCP Xmas Scan), listando as portas identificadas como abertas no linuxserver.

---

### Atividade 4 – OS Fingerprinting com Nmap

Realizado como root a partir do linuxclient.

```bash
nmap -O [IP_LINUXSERVER]
```

Verificação do sistema operacional real no linuxserver para comparação com o resultado do Nmap:

```bash
hostname
uname -r
```

> O resultado do OS fingerprinting costuma ser próximo da realidade, mas não necessariamente exato. Esta atividade não exige print para entrega.

---

### Atividade 5 – Reduzindo a Superfície Vulnerável

#### 5.1 – Hardening no Windows Server

Acesso ao windowsserver como Administrator via RDP.

Listagem de serviços em execução:

```powershell
Get-Service | Where-Object {$_.Status -eq 'Running'}
```

Desativação do serviço de impressão (spooler):

```powershell
Stop-Service -Name spooler
Set-Service -Name spooler -StartupType Disabled
```

Configuração do firewall para permitir apenas RDP e SSH da rede interna:

```powershell
# Permitir RDP apenas da rede interna
New-NetFirewallRule -DisplayName "Allow RDP from LAN" -Direction Inbound -Protocol TCP -LocalPort 3389 -RemoteAddress [REDE]/24 -Action Allow

# Permitir SSH apenas da rede interna
New-NetFirewallRule -DisplayName "Allow SSH from LAN" -Direction Inbound -Protocol TCP -LocalPort 22 -RemoteAddress [REDE]/24 -Action Allow

# Regra padrão: permitir todo tráfego de saída
Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultOutboundAction Allow

# Regra padrão: bloquear todo tráfego de entrada
Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Block
```

Configuração de expiração de senha do Administrator:

```powershell
Set-LocalUser -Name "Administrator" -PasswordNeverExpires $false
```

Instalação e configuração do **NxLog Community Edition** para encaminhamento de logs Windows no formato Syslog para o servidor centralizado.

Configuração adicionada ao arquivo `C:\Program Files\nxlog\conf\nxlog.conf`:

```
<Input in>
  Module im_msvistalog
</Input>

<Output out>
  Module om_udp
  Host [IP_LINUXSERVER]
  Port 5140
  Exec to_syslog_ietf();
</Output>

<Route 1>
  Path in => out
</Route>
```

Reinicialização do serviço NxLog:

```powershell
Restart-Service -Name nxlog
```

Geração de evento de teste e verificação no Event Viewer:

```powershell
New-EventLog -LogName "MeuLog" -Source "MinhaFonte"
Write-EventLog -LogName "MeuLog" -Source "MinhaFonte" -EventId 1 -EntryType Warning -Message "Olá Mundo!!!"
```

Configuração do Windows Exploit Guard (DEP e SEHOP):

```powershell
Set-ProcessMitigation -System -Enable "DEP","SEHOP"
Set-ProcessMitigation -Name AcroRd32.exe -Enable DEP,HighEntropy,StrictHandle,TerminateOnError,BottomUp
Set-ProcessMitigation -Name chrome.exe -Enable DEP,HighEntropy,StrictHandle,TerminateOnError,BottomUp
Get-ProcessMitigation -System
```

Desativação de compartilhamentos administrativos:

```powershell
Remove-SmbShare -Name NETLOGON
Remove-SmbShare -Name SYSVOL
Stop-Service -Name LanmanServer -Force
Restart-Service -Name lanmanserver

# Impedir recriação automática via registro
New-ItemProperty -Name AutoShareServer -Path HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters -Type DWORD -Value 0
```

Após reiniciar o windowsserver, verificação dos compartilhamentos restantes:

```powershell
Get-SmbShare
Get-ProcessMitigation -System
```

**Atividade 5** — Print da tela do PowerShell exibindo o resultado dos comandos `Get-SmbShare` e `Get-ProcessMitigation -System` após a aplicação das políticas de hardening no Windows Server, confirmando a remoção dos compartilhamentos administrativos e as configurações de mitigação de exploit ativas.

---

#### 5.2 – Hardening no Linux Debian

Acesso ao linuxserver como root.

Listagem de serviços em execução:

```bash
sudo systemctl list-units --type=service --state=running
```

Desativação do serviço de impressão (cups):

```bash
sudo systemctl stop cups
sudo systemctl disable cups
```

Configuração do firewall UFW:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

Configuração de política de senhas fortes via PAM:

```bash
sudo nano /etc/pam.d/common-password
```

Linha adicionada ao arquivo:

```
password requisite pam_pwquality.so retry=3 minlen=12 difok=3
```

Desativação do UFW ao final do laboratório:

```bash
sudo ufw disable
```

---

#### 5.3 – Hardening de Aplicações (DNS no Debian)

Restrição de transferências de zona no BIND para impedir que servidores não autorizados obtenham cópia completa da zona DNS:

```bash
sudo nano /etc/bind/named.conf.options
```

Linha adicionada dentro da seção `options`:

```
allow-transfer { none; };
```

Reinicialização do serviço BIND:

```bash
sudo systemctl restart bind9
```

---

## Aprendizado

- A pesquisa OSINT demonstra como informações públicas dispersas em diferentes plataformas podem ser consolidadas para revelar uma quantidade significativa de dados sobre um alvo. Do ponto de vista defensivo, o exercício reforça a importância de revisar periodicamente a exposição de dados pessoais e corporativos em fontes abertas.
- Os diferentes modos de escaneamento do Nmap (Connect, Null, FIN, Xmas) exploram comportamentos distintos da pilha TCP para identificar portas abertas contornando filtros de pacotes. O TCP FIN e o Xmas são especialmente úteis em cenários onde pacotes SYN são bloqueados por firewalls, mas seu comportamento varia conforme o sistema operacional alvo.
- O OS fingerprinting do Nmap raramente retorna o resultado exato, mas costuma ser preciso o suficiente para orientar um atacante. Isso reforça a importância de não expor serviços desnecessários na rede, reduzindo as informações disponíveis para reconhecimento passivo.
- A desativação do serviço de impressão (spooler no Windows, cups no Linux) é uma medida simples com impacto relevante: o spooler já foi vetor de vulnerabilidades críticas como o PrintNightmare (CVE-2021-34527), e desativá-lo em servidores que não precisam dessa função elimina esse vetor por completo.
- A restrição de transferências de zona DNS (`allow-transfer { none; }`) impede que atacantes obtenham um mapeamento completo da infraestrutura interna via AXFR — uma técnica clássica de reconhecimento que pode expor hostnames, IPs e a estrutura da rede.
- A configuração do NxLog no Windows resolve um problema real: logs Windows usam um formato proprietário (Event Log) incompatível com o padrão Syslog. O agente converte e encaminha os eventos para o servidor centralizado, garantindo visibilidade unificada independentemente do sistema operacional da fonte.
