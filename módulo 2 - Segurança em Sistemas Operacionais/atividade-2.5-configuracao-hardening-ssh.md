# Atividade 2.5 – Configuração e Hardening do SSH

## Objetivo

Aplicar boas práticas de hardening no SSH: criação de uma segunda instância em porta alternativa, desativação do acesso root remoto, autenticação exclusivamente por chaves e criação de um túnel SOCKS5 para mascaramento de IP.

## Aviso Legal

> Atividade realizada em ambiente controlado e autorizado para fins educacionais. IPs, usuários e senhas utilizados são exclusivos do ambiente de laboratório e não estão expostos neste documento.

---

## Conceito

O SSH é um protocolo extremamente versátil: além da administração remota, pode ser usado para transferência de arquivos e como VPN. Um hardening bem feito reduz a superfície de ataque sem perder funcionalidade. As principais medidas aplicadas aqui são:

- **Porta alternativa**: dificulta varreduras automatizadas que buscam a porta 22
- **Bloqueio do root remoto**: elimina ataques de força bruta direto ao superusuário
- **Autenticação por chave**: remove a dependência de senhas, que são vulneráveis a brute force e eavesdropping
- **Túnel SOCKS5 (`-D`)**: permite rotear tráfego HTTP pelo SSH, mascarando o IP de origem

---

## Passo a Passo

### Atividade 1 – Criação de segunda instância do SSH

A segunda instância serve como caminho de recuperação: se uma configuração quebrar o SSH principal, ainda é possível acessar o host pela instância alternativa.

Acessar o host Linux alvo como root via SSH:

```bash
ssh root@IP_HOST_LINUX
# Responder 'yes' na primeira conexão
```

Criar cópia do arquivo de configuração do SSH para a segunda instância:

```bash
cp /etc/ssh/sshd_config /etc/ssh/2fa_sshd_config
```

Alterar a porta da segunda instância para uma porta alta (ex: 23422):

```bash
sed -i "s/#Port 22/Port 23422/" /etc/ssh/2fa_sshd_config
```

Criar o arquivo de serviço systemd para a segunda instância:

```bash
cp /lib/systemd/system/ssh.service /lib/systemd/system/ssh_2fa.service
```

Ajustar o serviço para usar os arquivos da segunda instância:

```bash
sed -i "s/EnvironmentFile=-\/etc\/default\/ssh/EnvironmentFile=-\/etc\/default\/ssh_2fa/" /lib/systemd/system/ssh_2fa.service
sed -i "s/ExecStartPre=\/usr\/sbin\/sshd -t/ExecStartPre=\/usr\/sbin\/sshd -t -f \/etc\/ssh\/2fa_sshd_config/" /lib/systemd/system/ssh_2fa.service
sed -i "s/ExecStart=\/usr\/sbin\/sshd -D \$SSHD_OPTS/ExecStart=\/usr\/sbin\/sshd -D \$SSHD_OPTS -f \/etc\/ssh\/2fa_sshd_config/" /lib/systemd/system/ssh_2fa.service
sed -i "s/ExecReload=\/usr\/sbin\/sshd -t/ExecReload=\/usr\/sbin\/sshd -t -f \/etc\/ssh\/2fa_sshd_config/" /lib/systemd/system/ssh_2fa.service
sed -i "s/Alias=sshd.service/Alias=sshd_2fa.service/" /lib/systemd/system/ssh_2fa.service
```

Habilitar e iniciar a segunda instância:

```bash
systemctl enable ssh_2fa.service
systemctl start ssh_2fa.service
```

Verificar se ambas as portas estão ativas:

```bash
netstat -tupan4 | grep -Ei "ssh"
```

**Saída esperada:**

```
tcp  0  0  0.0.0.0:22     0.0.0.0:*  LISTEN  .../sshd
tcp  0  0  0.0.0.0:23422  0.0.0.0:*  LISTEN  .../sshd
```

Conectar pela segunda instância usando o parâmetro `-p`:

```bash
ssh root@IP_HOST_LINUX -p 23422
```

---

### Atividade 2 – Configuração do SSH: chaves, bloqueio de root e senha

**Geração do par de chaves (no host Windows, via PowerShell):**

