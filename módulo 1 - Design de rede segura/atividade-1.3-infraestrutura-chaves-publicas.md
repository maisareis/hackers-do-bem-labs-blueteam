# Atividade 1.3 – Infraestrutura de Chaves Públicas (PKI)

## Objetivo

Compreender e aplicar os principais conceitos de PKI: salt no armazenamento de senhas, criptografia simétrica com GPG, geração de certificados com OpenSSL e autenticação SSH baseada em certificados.

## Aviso Legal

> Atividade realizada em ambiente controlado e autorizado para fins educacionais. IPs, usuários e senhas utilizados são exclusivos do ambiente de laboratório e não estão expostos neste documento.

---

## Conceito

A **Infraestrutura de Chaves Públicas (PKI)** é um conjunto de políticas, processos e tecnologias que gerencia chaves criptográficas e certificados digitais. Ela viabiliza autenticação, confidencialidade e integridade em comunicações seguras.

---

## Passo a Passo

### Atividade 1 – Como funciona o salt no Linux

O **salt** é um valor aleatório adicionado à senha antes do hash, tornando hashes idênticos impossíveis mesmo para senhas iguais.

Criar usuário e verificar o salt no `/etc/shadow`:

```bash
sudo -i
useradd nome_usuario
passwd nome_usuario
grep nome_usuario /etc/shadow
```

**Saída esperada (estrutura):**

```
nome_usuario:$y$j9T$<salt>$<hash>:19920:0:99999:7:::
```

Os campos separados por `$` indicam:
- Algoritmo de hash (`y` = yescrypt)
- Parâmetros do algoritmo
- O salt (valor aleatório, diferente a cada execução)
- O hash resultante

---

### Atividade 2 – Criptografia simétrica com GPG

Verificar cifras disponíveis:

```bash
gpg --version | grep Cipher -A1
```

Criar arquivo, criptografar com AES256 e transferir para outro host:

```bash
echo 'Hacker do bem - blueteam' > aluno.txt
sudo gpg --symmetric --cipher-algo AES256 aluno.txt
# Insira e confirme a passphrase quando solicitado
```

Comparar o arquivo original e o criptografado:

```bash
cat aluno.txt          # conteúdo legível
cat aluno.txt.gpg      # conteúdo cifrado (ilegível)
```

Copiar o arquivo cifrado para outro host via SCP:

```bash
scp aluno.txt.gpg usuario@IP_DESTINO:~
```

Descriptografar no host de destino:

```bash
sudo gpg -o aluno.txt.out -d aluno.txt.gpg
cat aluno.txt.out      # deve exibir o conteúdo original
```

---

### Atividade 3 – Geração de chaves e certificados com OpenSSL

Gerar par de chaves RSA:

```bash
openssl genpkey -algorithm RSA -out private.key
```

Visualizar a chave privada:

```bash
openssl rsa -in private.key -text -noout | more
# Pressione 'q' para sair
```

Gerar certificado autoassinado (X.509):

```bash
openssl req -new -x509 -key private.key -out certificate.crt
# Preencher os campos: país, estado, cidade, organização, CN, e-mail
```

Visualizar o certificado:

```bash
openssl x509 -in certificate.crt -text -noout
```

A saída exibe: algoritmo de assinatura, emissor, validade, chave pública e extensões X.509v3.

---

### Atividade 4 – Autenticação SSH baseada em certificados

**Por que certificados SSH são superiores à autenticação por senha ou chave simples?**

| Método | Problemas |
|--------|-----------|
| Senha | Força bruta, eavesdropping, erro de digitação |
| Chave SSH (copiar/colar) | Sem verificação de identidade do servidor, difícil escalar |
| Certificado SSH | Validade configurável, CA centralizada, rastreável em logs |

**Criar a CA (Certificate Authority):**

```bash
ssh-keygen -t rsa -f ca_key
# Insira uma passphrase segura
```

Copiar a chave pública da CA para o diretório SSH:

```bash
sudo cp ca_key.pub /etc/ssh/
```

Configurar o servidor SSH para confiar na CA (no host servidor):

```bash
sudo bash -c 'echo "TrustedUserCAKeys /etc/ssh/ca_key.pub" >> /etc/ssh/sshd_config'
sudo service sshd restart
sudo systemctl status sshd
```

Assinar a chave pública do host com a CA (validade de 52 semanas):

```bash
sudo ssh-keygen -s ca_key -I Host -V +52w -h /etc/ssh/ssh_host_rsa_key.pub
sudo bash -c 'echo "HostCertificate /etc/ssh/ssh_host_rsa_key-cert.pub" >> /etc/ssh/sshd_config'
sudo service sshd restart
```

---

### Atividade 5 – Configurar o cliente para confiar na CA

No host cliente, criar o arquivo `known_hosts` com a chave pública da CA:

```bash
nano ~/.ssh/known_hosts
```

Adicionar a linha:

```
@cert-authority nome_servidor,nome_cliente ssh-rsa <conteúdo de ca_key.pub>
```

Gerar par de chaves para o usuário cliente:

```bash
ssh-keygen -t rsa -b 2048
```

Copiar a chave pública para a CA assinar:

```bash
scp .ssh/id_rsa.pub usuario@IP_CA:/home/usuario
```

Assinar a chave do cliente na CA:

```bash
sudo ssh-keygen -s ca_key -I Client -n root,host -V +52w id_rsa.pub
```

Copiar o certificado assinado de volta para o cliente:

```bash
scp id_rsa-cert.pub usuario@IP_CLIENTE:/home/usuario/.ssh/
```

Instalar a chave pública no servidor e conectar:

```bash
ssh-copy-id usuario@IP_SERVIDOR
ssh usuario@IP_SERVIDOR
# Insira a passphrase da chave quando solicitado
```

---

## Aprendizado

- O salt garante que dois usuários com a mesma senha tenham hashes diferentes no `/etc/shadow`, frustrando ataques de dicionário e rainbow tables
- O GPG permite criptografia simétrica para transferência segura de arquivos sem necessidade de PKI completa
- O OpenSSL gera chaves e certificados autoassinados, úteis em ambientes internos onde uma CA pública não é necessária
- Certificados SSH centralizam a confiança em uma CA, simplificam o gerenciamento em escala e permitem validade configurável com rastreabilidade em logs
