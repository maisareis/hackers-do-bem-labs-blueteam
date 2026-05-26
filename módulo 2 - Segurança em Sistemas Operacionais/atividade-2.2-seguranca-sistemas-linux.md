# Atividade 2.2 – Segurança em Sistemas Linux

## Objetivo

Implementar um sistema de gerenciamento de usuários centralizado com **OpenLDAP** no Linux, configurando autenticação segura via **SSL/TLS** e demonstrando na prática a diferença entre tráfego LDAP em texto claro e tráfego criptografado, usando `tcpdump` para captura e análise de pacotes.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### LDAP e OpenLDAP

O **LDAP** (Lightweight Directory Access Protocol) é um protocolo de acesso a diretórios que permite armazenar e consultar informações de usuários, grupos e recursos de forma centralizada. O **OpenLDAP** é a implementação open source mais difundida desse protocolo em sistemas Linux.

A estrutura de um diretório LDAP é organizada hierarquicamente por **Distinguished Names (DN)**:

```
dc=ldapserver,dc=esr,dc=rnp,dc=edu        ← Base DN (domínio)
├── ou=people                              ← Unidade organizacional de usuários
│   └── uid=myoda                          ← Entrada de usuário
└── ou=group                               ← Unidade organizacional de grupos
    └── cn=myoda                           ← Entrada de grupo
```

### Por que SSL/TLS no LDAP?

Por padrão, o LDAP transmite dados — incluindo senhas — em **texto claro** na porta 389. Qualquer host na mesma rede com acesso ao tráfego consegue capturar credenciais com ferramentas como `tcpdump`. O STARTTLS (flag `-ZZ` no cliente) negocia uma camada TLS sobre a conexão LDAP existente, tornando o payload ilegível para interceptadores.

---

## Passo a Passo

### Ambiente

| VM | IP | Usuário | Função |
|----|----|---------|--------|
| linuxserver | 192.168.98.10 | root | Servidor OpenLDAP |
| linuxclient | 192.168.98.11 | root | Cliente LDAP |

---

### Atividade 01 – Configuração do OpenLDAP

Acesso ao servidor:

```bash
ssh root@192.168.98.10
```

Verificação da configuração inicial do banco de dados SLAPD:

```bash
slapcat
```

Reconfiguração do pacote `slapd` com os parâmetros do laboratório:

```bash
dpkg-reconfigure slapd
```

Respostas fornecidas durante o assistente de configuração:

| Pergunta | Resposta |
|----------|----------|
| Omitir configuração do OpenLDAP? | **Não** |
| DNS domain name (FQDN) | `ldapserver.esr.rnp.edu` |
| Organization name | `esr.rnp.edu` |
| Senha do admin | `RnpEsr123@` |
| Remover banco ao purgar o slapd? | **Não** |
| Mover base de dados anterior? | **Sim** |

Após reconfiguração, o DN base passa de `dc=nodomain` para `dc=ldapserver,dc=esr,dc=rnp,dc=edu`.

Verificação do novo DN base:

```bash
ldapsearch -H ldapi:/// -x -LLL -s base -b "" namingContexts
```

Teste de conexão anônima:

```bash
ldapwhoami -H ldapi:/// -x
```

**Tarefa 01** — Print da saída do `ldapwhoami`:



A resposta `anonymous` confirma que o servidor está respondendo corretamente a consultas não autenticadas.

---

### Atividade 02 – Configuração da Base DN

Navegação até o diretório de arquivos do laboratório e verificação do arquivo de estrutura:

```bash
cd /curso/
cat userAndGroup.ldif
```

O arquivo define duas Unidades Organizacionais (OUs) sob o DN base:

```ldif
dn: ou=people,dc=ldapserver,dc=esr,dc=rnp,dc=edu
objectClass: organizationalUnit
ou: people

dn: ou=group,dc=ldapserver,dc=esr,dc=rnp,dc=edu
objectClass: organizationalUnit
ou: group
```