```powershell
ssh-keygen.exe -t rsa -b 2048 -f id_rsa_vpn -C "vpn-hacker-do-bem"
# Insira uma passphrase segura quando solicitado
```

Verificar os arquivos gerados:

```powershell
ls
# id_rsa_vpn      → chave privada (nunca compartilhar)
# id_rsa_vpn.pub  → chave pública (será copiada para o servidor)
```

**Copiar a chave pública para o host Linux (via SCP pela porta alternativa):**

```powershell
scp -P 23422 .\id_rsa_vpn.pub usuario_vpn@IP_HOST_LINUX:/home/usuario_vpn/.ssh/authorized_keys
```

**Desabilitar acesso root remoto e autenticação por senha (no host Linux):**

```bash
ssh root@IP_HOST_LINUX -p 23422

sed -i "s/PermitRootLogin yes/PermitRootLogin no/" /etc/ssh/2fa_sshd_config
sed -i "s/PasswordAuthentication yes/PasswordAuthentication no/" /etc/ssh/2fa_sshd_config

systemctl restart ssh_2fa.service
```

**Testar o bloqueio:**

```bash
# Tentativa com root → deve ser negada
ssh root@IP_HOST_LINUX -p 23422
# root@IP: Permission denied (publickey).

# Tentativa com usuário vpn sem informar a chave → deve ser negada
ssh usuario_vpn@IP_HOST_LINUX -p 23422
# usuario_vpn@IP: Permission denied (publickey).

# Conexão correta: informando a chave privada
ssh usuario_vpn@IP_HOST_LINUX -p 23422 -i .\id_rsa_vpn
# Insira a passphrase quando solicitado
```

---

### Atividade 3 – Criação do túnel SSH (SOCKS5)

O parâmetro `-D` abre uma porta local que funciona como proxy SOCKS5. Todo tráfego enviado por essa porta é encaminhado pelo túnel SSH e sai pelo IP do host remoto.

```bash
ssh usuario_vpn@IP_HOST_LINUX -p 23422 -D 8765 -i .\id_rsa_vpn
# Mantenha o terminal aberto — o túnel fica ativo enquanto a sessão existir
```

---

### Atividade 4 – Configurar o Firefox para usar o túnel

Instalar a extensão **FoxyProxy Standard** no Firefox:
[https://addons.mozilla.org/pt-BR/firefox/addon/foxyproxy-standard/](https://addons.mozilla.org/pt-BR/firefox/addon/foxyproxy-standard/)

Configurar o proxy em **Options → Proxies → Add**:

| Campo | Valor |
|-------|-------|
| Title | Hacker do bem |
| Proxy Type | SOCKS5 |
| Hostname | 127.0.0.1 |
| Port | 8765 |
| Send DNS through SOCKS5 proxy | On (padrão) |

Clicar em **Save**.

---

### Atividade 5 – Verificar o mascaramento de IP

**Sem proxy ativo:** acessar uma página que exibe o IP de origem. O IP exibido será o da máquina local (Windows Server).

**Com proxy ativo:** ativar o perfil "Hacker do bem" no FoxyProxy e recarregar a página (F5). O IP exibido deve ser o do host Linux que está servindo como ponto de saída do túnel.

> O tráfego segue o caminho: Firefox → porta local 8765 → túnel SSH → host Linux → servidor web. O servidor web enxerga o IP do host Linux, não o do cliente original.

---

## Aprendizado

- Rodar o SSH em porta alternativa não é segurança por obscuridade suficiente sozinha, mas reduz significativamente o ruído de bots automatizados nos logs
- Desabilitar o login root remoto e a autenticação por senha são as duas medidas de hardening SSH com maior impacto imediato
- A autenticação por chave é superior à senha: elimina ataques de força bruta e não expõe credencial em trânsito
- O túnel SOCKS5 via `ssh -D` demonstra como o protocolo SSH pode funcionar como uma VPN leve, roteando tráfego arbitrário de forma cifrada
- Manter uma segunda instância do SSH em porta alternativa é uma boa prática de segurança operacional: garante acesso de recuperação caso uma configuração incorreta bloqueie a instância principal
