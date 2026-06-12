# Atividade 1.2 – Ataques e Estratégias de Defesa

## Objetivo

Explorar ferramentas e estratégias de defesa cibernética, incluindo análise de ameaças com VirusTotal, criação de arquivo de teste EICAR, aplicação das diretrizes do CIS Controls e mapeamento de técnicas de ataque com MITRE ATT&CK e Navigator.

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

## Conceito

- **VirusTotal**: plataforma online que analisa arquivos e URLs com múltiplos antivírus.
- **EICAR**: arquivo de teste seguro para validar o funcionamento de antivírus.
- **CIS Controls**: conjunto de boas práticas para segurança de TI.
- **MITRE ATT&CK**: framework que cataloga táticas, técnicas e procedimentos (TTPs) de adversários.

## Passo a Passo

### Atividade 1 – Explorando recursos do VirusTotal

#### 1.1 – Analisar URL suspeita

Acessar o VirusTotal (https://www.virustotal.com) e cole a URL: `https://scoala10gl.ro/wp-includes/images/rtazmb0001/me.php`



#### 1.2 – Pesquisar por IP malicioso

Pesquisar pelo IP `xx.xx.xxx.x` no VirusTotal. Acesse a aba **Relations > Communicating Files**.


### Atividade 2 – Criando e analisando arquivo EICAR

Criar um arquivo `blueteam.txt` com o conteúdo: `X5O!P%@AP[4\PZX54(P^)7CC)7]$EICAR-STANDARD-ANTIVIRUS-TEST-FILE$H+H*`

Enviar o arquivo para análise no VirusTotal.


### Atividade 3 – CIS Controls e MITRE ATT&CK

#### 3.1 – CIS Control 6 – Access Control Management

No CIS Controls versão 8, localizar o controle **6.5 – Require MFA for Administrative Access**.



#### 3.2 – MITRE ATT&CK – Técnica Password Spraying

No MITRE ATT&CK, navegar até **Credential Access (TA0006)** → **Brute Force** → **Password Spraying (T1110.003)**.



### Atividade 4 – Mapeamento entre CIS Controls e MITRE

No CIS Navigator, ativar o mapeamento com **MITRE Enterprise ATT&CK v8.2**.


### Atividade 5 – MITRE Navigator: comparação entre APT3 e APT29

No MITRE ATT&CK Navigator:
1. Criar uma camada para **APT3** — cor e pontuação `3`
2. Criar uma camada para **APT29** — cor e pontuação `2`
3. Combinar as camadas com a expressão `a+b`



## Aprendizado

- O VirusTotal permite identificar ameaças antes mesmo de qualquer execução local, cruzando resultados de dezenas de antivírus
- O arquivo EICAR valida se a solução antivírus está ativa e funcional sem expor o ambiente a risco real
- O CIS Controls oferece orientações práticas e priorizadas para reduzir superfície de ataque
- O MITRE ATT&CK permite entender o comportamento de adversários reais e antecipar vetores de ataque
- O CIS Navigator e o MITRE Navigator facilitam a visualização de cobertura de controles e sobreposição entre grupos de ameaça
