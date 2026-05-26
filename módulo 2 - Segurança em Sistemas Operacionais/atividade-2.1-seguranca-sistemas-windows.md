# Atividade 2.1 – Segurança em Sistemas Windows

## Objetivo

Implementar o modelo **AGDLP** (Account → Global Group → Domain Local Group → Permission) em um ambiente Active Directory, criando usuários, grupos globais e grupos de domínio local com aninhamento, e aplicando permissões de leitura e escrita granulares a um diretório compartilhado em rede.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### AGDLP

O modelo AGDLP é uma convenção de gerenciamento de permissões no Active Directory que organiza o acesso em camadas:

| Sigla | Tipo | Função |
|-------|------|--------|
| **A** | Account (Conta de usuário) | Representa o usuário final |
| **G** | Global Group | Agrupa usuários por departamento/função |
| **DL** | Domain Local Group | Recebe as permissões no recurso |
| **P** | Permission | Permissão aplicada ao recurso (pasta, arquivo) |

A vantagem desta abordagem é separar a gestão de *quem é o usuário* (grupos globais) de *o que o recurso permite* (grupos de domínio local). Ao adicionar um novo colaborador ao financeiro, basta incluí-lo no `GG_FINANCEIRO` — as permissões já estão herdadas via aninhamento.

### Grupos Aninhados

O aninhamento consiste em adicionar um grupo como membro de outro grupo. Nesta atividade, os Global Groups são aninhados dentro dos Domain Local Groups:

```
usuario.fin.01  ──┐
                  ├──► GG_FINANCEIRO ──────► DL_FINANCEIRO_W ──► C:\SHARE\Financeiro (Escrita)
usuario.fin.02  ──┘

usuario.auditor.01 ──► GG_AUDITORIA_ESR ──► DL_FINANCEIRO_R ──► C:\SHARE\Financeiro (Leitura)
```

---

## Passo a Passo

### Ambiente

| VM | IP | Usuário | Função |
|----|----|---------|--------|
| Windows Server | 192.168.98.21 | administrator | Controlador de Domínio (esr.rnp.edu) |
| Windows Client | 192.168.98.22 | Administrator | Estação de trabalho do domínio |

---

### Atividade 1 – Criação de Usuários

Foram criados três usuários no domínio `esr.rnp.edu`, distribuídos nas OUs correspondentes ao departamento de cada um.

**Usuário Auditor 01** — criado via interface gráfica (Active Directory Users and Computers):

- OU: `esr.rnp.edu > BSB > Auditoria > Users`
- Nome: `Usuário`, Sobrenome: `Auditor 01`
- Login: `usuario.auditor.01`

**Usuário Financeiro 01** — criado via PowerShell:

```powershell
# Criação do usuário
New-ADUser -Name "Usuário Financeiro 01" `
  -GivenName "Usuário" -Surname "Financeiro 01" `
  -SamAccountName "usuario.fin.01" `
  -UserPrincipalName "usuario.fin.01@esr.rnp.edu" `
  -Path "OU=Users,OU=Financeiro,OU=BSB,OU=esr.rnp.edu,DC=esr,DC=rnp,DC=edu"

# Definindo senha
Set-ADAccountPassword -Identity "usuario.fin.01" `
  -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "RnpEsr123" -Force)

# Habilitando o usuário
Enable-ADAccount -Identity "usuario.fin.01"
```

> **Atenção:** Passar senha em texto claro no histórico do shell é um risco em produção. O comando recomendado para evitar isso é `Read-Host -AsSecureString`, que solicita a senha interativamente sem registrá-la em histórico.

**Usuário Financeiro 02** — criado com o mesmo método acima, alterando os campos para `usuario.fin.02`.

**Tarefa 01** — Verificação dos usuários criados nos últimos 3 dias:

```powershell
$Days = 03
$Time = (Get-Date).Adddays(-($Days))
Get-ADUser -Filter * -Property whenCreated | Where {$_.whenCreated -gt $Time} | ft Name, WhenCreated
```



---

### Atividade 2 – Criação e Configuração dos Grupos AGDLP

#### Grupos Globais (via interface gráfica)

Criados na OU `esr.rnp.edu > Grupos > Global Groups`:

| Grupo | Escopo | Tipo | Descrição |
|-------|--------|------|-----------|
| `GG_FINANCEIRO` | Global | Segurança | Usuários do departamento financeiro |
| `GG_AUDITORIA_ESR` | Global | Segurança | Usuários do departamento de auditoria |

> O prefixo `GG_` identifica o escopo (Global Group) e o sufixo indica o departamento — convenção que facilita a administração quando o número de grupos cresce para centenas.

#### Grupos de Domínio Local (via PowerShell)

```powershell
# DL_FINANCEIRO_R — somente leitura
$NAME = "DL_FINANCEIRO_R"
$DESCRICAO = "Grupo somente leitura no compartilhamento financeiro"
New-ADGroup -Name "$NAME" -SamAccountName "$NAME" `
  -GroupCategory Security -GroupScope DomainLocal `
  -DisplayName "$NAME" `
  -Path "OU=Domain Local Groups,OU=Grupos,OU=esr.rnp.edu,DC=esr,DC=rnp,DC=edu" `
  -Description "$DESCRICAO"

# DL_FINANCEIRO_W — escrita
$NAME = "DL_FINANCEIRO_W"
$DESCRICAO = "Grupo de escrita no compartilhamento financeiro"
New-ADGroup -Name "$NAME" -SamAccountName "$NAME" `
  -GroupCategory Security -GroupScope DomainLocal `
  -DisplayName "$NAME" `
  -Path "OU=Domain Local Groups,OU=Grupos,OU=esr.rnp.edu,DC=esr,DC=rnp,DC=edu" `
  -Description "$DESCRICAO"
