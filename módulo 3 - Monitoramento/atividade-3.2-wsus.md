# Atividade 3.2 – Windows Server Update Services (WSUS)

## Objetivo

Instalar, configurar e gerenciar o **WSUS (Windows Server Update Services)** em um ambiente Windows Server, centralizando a distribuição de atualizações e patches para produtos Microsoft, criando grupos de computadores para homologação e configurando aprovações automáticas para garantir um processo de atualização seguro e controlado.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### WSUS (Windows Server Update Services)

O WSUS é uma solução Microsoft que permite aos administradores gerenciar centralmente a distribuição de atualizações lançadas pelo Microsoft Update para computadores em uma rede corporativa. Em vez de cada estação buscar atualizações diretamente na internet, o WSUS atua como repositório interno, trazendo benefícios de economia de banda, controle de versão e rastreabilidade.

| Componente | Função |
|------------|--------|
| **WSUS Server** | Repositório central que sincroniza atualizações com o Microsoft Update |
| **WID (Windows Internal Database)** | Banco de dados interno que armazena metadados e configurações do WSUS |
| **GPO (Group Policy Object)** | Mecanismo para distribuir configurações do WSUS aos clientes do domínio |
| **Grupos de computadores** | Segmentação de máquinas para controle granular de quais atualizações cada grupo recebe |
| **Aprovações automáticas** | Regras que definem quais atualizações são aprovadas automaticamente para quais grupos |

### Por que usar grupos de homologação?

Antes de aplicar uma atualização em todo o parque de máquinas, é recomendável testá-la em um grupo reduzido de máquinas de homologação. Caso a atualização cause instabilidade, o impacto fica contido ao grupo de teste, evitando indisponibilidade generalizada no ambiente de produção.

### GPO e WSUS

A GPO do WSUS é responsável por:
- Definir o servidor WSUS interno como fonte de atualizações (substituindo o Microsoft Update direto)
- Configurar o comportamento de atualização automática nos clientes (notificar, baixar, instalar)
- Garantir que todos os computadores do domínio sigam a política de atualização centralizada

---

## Passo a Passo

### Ambiente

| VM | Usuário | Função |
|----|---------|--------|
| windowsserver | Administrator | Servidor WSUS |
| windowsclient | Administrator | Cliente que recebe atualizações via WSUS |

---

### Atividade 1 – Instalação do WSUS

Acesso ao windowsserver como Administrator via RDP.

Criação do diretório de armazenamento de atualizações no Windows Explorer: `C:\WSUS`

Instalação via Server Manager:

**Server Manager → Manage → Add Roles and Features → Role-based or feature-based installation → Windows Server Update Services → Add Features → Next**

Na etapa **Content location selection**, marcar a caixa e informar o caminho `C:\WSUS`.

Clicar em **Install** e aguardar a conclusão (aproximadamente 3 minutos).

Componentes instalados:

| Componente | Descrição |
|------------|-----------|
| WID Connectivity | Banco de dados interno do WSUS |
| WSUS Services | Serviço principal do WSUS |
| Web Server (IIS) | Interface web para comunicação com clientes |

**Tarefa 01** — Print da tela do wizard exibindo a instalação concluída com sucesso no windowsserver, mostrando o resumo dos componentes instalados: WID Connectivity, WSUS Services e Web Server (IIS).

---

### Atividade 2 – Criando GPO para WSUS

Configuração da GPO para direcionar os clientes do domínio ao servidor WSUS interno.

**Server Manager → Tools → Group Policy Management**

Expansão da árvore: **Forest → Domains → esr.rnp.edu → Group Policy Objects**

Criação de novo objeto de política: botão direito em **Group Policy Objects → New → Nome: WSUS**

Edição da GPO criada: botão direito na GPO → **Edit**

Navegação no editor:

```
Computer Configuration → Policies → Administrative Templates →
Windows Components → Windows Update
```

Configuração 1 — **Configure Automatic Updates:**
- Estado: **Enabled**
- Opção: `3 – Auto download and notify for install`

Configuração 2 — **Specify intranet Microsoft update service location:**
- Estado: **Enabled**
- Set the intranet update service for detecting updates: `http://windowsserver:8530`
- Set the intranet statistics server: `http://windowsserver:8530`

Vinculação da GPO ao domínio:

Botão direito sobre **esr.rnp.edu → Link an Existing GPO → WSUS → OK**

Atualização de política nos clientes. No PowerShell do windowsserver, conectar via SSH ao windowsclient e executar:

```powershell
ssh administrator@[IP_WINDOWSCLIENT]
gpupdate.exe /Force
exit
```