Importação das OUs para o banco SLAPD:

```bash
ldapadd -x -D cn=admin,dc=ldapserver,dc=esr,dc=rnp,dc=edu -W -f userAndGroup.ldif
```

Senha solicitada: `RnpEsr123@`

**Tarefa 02** — Print da saída do `ldapadd`:



---

### Atividade 03 – Criação de Conta de Usuário

Verificação do arquivo de definição do usuário:

```bash
cd /curso; cat newUser.ldif
```

O arquivo `newUser.ldif` define uma entrada do tipo `inetOrgPerson` + `posixAccount` + `shadowAccount`, atribuindo UID, GID, shell e diretório home ao usuário `myoda`, além de criar o grupo correspondente.

Importação do usuário:

```bash
ldapadd -x -D cn=admin,dc=ldapserver,dc=esr,dc=rnp,dc=edu -W -f newUser.ldif
```

Listagem dos objetos no diretório para confirmar a criação:

```bash
ldapsearch -x -LLL -b "dc=ldapserver,dc=esr,dc=rnp,dc=edu"
```

Alteração da senha do usuário `myoda` (o arquivo LDIF usa hash genérico; é necessário redefinir via `ldappasswd`):

```bash
ldappasswd -H ldapi:/// -x \
  -D "cn=admin,dc=ldapserver,dc=esr,dc=rnp,dc=edu" -W -S \
  "uid=myoda,ou=people,dc=ldapserver,dc=esr,dc=rnp,dc=edu"
```

Senhas solicitadas na ordem:
1. Nova senha do usuário (`RnpEsr123@`)
2. Confirmação da nova senha
3. Senha do admin LDAP (`RnpEsr123@`)

**Tarefa 03** — Print da saída do `ldappasswd`:



---

### Atividade 04 – Demonstração de Tráfego em Texto Claro

#### No linuxclient

```bash
ssh root@192.168.98.11
```

Teste de conexão remota ao servidor LDAP:

```bash
ldapwhoami -vvv -H ldap://ldapserver \
  -D "uid=myoda,ou=people,dc=ldapserver,dc=esr,dc=rnp,dc=edu" -x -W
```

#### No linuxserver (simultaneamente)

```bash
tcpdump -nnei ens5 port 389 -c 6 -tA
```

O `tcpdump` captura os 6 primeiros pacotes na porta LDAP (389) e os exibe em formato ASCII. Na saída, é possível ler em texto claro tanto o DN do usuário quanto a senha digitada — evidenciando o risco de se operar LDAP sem criptografia.

**Tarefa 04** — Print da saída do `tcpdump` no linuxserver:



---

### Atividade 05 – Configurar OpenLDAP com SSL/TLS

De volta ao **linuxserver**.

#### Criação da estrutura de diretórios para certificados

```bash
mkdir -p /etc/ssl/openldap/{private,certs,newcerts}
echo "1001" > /etc/ssl/openldap/serial
touch /etc/ssl/openldap/index.txt
```

#### Geração da chave e certificado da CA

```bash
# Gerar chave da CA (com senha)
openssl genrsa -aes256 -out /etc/ssl/openldap/private/cakey.pem 2048

# Remover senha da chave (para que o slapd possa carregá-la sem interação)
openssl rsa -in /etc/ssl/openldap/private/cakey.pem \
  -out /etc/ssl/openldap/private/cakey.pem

# Gerar certificado autoassinado da CA (válido por 10 anos)
openssl req -new -x509 -days 3650 \
  -key /etc/ssl/openldap/private/cakey.pem \
  -out /etc/ssl/openldap/certs/cacert.pem
```

Dados informados no certificado da CA:

| Campo | Valor |
|-------|-------|
| Country Name | `BR` |
| State or Province Name | `Distrito Federal` |
| Locality Name | `Brasilia` |
| Organization Name | `RNP` |
| Organizational Unit Name | `ESR` |
| Common Name | `ldapserver` |

