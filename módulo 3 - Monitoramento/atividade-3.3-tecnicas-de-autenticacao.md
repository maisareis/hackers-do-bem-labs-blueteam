# Atividade 3.3 – Técnicas de Autenticação

## Objetivo

Configurar o **Google Authenticator** como segundo fator de autenticação (2FA) para acesso remoto via SSH em um servidor Linux Debian, integrando o módulo **PAM** ao serviço SSH para exigir um código TOTP gerado pelo aplicativo além da senha tradicional.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### Autenticação Multifator (MFA)

Autenticação multifator é um mecanismo de segurança que exige que o usuário comprove sua identidade por meio de dois ou mais fatores independentes antes de obter acesso a um sistema. Os fatores se enquadram em três categorias:

| Fator | Descrição | Exemplo |
|-------|-----------|---------|
| **Conhecimento** | Algo que o usuário sabe | Senha, PIN |
| **Posse** | Algo que o usuário tem | Smartphone com app autenticador |
| **Inerência** | Algo que o usuário é | Biometria |

O 2FA (Two-Factor Authentication) combina dois desses fatores — tipicamente senha + código temporário — tornando o acesso significativamente mais resistente a ataques de força bruta, phishing e credential stuffing, já que o comprometimento da senha isoladamente não é suficiente para autenticar.

### TOTP (Time-Based One-Time Password)

O TOTP é um algoritmo definido na RFC 6238 que gera códigos numéricos temporários com base em dois elementos: uma chave secreta compartilhada entre o servidor e o aplicativo, e o timestamp atual. O código é válido por um curto intervalo de tempo (tipicamente 30 segundos) e se torna inválido após o uso, eliminando o risco de reutilização de credenciais interceptadas.

### Google Authenticator e PAM

O **Google Authenticator** é um aplicativo mobile que implementa o padrão TOTP. No lado do servidor, a integração é feita por meio do módulo `libpam-google-authenticator`, que conecta o mecanismo de autenticação PAM ao arquivo de configuração gerado por usuário (`~/.google_authenticator`). O PAM (Pluggable Authentication Module) é uma camada de abstração do Linux que permite adicionar, remover ou encadear mecanismos de autenticação sem modificar as aplicações — neste caso, o SSH.

---

## Passo a Passo

### Ambiente

| VM | IP | Usuário | Função |
|----|----|---------|--------|
| linuxserver | [IP_LINUXSERVER] | root / aluno-2fa | Servidor SSH com 2FA configurado |
| linuxclient | [IP_LINUXCLIENT] | aluno | Origem da conexão SSH |

---

### Atividade 1 – Criando o usuário para o Google Authenticator

Conectado ao linuxserver como root, criação do usuário dedicado ao 2FA:

```bash
adduser aluno-2fa
```

Campos preenchidos durante a criação:

| Campo | Valor |
|-------|-------|
| New password | `RnpEsr123` |
| Retype new password | `RnpEsr123` |
| Full Name, Room Number, Work Phone, Home Phone, Other | *(Enter — deixado em branco)* |
| Is the information correct? | `Y` |

Adição do usuário aos grupos `sudo` e `wheel`:

```bash
usermod -aG sudo,wheel aluno-2fa
```

**Tarefa 01** — Print da saída do comando `adduser aluno-2fa` confirmando a criação do usuário e atualização da senha.

---

### Atividade 2 – Verificando a instalação do Google Authenticator

No linuxserver, verificação do pacote `libpam-google-authenticator` (já instalado previamente no ambiente do laboratório):

```bash
which google-authenticator
```

**Tarefa 02** — Print da saída do comando confirmando o caminho `/usr/bin/google-authenticator`.

---

### Atividade 3 – Configurando o Google Authenticator no servidor

Antes de iniciar a configuração no servidor, o aplicativo **Google Authenticator** deve estar instalado no smartphone (Android ou iOS).

Alternância para o usuário `aluno-2fa` e navegação ao diretório home:

```bash
su aluno-2fa
cd /home/aluno-2fa/
```

Execução do comando de configuração:

```bash
google-authenticator
```

Respostas fornecidas durante o assistente de configuração:

| Pergunta | Resposta |
|----------|----------|
| Do you want authentication tokens to be time-based (y/n) | `y` |

Ao responder `y`, um QR Code é exibido no terminal. O QR Code deve ser lido com o aplicativo Google Authenticator no smartphone para registrar a conta.

