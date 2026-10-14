---
title: "Projeto 9/10 – Arquitetura Completa com HA, Backup, Monitoramento e Automação"
date: 2025-11-22T12:58:02-03:00
draft: true
categories: ["Linux & Infra", "PostgreSQL", "Scripts & Automação"]
tags: ["Postgre SQL", "Projetos"]
cover:
  image: "projeto9.png"
---

Neste laboratório eu montei um ambiente completo de PostgreSQL com alta disponibilidade, automação operacional, backup e monitoramento integrado.

A arquitetura usa:

- **Spilo (Patroni + PostgreSQL)** para gerenciar o cluster HA
- **etcd** como backend de consenso
- **HAProxy** como ponto único de acesso à escrita/leitura
- **postgres_exporter + Prometheus + Grafana** para monitoramento
- **scripts de backup e healthcheck** para operações do dia a dia

Tudo organizado em /home/lab09/projeto09, pronto para ser destruído e recriado sempre que necessário.

# Estrutura de diretórios

Primeiro, criei a estrutura base do projeto:

```text
mkdir -p /home/lab09/projeto09/{haproxy,monitoring,backup,automation,data}
cd /home/lab09/projeto09
```

A estrutura final fica assim:

```text
/home/lab09/projeto09
├── docker-compose.yml
├── haproxy/
│   └── haproxy.cfg
├── monitoring/
│   ├── prometheus.yml
│   └── exporter.env
├── backup/
│   └── run_backup.sh
├── automation/
│   └── healthcheck.sh
└── data/
    ├── patroni1/
    └── patroni2/
```

# Criando o docker-compose do laboratório completo

O docker-compose.yml orquestra tudo:

- etcd
- 2 nós Spilo (Patroni + PostgreSQL)
- HAProxy
- postgres_exporter
- Prometheus
- Grafana

```text
cd /home/lab09/projeto09
vi docker-compose.yml
```

Cole dentro do vi:

```text
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.15
    container_name: projeto09-etcd
    command: >
      /usr/local/bin/etcd
      --data-dir=/etcd-data
      --name=etcd0
      --initial-advertise-peer-urls=http://etcd:2380
      --listen-peer-urls=http://0.0.0.0:2380
      --listen-client-urls=http://0.0.0.0:2379
      --advertise-client-urls=http://etcd:2379
      --initial-cluster=etcd0=http://etcd:2380
      --initial-cluster-token=etcd-cluster-1
      --initial-cluster-state=new
    volumes:
      - ./data/etcd:/etcd-data
    networks:
      - projeto09-net

  patroni1:
    image: ghcr.io/zalando/spilo-17:4.0-p2
    container_name: projeto09-patroni1
    depends_on:
      - etcd
    environment:
      SPILO_PROVIDER: local
      SCOPE: projeto09
      ETCD3_HOSTS: projeto09-etcd:2379
      PGUSER_SUPERUSER: postgres
      PGPASSWORD_SUPERUSER: postgres
      PGUSER_STANDBY: standby
      PGPASSWORD_STANDBY: standby
      RESTAPI_CONNECT_ADDRESS: projeto09-patroni1:8008
      SPILO_CONFIGURATION: |
        postgresql:
          connect_address: projeto09-patroni1:5432
          pg_hba:
            - local   all             all                                   trust
            - host    all             all              0.0.0.0/0            trust
            - host    replication     all              0.0.0.0/0            trust
    volumes:
      - ./data/patroni1:/home/postgres/pgdata
    ports:
      - "5433:5432"
      - "8008:8008"
    networks:
      - projeto09-net

  patroni2:
    image: ghcr.io/zalando/spilo-17:4.0-p2
    container_name: projeto09-patroni2
    depends_on:
      - etcd
    environment:
      SPILO_PROVIDER: local
      SCOPE: projeto09
      ETCD3_HOSTS: projeto09-etcd:2379
      PGUSER_SUPERUSER: postgres
      PGPASSWORD_SUPERUSER: postgres
      PGUSER_STANDBY: standby
      PGPASSWORD_STANDBY: standby
      RESTAPI_CONNECT_ADDRESS: projeto09-patroni2:8008
      SPILO_CONFIGURATION: |
        postgresql:
          connect_address: projeto09-patroni2:5432
          pg_hba:
            - local   all             all                                   trust
            - host    all             all              0.0.0.0/0            trust
            - host    replication     all              0.0.0.0/0            trust
    volumes:
      - ./data/patroni2:/home/postgres/pgdata
    ports:
      - "5434:5432"
      - "8009:8008"
    networks:
      - projeto09-net

  haproxy:
    image: haproxy:2.9
    container_name: projeto09-haproxy
    depends_on:
      - patroni1
      - patroni2
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    ports:
      - "5000:5000"  # escrita (sempre no master)
      - "7000:7000"  # leitura (replicas)
    networks:
      - projeto09-net

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres_exporter
    env_file:
      - ./monitoring/exporter.env
    ports:
      - "9187:9187"
    networks:
      - projeto09-net

  prometheus:
    image: prom/prometheus
    container_name: projeto09-prometheus
    depends_on:
      - postgres_exporter
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    networks:
      - projeto09-net

  grafana:
    image: grafana/grafana
    container_name: projeto09-grafana
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    networks:
      - projeto09-net

networks:
  projeto09-net:
    driver: bridge
```