#### Geração da chave e certificado do servidor LDAP

```bash
# Chave do servidor
openssl genrsa -aes256 -out /etc/ssl/openldap/private/ldapserver-key.key 2048

# Remover senha da chave do servidor
openssl rsa -in /etc/ssl/openldap/private/ldapserver-key.key \
  -out /etc/ssl/openldap/private/ldapserver-key.key

# CSR (Certificate Signing Request) com os mesmos dados do certificado da CA
openssl req -new \
  -key /etc/ssl/openldap/private/ldapserver-key.key \
  -out /etc/ssl/openldap/certs/ldapserver-cert.csr

# Assinar o CSR com a chave da CA
openssl ca \
  -keyfile /etc/ssl/openldap/private/cakey.pem \
  -cert /etc/ssl/openldap/certs/cacert.pem \
  -in /etc/ssl/openldap/certs/ldapserver-cert.csr \
  -out /etc/ssl/openldap/certs/ldapserver-cert.crt
```

> Responder `y` nas duas perguntas de confirmação da assinatura.

Verificação do certificado gerado:

```bash
openssl verify -CAfile /etc/ssl/openldap/certs/cacert.pem \
  /etc/ssl/openldap/certs/ldapserver-cert.crt
# Esperado: ldapserver-cert.crt: OK
```

#### Ajuste de permissões

```bash
chown -R openldap: /etc/ssl/openldap/
```

#### Importação dos certificados TLS no OpenLDAP

Verificação do arquivo de configuração:

```bash
cat /curso/ldapTLS.ldif
```

O arquivo instrui o slapd a usar os três arquivos gerados: `cacert.pem`, `ldapserver-cert.crt` e `ldapserver-key.key`.

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f /curso/ldapTLS.ldif
```

Validação da configuração:

```bash
slaptest -u
# Esperado: config file testing succeeded
```

#### Validação: tráfego agora é criptografado

No **linuxserver**:

```bash
tcpdump -nnei ens5 port 389 -tA -c 25
```

No **linuxclient**, repetindo a consulta com a flag `-ZZ` (STARTTLS obrigatório):

```bash
ldapwhoami -vvv -H ldap://ldapserver \
  -D "uid=myoda,ou=people,dc=ldapserver,dc=esr,dc=rnp,dc=edu" -x -W -ZZ
```

A flag `-ZZ` força o uso de STARTTLS; se o servidor não suportar TLS, a conexão é encerrada. A saída do `tcpdump` agora exibe apenas bytes aleatórios — nenhuma credencial legível.

**Tarefa 05** — Print da saída do `tcpdump` com tráfego criptografado:



---

## Aprendizado

- O protocolo LDAP, sem configuração adicional, transmite **tudo em texto claro** na porta 389, incluindo senhas. Qualquer host com acesso ao segmento de rede pode interceptar credenciais com um simples `tcpdump`.
- O **STARTTLS** (`-ZZ` no cliente, `ldapTLS.ldif` no servidor) negocia criptografia TLS sobre a conexão LDAP padrão, sem mudar a porta. A alternativa é LDAPS (porta 636), que encapsula toda a sessão em TLS desde o início.
- A geração de uma **CA própria** (autoassinada) é suficiente para laboratório e ambientes internos. Em produção, o certificado deve ser assinado por uma CA confiável ou pela CA interna da organização para evitar erros de validação nos clientes.
- A remoção da **senha da chave privada** (`openssl rsa ... -out ...`) é necessária para que o `slapd` carregue o certificado automaticamente na inicialização, sem interação humana. Em produção, avalie o uso de um cofre de segredos para proteger essas chaves.
- O arquivo `newUser.ldif` cria usuários com hash de senha genérico; sempre redefinir a senha via `ldappasswd` após a importação, nunca deixar o hash padrão em produção.
