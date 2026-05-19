# Atividade 1.4 – Gerenciamento de Configuração e Infraestrutura como Código

## Objetivo

Utilizar o Ansible para gerenciar configurações de forma centralizada e automatizada em hosts Linux e Windows, e praticar o fluxo básico do Git/GitHub para controle de versão.

## Aviso Legal

> Atividade realizada em ambiente controlado e autorizado para fins educacionais. IPs, usuários e senhas utilizados são exclusivos do ambiente de laboratório e não estão expostos neste documento.

---

## Conceito

**Gerenciamento de configuração** permite definir um estado desejado para sistemas e aplicar automaticamente as alterações necessárias quando esse estado não é atingido. O **Ansible** opera sem agentes, conectando-se via SSH ou PowerShell, e é **idempotente**: não aplica mudanças desnecessárias em execuções repetidas.

---

## Passo a Passo

### Atividade 1 – Preparando o ambiente

Gerar chave SSH dedicada ao Ansible (sem passphrase):

```bash
ssh-keygen -t rsa -C "Ansible"
# Salvar em: /home/aluno/.ssh/ansible
```

Copiar a chave pública para o host Linux alvo:

```bash
ssh-copy-id -i /home/aluno/.ssh/ansible.pub usuario@IP_HOST_LINUX
```

Testar acesso SSH aos hosts Windows:

```bash
ssh administrador@IP_HOST_WINDOWS
# Responder 'yes' na primeira conexão
exit
```

---

### Atividade 2 – Configurando o Ansible

Criar o diretório de configuração e gerar o arquivo padrão:

```bash
sudo mkdir /etc/ansible
sudo ansible-config init --disabled | sudo tee /etc/ansible/ansible.cfg
sudo sed -i '/^\[defaults]/a interpreter_python = auto_silent' /etc/ansible/ansible.cfg
```

Criar e editar o inventário de hosts:

```bash
sudo touch /etc/ansible/hosts
sudo nano /etc/ansible/hosts
```

Estrutura do arquivo `hosts`:

```ini
[servidoreslinux]
nome_host_linux

[servidoreslinux:vars]
ansible_ssh_private_key_file=/home/aluno/.ssh/ansible

[servidoreswindows]
windowsserver ansible_host=IP_WINDOWS_SERVER
windowsclient ansible_host=IP_WINDOWS_CLIENT

[servidoreswindows:vars]
ansible_connection=ssh
ansible_shell_type=cmd
ansible_ssh_user=administrador
```

Testar conectividade com hosts Linux:

```bash
ansible servidoreslinux -m shell -a 'hostname --fqdn; whoami'
```

Testar conectividade com hosts Windows:

```bash
ansible servidoreswindows -m win_ping -k
```

**Saída esperada (Windows):**

```json
windowsserver | SUCCESS => { "changed": false, "ping": "pong" }
windowsclient | SUCCESS => { "changed": false, "ping": "pong" }
```

---

### Atividade 3 – Utilizando roles no Ansible

Criar o diretório de roles e inicializar a primeira role:

```bash
sudo mkdir /etc/ansible/roles
sudo ansible-galaxy init --init-path /etc/ansible/roles/ linux_whoami
```

O Ansible cria automaticamente a estrutura:

```
linux_whoami/
├── defaults/
├── files/
├── handlers/
├── meta/
├── tasks/
├── templates/
├── tests/
└── vars/
```

Editar o arquivo de tarefas (`tasks/main.yml`):

```yaml
---
- name: Who am I
  shell: whoami
  register: whoami_content

- name: lendo conteudo
  debug:
    msg: "{{ whoami_content.stdout }}"
```

Editar o arquivo de teste (`tests/test.yml`):

```yaml
---
- hosts: localhost
  remote_user: aluno
  roles:
    - linux_whoami
```

Executar o playbook:

```bash
ansible-playbook /etc/ansible/roles/linux_whoami/tests/test.yml
```

**Saída esperada:**

```
TASK [linux_whoami : lendo conteudo]
ok: [localhost] => { "msg": "aluno" }
PLAY RECAP: localhost : ok=3 changed=1 unreachable=0 failed=0
```

Criar e testar role para ping em hosts Windows:

```yaml
# tasks/main.yml
---
- name: windows ping test
  win_ping:
  register: win_ping_content

- name: resposta do teste
  debug:
    msg: "{{ win_ping_content }}"
```

```bash
ansible-playbook /etc/ansible/roles/windows_ping/tests/test.yml -k
```

Criar role para listar arquivos em hosts Linux (`linux_lista_arquivos`):

```yaml
---
- name: Lista arquivos na pasta Downloads
  command: ls -l --time=ctime ~/Downloads
  register: downloads_files

- name: Lista arquivos na pasta Temp
  command: ls -l --time=ctime /tmp
  register: temp_files

- name: Exibe arquivos na pasta Downloads
  debug:
    var: downloads_files.stdout_lines

- name: Exibe arquivos na pasta Temp
  debug:
    var: temp_files.stdout_lines
```

Criar role equivalente para Windows (`windows_lista_arquivos`) que verifica pastas `Downloads` e `%TEMP%` via `win_command`.

---

### Atividade 4 – Configurando o GitHub

1. Criar conta em [https://github.com](https://github.com)
2. Criar novo repositório público com README inicial
3. Registrar a URL do repositório para uso no próximo passo

---

### Atividade 5 – Clonando e versionando um repositório

```bash
# Instalar o Git (se necessário)
# Clonar o repositório
git clone URL_DO_REPOSITORIO

# Navegar até o diretório
cd nome_do_repositorio

# Verificar status
git status

# Adicionar alterações
git add .

# Criar commit
git commit -m "descrição das alterações"

# Enviar ao repositório remoto
git push origin main
```

---

## Aprendizado

- O Ansible elimina a necessidade de configurar manualmente cada máquina: o estado desejado é declarado uma vez e aplicado em escala
- A idempotência garante que rodar o mesmo playbook múltiplas vezes não causa efeitos colaterais inesperados
- Roles organizam playbooks em estruturas reutilizáveis e fáceis de manter
- O Git + GitHub permitem rastrear todas as alterações nos scripts de automação, criando histórico auditável das mudanças de infraestrutura