Salvar e sair do vi.

# Configurando o HAProxy

O HAProxy é o front-end do cluster: a aplicação só enxerga localhost:5000 (escrita) e localhost:7000 (leituras).

```text
cd /home/lab09/projeto09
mkdir -p haproxy
cd haproxy
vi haproxy.cfg
```

Conteúdo:

```text
global
    daemon
    maxconn 256
    log stdout format raw local0

defaults
    mode tcp
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend pg_write
    bind *:5000
    default_backend pg_master

backend pg_master
    option httpchk GET /master
    http-check expect status 200
    server patroni1 projeto09-patroni1:5432 check port 8008
    server patroni2 projeto09-patroni2:5432 check port 8008 backup

frontend pg_read
    bind *:7000
    default_backend pg_replicas

backend pg_replicas
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    server patroni1 projeto09-patroni1:5432 check port 8008
    server patroni2 projeto09-patroni2:5432 check port 8008
```

# Configurando Prometheus e postgres_exporter

O Prometheus coleta as métricas do exporter e o Grafana consome do Prometheus.

```text
cd /home/lab09/projeto09
mkdir -p monitoring
cd monitoring
vi prometheus.yml
```

Conteúdo:

```text
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgres_exporter'
    static_configs:
      - targets: ['postgres_exporter:9187']
```

Agora o arquivo de ambiente do exporter:

```text
cd /home/lab09/projeto09/monitoring
vi exporter.env
```

Conteúdo:

```text
DATA_SOURCE_NAME=postgresql://postgres:postgres@projeto09-patroni1:5432/postgres?sslmode=disable
```

O exporter conecta direto no projeto09-patroni1 (que será o líder inicial) com usuário postgres.

# Script de backup lógico

Para este lab usei pg_dumpall, que já vem dentro do container Spilo.

```text
cd /home/lab09/projeto09
mkdir -p backup
cd backup
vi run_backup.sh
```

Conteúdo:

```text
#!/bin/bash

BACKUP_DIR="/home/lab09/projeto09/backup/files"
mkdir -p "$BACKUP_DIR"

FILE="$BACKUP_DIR/dump_$(date +%Y%m%d%H%M%S).sql"

echo "Gerando backup lógico em $FILE ..."
docker exec projeto09-patroni1 pg_dumpall -U postgres > "$FILE"

if [ $? -eq 0 ]; then
  echo "Backup concluído com sucesso."
else
  echo "Falha ao gerar backup."
fi
```

Dar permissão de execução:

```text
cd /home/lab09/projeto09/backup
chmod +x run_backup.sh
```

# Script de healthcheck do cluster

Esse script faz duas coisas:

- Mostra o estado do cluster pelo patronictl list
- Executa um SELECT simples para validar o nó atual

```text
cd /home/lab09/projeto09
mkdir -p automation
cd automation
vi healthcheck.sh
```

Conteúdo:

```text
#!/bin/bash

echo "=== Estado do cluster (patronictl list) ==="
docker exec projeto09-patroni1 patronictl list || {
  echo "Falha ao executar patronictl."
  exit 1
}

echo
echo "=== Testando SELECT no nó atual (via projeto09-patroni1) ==="
docker exec projeto09-patroni1 psql -U postgres -d postgres -c "SELECT inet_server_addr(), pg_is_in_recovery(), now();" || {
  echo "Falha ao executar SELECT no PostgreSQL."
  exit 1
}
```

Permissão:

```text
cd /home/lab09/projeto09/automation
chmod +x healthcheck.sh
```

