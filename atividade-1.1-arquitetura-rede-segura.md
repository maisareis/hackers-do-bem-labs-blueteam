# Atividade 1.1 – Arquitetura e Design de Rede Segura

## Objetivo

Projetar e documentar uma arquitetura de rede segura com DMZ, VPN e nuvem pública, além de explorar ferramentas de troubleshooting de rede no Linux e no Windows.

## Aviso Legal

> Atividade realizada em ambiente controlado e autorizado para fins educacionais.

---

## Conceito

Uma **rede segura** segmenta o tráfego em zonas com diferentes níveis de confiança. A **DMZ (Zona Desmilitarizada)** isola serviços expostos à internet dos recursos internos. Firewalls controlam o tráfego entre as zonas, e VPNs garantem comunicação cifrada entre sites e usuários remotos.

---

## Passo a Passo

### Atividade 1 – Diagrama de rede da empresa (draw.io)

Ferramenta utilizada: [https://app.diagrams.net](https://app.diagrams.net)

O diagrama inclui:

- **DMZ** (range sugerido: `192.168.0.0/24`): firewall externo, servidores web e de e-mail
- **Intranet** (range sugerido: `10.0.0.0/24`): servidores internos, estações de trabalho
- **Firewall externo**: entre internet e DMZ
- **Firewall interno**: entre DMZ e intranet
- **Switches**: interconectando dispositivos dentro de cada zona

### Atividade 2 – Recursos avançados (VPN e nuvem)

Adicionados ao diagrama:

- **VPN site-to-site**: conecta sede e filial passando pelos firewalls de ambos os lados
- **VPN client-to-site**: permite que colaboradores em home office acessem a rede interna via firewall externo
- **Nuvem pública (AWS)**: instâncias hospedadas com conexão à DMZ

### Atividade 3 – PowerShell vs CMD (windowsclient)

Listar executáveis em `C:\Windows` maiores que 61.952 bytes, excluindo os que começam com "r":

```powershell
Get-ChildItem -Path C:\Windows -Recurse -File -Include *.exe | Where-Object { $_.Name -notmatch '^r' -and $_.Length -gt 61952 }
```

Listar DLLs em `C:\Windows\System32` cujos nomes não começam com "A" ou "B":

```powershell
Get-ChildItem C:\Windows\System32\*.dll | Where-Object {$_.Name -NotLike "A*" -and $_.Name -NotLike "B*"}
```

Listar executáveis iniciados com "C" ou "D" e salvar em HTML:

```powershell
Get-ChildItem C:\Windows\System32\*.exe | Where-Object {$_.Name -Like "C*" -or $_.Name -like "D*"} | ConvertTo-Html | Out-File C:\"arquivos.html"
```

Salvar serviços em execução:

```powershell
Get-Service | Where-Object {$_.Status -eq "Running"} | Out-File C:\"ServiçosExecução.txt"
```

Verificar configuração de rede:

```powershell
Get-NetAdapter | Format-List
Get-NetIpConfiguration -Detailed
```

```cmd
ipconfig /all
```

### Atividade 4 – Troubleshooting de rede no Linux (linuxclient)

```bash
nmcli device show
nmcli -f NAME,FILENAME connection
ifconfig ens5
ip addr show ens5
ip link ls up
```

Verificar sockets no linuxserver:

```bash
ss -tnop
ss -o state listening
ethtool ens5
```

### Atividade 5 – iperf: teste de throughput

No linuxserver (modo servidor TCP):

```bash
sudo iperf -s
```

No linuxclient (modo cliente TCP):

```bash
sudo iperf -c 192.168.98.10
```

Teste com UDP:

```bash
# servidor
sudo iperf -s -u

# cliente
sudo iperf -c 192.168.98.10 -u
sudo iperf -c 192.168.98.10 -u -b 77M
```

Opções adicionais:

```bash
sudo iperf -c 192.168.98.10 -i 3          # exibir status a cada 3s
sudo iperf -c 192.168.98.10 -P 8          # 8 conexões paralelas
sudo iperf -c 192.168.98.10 -i 2 -t 15 -f Mbytes
```

---

## Aprendizado

- A DMZ protege a rede interna ao isolar serviços expostos, mesmo que um servidor na DMZ seja comprometido
- VPN site-to-site e client-to-site atendem cenários diferentes: filiais fixas e usuários remotos respectivamente
- O PowerShell supera o CMD na automação, mas o `ipconfig` ainda oferece informações mais diretas sobre DHCP
- `nmcli`, `ifconfig`, `ip addr` e `ss` cobrem diferentes aspectos do diagnóstico de rede no Linux
- O `iperf` mede throughput real da rede e diferencia latência (jitter) e perda de pacotes no modo UDP