Executar também no windowsserver:

```powershell
gpupdate.exe /Force
```

**Tarefa 02** — Print da aba "Linked Group Policy Objects" do domínio esr.rnp.edu exibindo a GPO "WSUS" vinculada ao domínio com status habilitado.

---

### Atividade 3 – Configuração Inicial do WSUS

**Server Manager → Tools → Windows Server Update Services**

Na janela **Complete WSUS Installation**, verificar que o diretório é `C:\WSUS` e clicar em **Run**. Aguardar a conclusão da pós-instalação.

Wizard de configuração inicial:

| Etapa | Configuração |
|-------|-------------|
| Before You Begin | Next |
| Microsoft Update Improvement Program | Desmarcar a caixa → Next |
| Choose Upstream Server | Synchronize from Microsoft Update → Next |
| Specify Proxy Server | Sem proxy → Next |
| Connect to Upstream Server | Start Connecting (aguardar ~30 min) → Next |
| Choose Languages | English apenas → Next |
| Choose Products | Windows Subsystem for Linux (laboratório) → Next |
| Choose Classifications | Critical Updates + Security Updates → Next |
| Set Sync Schedule | Synchronize manually → Next |
| Finished | Marcar "Begin initial synchronization" → Next → Finish |

Após o wizard, verificar o status da sincronização no painel do WSUS em **WINDOWSSERVER → Overview**.

Atualização via Windows Update nos clientes:

No windowsclient e no windowsserver, buscar na barra de pesquisa por `update` e clicar em **Check for updates**.

**Tarefa 03** — Print da tela de visão geral (Overview) do WSUS Administration Console exibindo o status da sincronização e as estatísticas do servidor após a configuração inicial.

---

### Atividade 4 – Grupos de Computadores

Criação do grupo de homologação no console do WSUS:

**Computers → All Computers → botão direito → Add Computer Group → Nome: Homolog → Add**

Adição do windowsclient ao grupo Homolog:

Em **All Computers**, alterar status para **Any** e clicar em **Refresh**. Botão direito sobre **windowsclient → Change Membership → Homolog → OK**

**Tarefa 04** — Print da tela do grupo "Homolog" no WSUS Administration Console exibindo o windowsclient como membro do grupo, com as informações de sistema operacional, status de instalação e endereço IP.

---

### Atividade 5 – Criando Aprovações Automáticas

**WSUS Console → Options → Automatic Approvals**

Edição da regra padrão existente (**Default Automatic Approval Rule**):

Clicar sobre a regra → clicar em **all computers** nas propriedades → desmarcar **All Computers** e selecionar apenas **Homolog** → OK

Aplicação e execução da regra:

Marcar a regra → **Apply** → **Run Rule**

Aguardar a janela **Running Rule** confirmar a quantidade de atualizações aprovadas.

Regra configurada:

> Quando uma atualização for classificada como **Critical Updates** ou **Security Updates**, aprovar automaticamente para o grupo **Homolog**.

**Tarefa 05** — Print da janela de execução da regra exibindo a confirmação de atualizações aprovadas automaticamente para o grupo Homolog, com a barra de progresso concluída.

---

## Aprendizado

- A centralização de atualizações via WSUS resolve um problema real de escalabilidade: em redes com centenas de máquinas, cada dispositivo buscando atualizações diretamente no Microsoft Update gera consumo massivo de banda e dificulta o controle de versão. O WSUS baixa uma única vez e distribui internamente.
- A separação entre grupos de homologação e produção é uma prática de gestão de mudanças fundamental. Aplicar uma atualização crítica diretamente em servidores de produção sem teste prévio é um risco operacional que o WSUS ajuda a mitigar com pouco esforço de configuração.
- A GPO que aponta os clientes para o servidor WSUS interno substitui silenciosamente o comportamento padrão do Windows Update sem exigir intervenção manual em cada máquina. Uma única configuração propagada via GPO atinge automaticamente todo o domínio.
- As aprovações automáticas por classificação (Critical Updates, Security Updates) garantem que vulnerabilidades críticas sejam corrigidas sem depender de aprovação manual, reduzindo o tempo de exposição. Para atualizações menos urgentes, a aprovação manual permite avaliação prévia antes da distribuição.
- O WSUS armazena metadados de todas as atualizações disponíveis, mas só faz download dos binários quando uma atualização é aprovada. Em ambientes de produção com múltiplos produtos e idiomas, o volume de dados armazenados em `C:\WSUS` pode ser considerável e deve ser planejado no dimensionamento do servidor.
