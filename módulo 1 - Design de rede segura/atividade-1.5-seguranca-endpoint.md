# Atividade 1.5 – Segurança do Endpoint

## Objetivo

Aplicar práticas de hardening em um servidor Linux: remoção de serviços desnecessários, controle de acesso via `sudo`, restrição do binário `su` e uso do antivírus ClamAV para detecção de arquivos maliciosos.

## Aviso Legal

> Atividade realizada em ambiente controlado e autorizado para fins educacionais. IPs, usuários e senhas utilizados são exclusivos do ambiente de laboratório e não estão expostos neste documento.

---

## Conceito

**Hardening de endpoint** é o processo de reduzir a superfície de ataque de um sistema removendo serviços desnecessários, restringindo privilégios e monitorando ameaças. Cada serviço ativo é uma potencial porta de entrada.

---

## Passo a Passo

### Atividade 1 – Remoção de serviços desnecessários

Verificar se o FTP está instalado:

```bash
which ftp
```

Remover o FTP:

```bash
sudo apt-get remove ftp -y
```

Identificar serviços escutando por conexões TCP e UDP:

```bash
netstat -tnlp    # TCP
netstat -unlp    # UDP
```

Investigar arquivos abertos e conexões com `lsof`:

```bash
sudo lsof -i tcp:25 -n
```

Identificar o pacote associado à porta 25 (SMTP):

```bash
dpkg -l | grep exim
```

Remover o pacote junto com seus arquivos de configuração:

```bash
sudo -i
apt-get purge exim4* -y
exit
```

> **Por que remover o Exim?** Servidores de e-mail expostos sem necessidade ampliam a superfície de ataque. Serviços como Telnet, FTP, NFS e SNMP devem ser avaliados e desativados quando não forem essenciais ao ambiente.

---

### Atividade 2 – Controle de acesso a comandos com sudo

Verificar se o `sudo` está instalado:

```bash
which sudo
```

Adicionar usuário ao grupo sudo e configurar execução sem senha:

```bash
sudo -i
usermod -aG sudo nome_usuario
visudo
```

Linha a adicionar no `visudo` abaixo de `Defaults env_reset`:

```
nome_usuario ALL=(ALL) NOPASSWD: ALL
```

Criar novo usuário com permissão restrita (apenas `iptables`):

```bash
useradd -m nome_usuario2
passwd nome_usuario2
visudo
```

Linha a adicionar no `visudo`:

```
nome_usuario2 ALL=(root) /sbin/iptables
```

Testar a configuração:

```bash
su - nome_usuario2
/sbin/iptables    # deve executar sem solicitar senha
exit
```

---

### Atividade 3 – Controlando o uso do binário `su`

O comando `su` permite trocar de usuário, inclusive para root. Restringir seu uso a um grupo específico reduz o risco de escalonamento de privilégios não autorizado.

Criar usuário, grupo `wheel` e adicionar o usuário ao grupo:

```bash
useradd -m nome_usuario
passwd nome_usuario
groupadd wheel
usermod -aG wheel nome_usuario
```

Editar o PAM para restringir o `su` ao grupo `wheel`:

```bash
nano /etc/pam.d/su
```

Adicionar no final do arquivo:

```
auth required pam_wheel.so trust use_uid
```

Testar: usuário no grupo `wheel` consegue usar `su -`, outros recebem `Permission denied`:

```bash
su - nome_usuario    # sucesso (membro do wheel)
su -                 # funciona a partir do membro do wheel

su - outro_usuario   # funciona para trocar para esse usuário
su -                 # deve retornar: su: Permission denied
```

---

### Atividade 4 – Antivírus ClamAV: configuração

Verificar a instalação:

```bash
which clamscan
```

Atualizar as definições de vírus:

```bash
freshclam
```

Verificar a versão:

```bash
clamscan --version
```

Realizar varredura completa do sistema:

```bash
clamscan -r /
# Atenção: pode levar mais de 15 minutos
```

Agendar varredura diária às 2h via cron:

```bash
crontab -e
```

Linha a adicionar:

```
0 2 * * * clamscan -r /
```

---

### Atividade 5 – ClamAV: detecção de arquivo de teste (EICAR)

Verificar o status do serviço:

```bash
systemctl status clamav-freshclam
```

Fazer download do arquivo de teste EICAR:

```bash
wget https://www.eicar.org/download/eicar.com
```

Verificar o arquivo com o ClamAV:

```bash
clamscan eicar.com
```

**Saída esperada:**

```
/root/eicar.com: OK
----------- SCAN SUMMARY -----------
Known viruses: 8696251
Scanned files: 1
Infected files: 0
```

> **Observação:** O arquivo EICAR é um padrão de teste reconhecido pela indústria. Em versões mais recentes da base de definições do ClamAV, ele pode não ser detectado como ameaça. O resultado `OK` não indica falha da ferramenta, mas sim uma atualização na classificação do arquivo de teste.

---

## Aprendizado

- Remover serviços desnecessários (FTP, Telnet, SNMP, NFS, Exim) reduz diretamente a superfície de ataque sem impacto funcional quando esses serviços não são utilizados
- O `sudo` permite delegar comandos específicos sem conceder acesso total ao root, aplicando o princípio do menor privilégio
- Restringir o `su` ao grupo `wheel` via PAM impede que qualquer usuário autenticado escalone privilégios, mesmo conhecendo a senha do root
- O ClamAV é uma solução open source viável para detecção de malware em endpoints Linux, especialmente em ambientes com arquivos originados de sistemas Windows
- Automatizar varreduras via cron garante monitoramento contínuo sem intervenção manual
