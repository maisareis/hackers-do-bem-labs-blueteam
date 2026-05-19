# Atividade 2.4 – Utilizando o Docker

## Objetivo

Explorar os principais recursos do Docker em um ambiente Linux: execução de containers, orquestração com **Docker Compose**, escalabilidade com **Docker Swarm**, limitação de recursos e gerenciamento visual com **Portainer**. A atividade demonstra na prática como containers facilitam a implantação de aplicações e como o isolamento de recursos protege o ambiente de indisponibilidades.

---

## Aviso Legal

> As atividades de segurança documentadas aqui foram realizadas exclusivamente em ambientes controlados e autorizados para fins educacionais. O uso de qualquer técnica descrita em sistemas sem autorização prévia é ilegal.

---

## Conceito

### Docker

O **Docker** é uma plataforma de conteinerização que empacota aplicações e suas dependências em unidades isoladas chamadas **containers**. Diferente de máquinas virtuais, containers compartilham o kernel do sistema operacional host, tornando-os mais leves e rápidos para inicializar.

| Conceito | Descrição |
|----------|-----------|
| **Image** | Template imutável a partir do qual containers são criados |
| **Container** | Instância em execução de uma imagem |
| **Docker Compose** | Ferramenta para definir e orquestrar múltiplos containers via arquivo YAML |
| **Docker Swarm** | Orquestrador nativo do Docker para clusters de containers |
| **Stack** | Conjunto de serviços implantados via Swarm com um único arquivo Compose |
| **Portainer** | Interface gráfica web para gerenciamento de ambientes Docker |

### Limitação de Recursos

Sem limites definidos, um container pode consumir todos os recursos da máquina host, causando indisponibilidade de outros serviços. Os principais parâmetros de controle são `--cpus` (limite de CPU) e `--memory` (limite de RAM).

---

## Passo a Passo

### Ambiente

| VM | IP | Usuário | Função |
|----|----|---------|--------|
| linuxserver | 192.168.98.10 | root | Host Docker |
| linuxclient | 192.168.98.11 | root | Cliente de testes (httperf) |

---

### Atividade 01 – Verificando a Versão e Executando o Primeiro Container

Acesso ao servidor:

```bash
ssh root@192.168.98.10
whoami; hostname
```

Inicialização e verificação do Docker:

```bash
systemctl start docker
systemctl status docker
```

Verificação da versão instalada:

```bash
docker --version
```

Execução do container de teste:

```bash
docker run hello-world
```

O Docker verifica se a imagem `hello-world` existe localmente; não encontrando, faz o pull do Docker Hub, cria o container, executa o binário que imprime a mensagem e encerra.

Listagem de todos os containers (incluindo os encerrados):

```bash
date; docker ps -a
```

**Tarefa 01** — Print da saída do `docker ps -a` mostrando o container `hello-world` com status `Exited (0)`:

> **[INSERIR PRINT DA TAREFA 01]**

---

### Atividade 02 – Docker Compose: Ambiente LAMP

Verificação da versão do Docker Compose:

```bash
docker-compose --version
```

Criação da estrutura de diretórios do projeto:

```bash
mkdir -p /root/lamp-lab/www; cd /root/lamp-lab
```

Criação do arquivo `docker-compose.yml`:

```bash
nano /root/lamp-lab/docker-compose.yml
```

Conteúdo do arquivo:

```yaml
version: '3.7'

services:
  apache:
    image: php:7.4-apache
    container_name: apache
    ports:
      - "8088:80"
    volumes:
      - ./www:/var/www/html
    depends_on:
      - mysql
    deploy:
      resources:
        limits:
          cpus: '0.5'

  mysql:
    image: mysql:5.7
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - mysql-data:/var/lib/mysql
    deploy:
      resources:
        limits:
          cpus: '0.5'

volumes:
  mysql-data:
```

Criação da página PHP de teste:

```bash
nano /root/lamp-lab/www/index.php
```

Conteúdo:

```php
<?php
phpinfo();
?>
```

Inicialização dos containers:

```bash
docker-compose up -d
```

Acesso à aplicação via navegador: `http://192.168.98.10:8088`

**Tarefa 02** — Print da página `PHP Version 7.4.33 phpinfo()` no navegador:

> **[INSERIR PRINT DA TAREFA 02]**

Remoção dos containers e limpeza das imagens:

```bash
docker-compose down
docker system prune -a
# Confirmar com: y
```

---

### Atividade 03 – Docker Swarm: Escalabilidade Horizontal

Criação do diretório do projeto:

```bash
mkdir /root/stack-lab; cd /root/stack-lab
```

Criação do arquivo `docker-compose.yml`:

```yaml
version: '3.7'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '0.5'
          memory: 50M
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

Inicialização do cluster Swarm (modo single-node):

```bash
docker swarm init
```

Deploy da stack:

```bash
docker stack deploy -c docker-compose.yml webstack
```

#### Terminal 01 — linuxserver: monitoramento de recursos

```bash
docker stats
```

Com 1 réplica, o container atinge ~50% de CPU sob carga.

#### Terminal do linuxclient: teste de carga

```bash
httperf --server 192.168.98.10 --port 8080 --num-call 10 --num-conns 10000 \
  --rate 500 --timeout 20
