# Atividade 2.3 – Bloqueando Ataques de Bruteforce com Fail2ban

## Objetivo

Configurar o **Fail2ban** no Linux para detectar e bloquear automaticamente ataques de força bruta contra o serviço SSH, demonstrando na prática a diferença entre um ambiente sem proteção (onde o Hydra consegue descobrir a senha) e um ambiente protegido (onde o IP atacante é banido após tentativas excessivas).

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### Fail2ban

O **Fail2ban** monitora arquivos de log em busca de padrões de falha de autenticação (definidos em **filtros**) e, ao atingir o limiar configurado, executa uma ação de bloqueio — normalmente uma regra de `iptables` que descarta pacotes do IP ofensor por um período determinado.

Os principais componentes de configuração são:

| Componente | Função |
|------------|--------|
| **jail** | Define o serviço monitorado e os parâmetros de bloqueio |
| **filter** | Expressão regular que identifica falhas no log |
| **action** | O que fazer ao banir (ex: `iptables-multiport`) |
| **findtime** | Janela de tempo para contagem de falhas |
| **maxretry** | Número máximo de falhas antes do banimento |
| **bantime** | Duração do bloqueio |

### Hydra

O **Hydra** é uma ferramenta de ataque de força bruta que testa listas de credenciais contra protocolos de autenticação (SSH, FTP, HTTP, etc.). Nesta atividade é usado para simular um atacante externo, validando tanto a vulnerabilidade (sem Fail2ban) quanto a eficácia da proteção (com Fail2ban ativo).

---

## Passo a Passo

### Ambiente

| VM | IP | Usuário | Função |
|----|----|---------|--------|
| linuxserver | 192.168.98.10 | root | Servidor SSH + Fail2ban |
| linuxclient | 192.168.98.11 | root | Máquina atacante (Hydra) |

---

### Atividade 01 – Iniciando o Fail2ban

Acesso ao servidor:

```bash
ssh root@192.168.98.10
```

Verificação do status inicial (serviço em estado `failed`):

```bash
systemctl status fail2ban.service
```

Parada do serviço para reconfiguração:

```bash
systemctl stop fail2ban.service
```

Criação do arquivo de configuração base para o jail do SSH:

```bash
echo -e "[sshd]\nenabled = true\nbackend = systemd" > \
  /etc/fail2ban/jail.d/defaults-debian.conf
```

Verificação do arquivo gerado:

```bash
cat /etc/fail2ban/jail.d/defaults-debian.conf
```

**Tarefa 01** — Print da saída do `cat` acima:



Inicialização do serviço:

```bash
systemctl start fail2ban.service
systemctl status fail2ban.service
```

A linha `Active: active (running)` confirma que o serviço subiu corretamente.

---

### Atividade 02 – Configuração Inicial

Listagem do diretório de configuração do Fail2ban:

```bash
ls -lh /etc/fail2ban
```

Cópia do arquivo padrão para criar a configuração específica do SSH:

```bash
cp /etc/fail2ban/jail.d/defaults-debian.conf /etc/fail2ban/jail.d/sshd.local
```

Comentando o arquivo original para evitar conflito (o `sshd.local` será o arquivo ativo):

```bash
sed -e 's/^/#/' -i /etc/fail2ban/jail.d/defaults-debian.conf
```

Parada do serviço para executar o ataque de demonstração sem interferência:

```bash
systemctl stop fail2ban
systemctl status fail2ban
```

**Tarefa 02** — Print da saída do `systemctl status fail2ban` mostrando `inactive (dead)`:



---

### Atividade 03 – Ataque de Força Bruta sem Proteção

#### No linuxserver

Confirmação de que autenticação por senha está habilitada no SSH:

```bash
cat /etc/ssh/sshd_config | grep -Ei "^PasswordAuthentication"
# Esperado: PasswordAuthentication yes
```

#### No linuxclient

```bash
ssh root@192.168.98.11
```

Verificação da versão do Hydra:

```bash
hydra | head -n 2
```

Execução do ataque de força bruta contra o SSH do linuxserver:

```bash
hydra -l aluno -P /curso/wordlist.txt 192.168.98.10 ssh -f
```

| Opção | Descrição |
|-------|-----------|
| `-l aluno` | Usuário alvo do ataque |
| `-P /curso/wordlist.txt` | Wordlist de senhas a testar |
| `ssh` | Protocolo alvo |
| `-f` | Encerra após encontrar a primeira senha válida |

Com o Fail2ban desativado, o Hydra consegue iterar a wordlist sem interrupção até encontrar a senha correta (`rnpesr`).