```

**Tarefa 02** — Listar os grupos de Domínio Local:

```powershell
Get-ADGroup -Filter {GroupScope -eq "DomainLocal"} -SearchBase "OU=esr.rnp.edu,DC=esr,DC=rnp,DC=edu"
```



#### Adicionando Usuários aos Grupos Globais

```powershell
# usuario.fin.01 → GG_FINANCEIRO
$User = Get-ADUser -Identity "usuario.fin.01"
$Group = Get-ADGroup -Identity "GG_FINANCEIRO"
Add-ADGroupMember -Identity $Group -Members $User

# usuario.fin.02 → GG_FINANCEIRO
$User = Get-ADUser -Identity "usuario.fin.02"
Add-ADGroupMember -Identity $Group -Members $User

# usuario.auditor.01 → GG_AUDITORIA_ESR
$User = Get-ADUser -Identity "usuario.auditor.01"
$Group = Get-ADGroup -Identity "GG_AUDITORIA_ESR"
Add-ADGroupMember -Identity $Group -Members $User
```

#### Criando o Aninhamento

```powershell
# GG_FINANCEIRO → DL_FINANCEIRO_W (escrita)
$GroupMember = Get-ADGroup -Identity "GG_FINANCEIRO"
$Group = Get-ADGroup -Identity "DL_FINANCEIRO_W"
Add-ADGroupMember -Identity $Group -Members $GroupMember

# GG_AUDITORIA_ESR → DL_FINANCEIRO_R (leitura)
$GroupMember = Get-ADGroup -Identity "GG_AUDITORIA_ESR"
$Group = Get-ADGroup -Identity "DL_FINANCEIRO_R"
Add-ADGroupMember -Identity $Group -Members $GroupMember
```

---

### Atividade 3 – Criação do Diretório Compartilhado

No Windows Server, via Windows Explorer:

1. Criada a pasta `C:\SHARE` para centralizar os compartilhamentos da empresa
2. Criada a subpasta `C:\SHARE\Financeiro`
3. Compartilhamento configurado em **Propriedades > Compartilhamento > Compartilhamento Avançado**, com permissão total para `Everyone` no nível de compartilhamento (controle granular feito pelas ACLs NTFS)
4. Na aba **Segurança > Avançado**, a herança foi desabilitada e convertida em permissões explícitas, mantendo apenas `SYSTEM` e `Administrators` com controle total

**Tarefa 03** — Verificar a ACL do diretório:

```powershell
Get-Acl -Path c:\share\financeiro
```



---

### Atividade 4 – Aplicando Permissões aos Grupos

Em **Propriedades de Financeiro > Segurança > Editar**, os grupos `DL_FINANCEIRO_R` e `DL_FINANCEIRO_W` foram adicionados com as seguintes permissões NTFS:

| Grupo | Permissões permitidas |
|-------|-----------------------|
| `DL_FINANCEIRO_R` | Ler & Executar, Listar conteúdo da pasta, Leitura |
| `DL_FINANCEIRO_W` | Modificar, Ler & Executar, Listar conteúdo da pasta, Leitura, Escrever |

**Tarefa 04** — Print da janela de propriedades com `DL_FINANCEIRO_R` selecionado:



---

### Atividade 5 – Validando o Aninhamento

No Windows Client (via Remote Desktop a partir do Windows Server):

1. Configurado o DNS primário do cliente para apontar para o Windows Server:

```powershell
# Identificar o índice da interface de rede
Get-NetIPInterface -AddressFamily IPv4 | Where-Object { $_.InterfaceDescription -notmatch "Loopback" } | Select-Object InterfaceIndex, InterfaceAlias, InterfaceDescription | Format-Table -AutoSize

# Configurar DNS primário e secundário (ajustar InterfaceIndex conforme o ambiente)
Set-DnsClientServerAddress -InterfaceIndex 11 -ServerAddresses "192.168.98.21","192.168.98.2"

# Forçar atualização de GPO
gpupdate.exe /force

# Permitir login remoto para usuários do domínio
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "Domain Users"
```

2. Login no Windows Client com `usuario.fin.01` → acesso a `\\WINDOWSSERVER\Financeiro` → criação de pasta e arquivo com sucesso (permissão de escrita herdada via `GG_FINANCEIRO > DL_FINANCEIRO_W`)

3. Disconnect e novo login com `usuario.auditor.01` → acesso ao mesmo compartilhamento → tentativa de remover o arquivo criado pelo usuário anterior **resulta em acesso negado** (permissão apenas de leitura via `GG_AUDITORIA_ESR > DL_FINANCEIRO_R`)

**Tarefa 05** — Print do erro ao tentar remover os arquivos:



---

## Aprendizado

- O modelo **AGDLP** desacopla a gestão de usuários da atribuição de permissões: adicionar um novo colaborador ao departamento financeiro exige apenas incluí-lo no grupo global correspondente, sem tocar nas ACLs do recurso.
- A **desabilitação de herança** nas ACLs NTFS é uma prática de hardening importante — evita que permissões de diretórios pai vazem para subpastas sensíveis.
- O PowerShell oferece vantagens sobre a GUI para operações em escala, mas exige cuidado com o histórico de comandos: senhas em texto claro ficam registradas e podem ser recuperadas em caso de comprometimento da máquina. Prefira `Read-Host -AsSecureString` em ambientes reais.
- A validação prática (login com cada usuário e tentativa de operações além da permissão atribuída) é o passo mais importante: confirma que a configuração funciona como esperado e não apenas que os objetos foram criados.