```

Resultado com 1 réplica: `Reply rate avg ~2185 req/s`, `stddev 386.2` (alta variação, muitos erros).

#### Terminal 02 — linuxserver: escalonamento para 8 réplicas

```bash
docker service scale webstack_web=8
```

Após todos os 8 containers subirem, repetir o teste no linuxclient. Com a carga distribuída entre as réplicas, cada container usa ~15% de CPU.

Resultado com 8 réplicas: `Reply rate avg ~4999 req/s`, `stddev 1.4` (resposta estável, zero erros).

| Métrica | 1 réplica | 8 réplicas |
|---------|-----------|------------|
| Reply rate avg | ~2185 req/s | ~4999 req/s |
| Stddev | 386.2 | 1.4 |
| Erros totais | 4513 | 0 |
| CPU por container | ~50% | ~15% |

O desvio padrão próximo de zero na segunda execução indica que a carga foi distribuída uniformemente entre os containers, resultando em respostas consistentes e previsíveis.

**Tarefa 03** — Print da saída do httperf com 8 réplicas mostrando `Errors: total 0`:

> **[INSERIR PRINT DA TAREFA 03]**

Encerramento dos containers:

```bash
docker stop $(docker ps -q)
```

---

### Atividade 04 – Limitando o Uso de Recursos

Criação do diretório do projeto:

```bash
mkdir -p /root/temp-lab; cd /root/temp-lab
```

#### Terminal 01 — linuxserver: iniciar container sem limite de CPU

```bash
docker run -p 8888:8080 -d rconz/conversao-temperatura
docker stats
```

#### Terminal 02 — linuxserver: estressar a CPU do container

```bash
curl -X PUT http://localhost:8888/stress/tempo/120/intervalo/1/ciclos/2
```

O terminal 01 mostrará o container consumindo ~99% de CPU — comportamento que pode derrubar outros serviços no host.

Parada do container:

```bash
docker ps -q | xargs docker stop
```

> **Atenção:** Este comando para **todos** os containers em execução. Use apenas em ambientes de laboratório.

Reinicialização com limite de 50% de CPU:

```bash
docker run -p 8888:8080 --cpus 0.5 -d rconz/conversao-temperatura
```

Novo teste de estresse:

```bash
date; curl -X PUT http://localhost:8888/stress/tempo/120/intervalo/1/ciclos/2
```

O terminal 01 confirmará que o uso de CPU não ultrapassa 50%, independentemente da demanda interna do container.

**Tarefa 04** — Print dos dois terminais: Terminal 01 com `docker stats` mostrando CPU ~50% e Terminal 02 com o comando `curl` e a data:

> **[INSERIR PRINT DA TAREFA 04]**

---

### Atividade 05 – Docker Portainer

Criação do diretório do projeto:

```bash
mkdir -p /root/portainer-lab; cd /root/portainer-lab
```

Criação do arquivo `docker-compose.yml`:

```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

volumes:
  portainer_data:
```

Deploy da stack no Swarm:

```bash
docker stack deploy -c docker-compose.yml portainer_stack
```

Verificação do status do serviço:

```bash
docker service ls
docker service ps portainer_stack_portainer
```

Acesso via navegador: `https://192.168.98.10:9443` (aceitar o aviso de certificado autoassinado).

Na primeira execução, criar o usuário administrador:

| Campo | Valor |
|-------|-------|
| Usuário | `admin` |
| Senha | `RnpEsr123456@` |

Caso a interface não carregue, forçar reinicialização do serviço:

```bash
docker service update --force portainer_stack_portainer
```

No painel do Portainer: **Local → Containers** para visualizar todos os containers em execução.

**Tarefa 05** — Print da tela "Containers" do Portainer:

> **[INSERIR PRINT DA TAREFA 05]**

Limpeza ao final do laboratório:

```bash
docker stack rm portainer_stack
docker system prune -a
systemctl stop docker.service
systemctl stop docker.socket
```

---

## Aprendizado

- Containers sem limites de recurso podem monopolizar CPU e memória do host, causando indisponibilidade em cascata. O parâmetro `--cpus` (ou `deploy.resources.limits.cpus` no Compose) é uma medida simples e eficaz de isolamento de falhas.
- O **Docker Swarm** permite escalonamento horizontal com um único comando (`docker service scale`). A comparação dos resultados do httperf (stddev de 386 vs 1.4) ilustra objetivamente como a distribuição de carga aumenta a previsibilidade e a resiliência da aplicação.
- O volume `/var/run/docker.sock` montado no Portainer concede ao container acesso total ao daemon Docker do host — equivalente a privilégios de root. Em produção, proteger o acesso ao Portainer com autenticação forte e restringir o acesso à porta 9443 por firewall.
- O comando `docker system prune -a` remove **todas** as imagens não associadas a containers em execução, liberando espaço em disco. Útil em laboratório, mas potencialmente destrutivo em produção se houver imagens que ainda serão reutilizadas.
- A montagem de volumes (`./www:/var/www/html` no LAMP lab) permite desenvolver localmente e ver as alterações refletidas instantaneamente no container, sem precisar reconstruir a imagem.