**Tarefa 03** — Print da saída do Hydra com a senha descoberta:



---

### Atividade 04 – Configurando o Fail2ban para Bloquear Bruteforce

De volta ao **linuxserver**.

Inicialização do Fail2ban:

```bash
systemctl start fail2ban
```

Verificação de jails ativas (nenhuma ainda, pois `sshd.local` ainda não tem os parâmetros completos):

```bash
fail2ban-client status
```

Adição dos parâmetros de bloqueio ao jail do SSH:

```bash
echo -e "findtime = 5m\nbantime = 600m\nmaxretry = 4\nbanaction = iptables-multiport" \
  >> /etc/fail2ban/jail.d/sshd.local
```

Verificação do arquivo final:

```bash
cat /etc/fail2ban/jail.d/sshd.local
```

Configuração resultante:

```ini
[sshd]
enabled = true
backend = systemd
findtime = 5m
bantime = 600m
maxretry = 4
banaction = iptables-multiport
```

| Parâmetro | Valor | Significado |
|-----------|-------|-------------|
| `findtime` | `5m` | Janela de 5 minutos para contar falhas |
| `bantime` | `600m` | IP banido por 5 horas |
| `maxretry` | `4` | 4 tentativas falhas disparam o ban |
| `banaction` | `iptables-multiport` | Bloqueio via regra de iptables |

Limpeza do log de autenticação para evitar que o Fail2ban bana o linuxclient imediatamente (por conta das tentativas do ataque anterior):

```bash
echo > /var/log/auth.log
```

Reinicialização do serviço com a nova configuração:

```bash
systemctl start fail2ban
```

Confirmação de que o jail SSH está ativo:

```bash
fail2ban-client status
```

**Tarefa 04** — Print da saída do `fail2ban-client status` mostrando `Jail list: sshd`:



---

### Atividade 05 – Bruteforce com Fail2ban Ativo

Dois terminais simultâneos são necessários.

#### Terminal 01 — linuxserver

Desbloqueio preventivo do IP do linuxclient (caso já esteja na lista de banidos):

```bash
fail2ban-client set sshd unbanip 192.168.98.11
```

Monitoramento em tempo real do log do Fail2ban:

```bash
tail -f /var/log/fail2ban.log
```

#### Terminal 02 — linuxclient

Repetição do ataque:

```bash
hydra -l aluno -P /curso/wordlist.txt 192.168.98.10 ssh -f
```

Desta vez, após 4 tentativas falhas dentro de 5 minutos, o Fail2ban detecta o padrão no `auth.log`, cria uma regra de `iptables` bloqueando o IP `192.168.98.11` e registra o evento no `fail2ban.log`. O Hydra começa a receber timeouts e a progressão do ataque para.

Para encerrar o monitoramento: `Ctrl + C`

Desbloqueio manual do IP após o teste:

```bash
fail2ban-client set sshd unbanip 192.168.98.11
```

**Tarefa 05** — Print do Terminal 01 mostrando o IP `192.168.98.11` sendo banido no log:



---

### Comandos Úteis do Fail2ban

```bash
# Listar IPs atualmente banidos
fail2ban-client get sshd banned

# Verificar status detalhado de um jail
fail2ban-client status sshd

# Desbanir um IP manualmente
fail2ban-client set sshd unbanip <IP>

# Desativar o Fail2ban ao fim do laboratório
systemctl stop fail2ban
systemctl disable fail2ban
```

---

## Aprendizado

- O SSH com autenticação por senha e sem proteção contra força bruta é um alvo trivial: o Hydra encontrou a senha em menos de 2 minutos iterando uma wordlist. Isso reforça a importância de usar **autenticação por chave SSH** em produção e, quando senha for necessária, combinar com ferramentas como o Fail2ban.
- A **limpeza do `auth.log`** antes de reativar o Fail2ban é um detalhe operacional importante: o serviço lê o histórico do arquivo ao iniciar e pode banir IPs legítimos com base em tentativas antigas.
- O parâmetro **`backend = systemd`** instrui o Fail2ban a ler o journal do systemd em vez do arquivo de log tradicional — necessário em distribuições que não gravam o `auth.log` por padrão.
- **`bantime` negativo** (`bantime = -1`) configura banimento permanente — útil para IPs de scanners conhecidos, mas deve ser usado com cuidado para não bloquear IPs legítimos que mudam com frequência (ex: ISPs com IP dinâmico).
- O Fail2ban é uma **camada adicional de defesa**, não um substituto para boas práticas: desativar login root via SSH (`PermitRootLogin no`), mudar a porta padrão e exigir autenticação por chave continuam sendo medidas fundamentais de hardening.