# Subindo o ambiente

Agora sim, com todos os arquivos no lugar:

```text
cd /home/lab09/projeto09

# Derruba qualquer coisa antiga
docker compose down

# Garante diretórios limpos para bootstrap
rm -rf data/patroni1 data/patroni2

# Sobe tudo
docker compose up -d
```

Verificando:

```text
docker ps
```

Você deve ver:

![](image-27-1024x166.png)

- projeto09-etcd
- projeto09-patroni1
- projeto09-patroni2
- projeto09-haproxy
- postgres_exporter
- projeto09-prometheus
- projeto09-grafana

# Validando o cluster e o HA

Ver o estado do cluster:

```text
cd /home/lab09/projeto09
./automation/healthcheck.sh
```

Saída esperada:

- Tabela do patronictl list com um **Leader (running)** e uma **Replica (running)**
- pg_is_in_recovery() = false no líder

## Testando o cluster e o acesso via HAProxy

Depois de subir o ambiente com:

```text
cd /home/lab09/projeto09
docker compose up -d
docker ps
```

validei o estado do cluster e as conexões usando apenas comandos dentro dos containers.

### Verificando o estado do cluster com Patroni

```text
cd /home/lab09/projeto09

docker exec projeto09-patroni1 patronictl list
```

Esse comando mostra quem é o líder, quem é réplica e o estado de cada nó.

![](image-28.png)

## Testando conexão direta no líder

Para validar se o PostgreSQL do líder está aceitando conexões normalmente:

```text
docker exec -it projeto09-patroni1 \\
  psql -U postgres -d postgres \\
  -c "SELECT inet_server_addr(), pg_is_in_recovery(), now();"
```

Aqui:

- pg_is_in_recovery() = false indica que esse nó é o primário
- now() confirma o horário da consulta
- inet_server_addr() pode vir vazio por ser conexão local via socket, o que é esperado

## Testando conexão via HAProxy (como se fosse a aplicação)

Em vez de usar psql no host, eu também testei o fluxo passando pelo HAProxy, mas ainda de dentro do container:

```text
docker exec -it projeto09-patroni1 \\
  psql -h projeto09-haproxy -p 5000 -U postgres -d postgres \\
  -c "SELECT inet_server_addr(), pg_is_in_recovery(), now();"
```![](image-29.png)

Esse comando simula exatamente o que uma aplicação faria:

- Conecta em projeto09-haproxy:5000
- O HAProxy identifica o líder pelo healthcheck do Patroni
- Redireciona a conexão para o nó primário correto

O inet_server_addr() passa a mostrar o IP do container que está atuando como líder (por exemplo, 172.28.0.5), e pg_is_in_recovery() = false confirma que a escrita está caindo no primário.

## Teste de failover usando apenas Docker

Para validar o failover automático, derrubei o nó líder:

```text
cd /home/lab09/projeto09
docker stop projeto09-patroni1
```

Em seguida, conferi o estado do cluster a partir do outro nó:

```text
docker exec projeto09-patroni2 patronictl list
```![](image-30.png)

E validei a conexão via HAProxy, ainda usando docker exec:

```text
docker exec -it projeto09-patroni2 \\
  psql -h projeto09-haproxy -p 5000 -U postgres -d postgres \\
  -c "SELECT inet_server_addr(), pg_is_in_recovery(), now();"
```

O resultado mostra:

- Novo IP em inet_server_addr() (do container que virou líder)
- pg_is_in_recovery() = false confirmando que o tráfego já está indo para o novo primário

# Monitoramento

Assim como já ensinei nos meus outros projetos, apenas relembrando, para acessar a parte de monitoração:

Prometheus:

```text
http://SEU_HOST:9090
```

Grafana:

```text
http://SEU_HOST:3000
```

Login padrão:

```text
admin / admin
```

Importe dashboards de PostgreSQL e Prometheus e comece a visualizar métricas como:

- Lag de replicação
- Conexões ativas
- Buffer cache
- Checkpoints
- Uso de WAL

O Projeto 9/10 entrega um ambiente PostgreSQL completo, resiliente e observável, com todas as peças que compõem uma arquitetura profissional:

- Alta Disponibilidade real com **Patroni + etcd**
- Roteamento inteligente com **HAProxy**
- Monitoramento avançado com **Prometheus + Grafana**
- Exportação de métricas dedicada com **postgres_exporter**
- **Backup** e **healthcheck** prontos para automação