**Tarefa 03** — Print da tela do terminal exibindo o QR Code gerado pelo comando `google-authenticator`.

---

### Atividade 4 – Configurando o Google Authenticator no smartphone

Com o aplicativo aberto no smartphone:

1. Tocar em **Adicionar um código**
2. Selecionar **Ler código QR**
3. Apontar a câmera para o QR Code exibido no terminal do linuxserver
4. Após o escaneamento, o aplicativo passa a exibir um código TOTP de 6 dígitos rotativo para a conta `aluno-2fa@linuxserver`

O código gerado pelo aplicativo deve ser inserido no prompt do terminal:

```
Enter code from app (-1 to skip): <código do smartphone>
```

Após a confirmação do código, são gerados 5 **códigos de emergência** — utilizáveis caso o smartphone esteja indisponível. Respostas fornecidas às perguntas subsequentes:

| Pergunta | Resposta |
|----------|----------|
| Do you want me to update your `~/.google_authenticator` file? | `y` |
| Do you want to disallow multiple uses of the same authentication token? | `y` |
| Do you want to increase the time skew window to 17 codes? | `y` |
| Do you want to enable rate-limiting? | `y` |

**Tarefa 04** — Print da tela do terminal exibindo os 5 códigos de emergência gerados após a confirmação do OTP.

---

### Atividade 5 – Configurando o SSH para exigir o código OTP

Logado como root no linuxserver, adição da diretiva ao módulo PAM do SSH:

```bash
echo "auth required pam_google_authenticator.so" >> /etc/pam.d/sshd
```

Habilitação da autenticação interativa por teclado no arquivo de configuração do SSH:

```bash
sed -i "s/KbdInteractiveAuthentication no/KbdInteractiveAuthentication yes/" /etc/ssh/sshd_config
```

Reinicialização do serviço SSH para aplicar as alterações:

```bash
systemctl restart ssh.service
```

---

### Atividade 6 – Acessando o servidor via SSH com 2FA

A partir do linuxclient, conexão SSH com o usuário `aluno-2fa`:

```bash
ssh aluno-2fa@[IP_LINUXSERVER]
```

Durante o processo de autenticação, são solicitados dois fatores em sequência:

```
Password: RnpEsr123
Verification code: <código exibido no Google Authenticator>
```

Após a autenticação bem-sucedida, o acesso ao linuxserver é concedido.

**Tarefa 06** — Print da tela exibindo o prompt de `Verification code` e o login bem-sucedido no linuxserver.

Ao final do laboratório, reversão das configurações para o estado padrão:

```bash
sed -i "s/KbdInteractiveAuthentication yes/KbdInteractiveAuthentication no/" /etc/ssh/sshd_config
sed -i "s/auth required pam_google_authenticator.so//" /etc/pam.d/sshd
systemctl restart sshd.service
```

---

## Aprendizado

- A autenticação por senha isolada é um ponto único de falha: uma vez comprometida — por phishing, força bruta ou vazamento — o acesso ao sistema é imediato. O 2FA com TOTP adiciona uma segunda camada que invalida credenciais roubadas, pois o código é válido por apenas 30 segundos e vinculado a um dispositivo físico.
- O TOTP funciona sem conexão de rede no smartphone: o código é calculado localmente com base na chave secreta e no tempo atual. Isso elimina dependências de infraestrutura como SMS ou e-mail para entrega do segundo fator, tornando a solução mais robusta e menos suscetível a ataques de SIM swapping.
- A integração via PAM é o que torna a configuração flexível: o módulo `pam_google_authenticator.so` pode ser adicionado a qualquer serviço que use PAM (SSH, sudo, login local) sem modificar o código das aplicações. A diretiva `required` no PAM garante que a ausência ou falha do 2FA bloqueia o acesso independentemente dos outros fatores.
- A opção `KbdInteractiveAuthentication yes` no sshd_config é um pré-requisito frequentemente esquecido: sem ela, o SSH não exibe o prompt de código OTP, pois a autenticação interativa por teclado fica desabilitada por padrão em muitas distribuições.
- Os códigos de emergência são um mecanismo de contingência importante em ambientes de produção: sem eles, a perda ou troca do smartphone pode bloquear permanentemente o acesso ao servidor se não houver outro método de autenticação disponível.
