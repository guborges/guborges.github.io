---
title: "Projeto 10/10 – Aplicação + PostgreSQL HA com Patroni, Failover Automático, CI/CD, Monitoramento e Backup"
date: 2025-11-24T08:54:39-03:00
draft: true
categories: ["PostgreSQL", "Replicação", "Scripts & Automação"]
tags: ["Postgre SQL", "Projetos"]
cover:
  image: "projeto10.png"
---

Este é o décimo e último projeto da minha série de laboratórios de Database Engineering.

Nos projetos anteriores eu fui construindo as peças separadas: instalação, tuning, replicação, alta disponibilidade, observabilidade, automação, backups e pipelines de migração.

Agora, neste projeto final, o objetivo é **juntar tudo em uma plataforma funcional ponta a ponta**:

- Aplicação real (FastAPI) consumindo o banco
- Cluster PostgreSQL em **alta disponibilidade com Patroni (Spilo) + etcd**
- **HAProxy** como ponto único de acesso
- **Flyway** aplicando migrações de schema
- Observabilidade com **Prometheus + Grafana**
- Métricas de banco com **postgres_exporter**
- Script de **backup lógico**
- Script de **healthcheck do cluster**

Tudo isso organizado em /home/lab10/projeto10, pronto para subir, derrubar e recriar quantas vezes for necessário.

# Estrutura do laboratório

Crie a estrutura base:

```
mkdir -p /home/lab10/projeto10/{app,haproxy,monitoring,backup,automation,cicd/sql,data}
cd /home/lab10/projeto10
```

Estrutura final:

```
/home/lab10/projeto10
├── docker-compose.yml
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
├── haproxy/
│   └── haproxy.cfg
├── monitoring/
│   ├── prometheus.yml
│   └── exporter.env
├── backup/
│   └── run_backup.sh
├── automation/
│   └── healthcheck.sh
└── cicd/
    ├── sql/
    │   └── V1__create_users.sql
    └── flyway.yml   (exemplo de pipeline CI/CD)
```

# Criando a aplicação FastAPI

A aplicação representa um cenário real: grava e lê usuários na tabela users do banco appdb, sempre passando pelo HAProxy.

## requirements.txt

```
cd /home/lab10/projeto10/app
vi requirements.txt
```

Conteúdo:

```
fastapi
uvicorn
psycopg2-binary
```

## [app.py](http://app.py)

```
cd /home/lab10/projeto10/app
vi app.py
```

Conteúdo:

```
from fastapi import FastAPI
import psycopg2
import os

app = FastAPI()

def get_conn():
    return psycopg2.connect(
        host=os.getenv("DB_HOST"),
        port=5000,
        user="postgres",
        password="postgrespass",
        dbname="appdb"
    )

@app.get("/users")
def list_users():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, name, created_at FROM users ORDER BY id;")
    rows = cur.fetchall()
    conn.close()
    return [{"id": r[0], "name": r[1], "created_at": r[2]} for r in rows]

@app.post("/users/{name}")
def create_user(name: str):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("INSERT INTO users (name) VALUES (%s)", (name,))
    conn.commit()
    conn.close()
    return {"status": "ok", "name": name}
```

## Dockerfile

```
cd /home/lab10/projeto10/app
vi Dockerfile
```

Conteúdo:

```
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

# Migrações com Flyway

As migrações garantem que o schema do banco acompanhe a aplicação.

## V1__create_users.sql

```
cd /home/lab10/projeto10/cicd/sql
vi V1__create_users.sql
```

Conteúdo:

```
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

## Exemplo de pipeline CI/CD (opcional)

Esse arquivo é um exemplo de pipeline (GitHub Actions) rodando o Flyway:

```
cd /home/lab10/projeto10/cicd
vi flyway.yml
```

Conteúdo:

```
name: Flyway Migrations

on:
  push:
    paths:
      - "cicd/sql/**"

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Flyway migrations
        uses: flyway/flyway-action@v2
        with:
          url: jdbc:postgresql://YOUR-HA-ENDPOINT:5000/appdb
          user: postgres
          password: postgrespass
          locations: filesystem:cicd/sql
```

# Configurando o HAProxy

O HAProxy será o endpoint único das conexões da aplicação, sempre apontando para o nó líder.

```
cd /home/lab10/projeto10/haproxy
vi haproxy.cfg
```

Conteúdo:

```
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
    server patroni1 projeto10-patroni1:5432 check port 8008
    server patroni2 projeto10-patroni2:5432 check port 8008 backup
```

# Monitoramento com Prometheus e postgres_exporter

## prometheus.yml

```
cd /home/lab10/projeto10/monitoring
vi prometheus.yml
```

Conteúdo:

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgres_exporter'
    static_configs:
      - targets: ['postgres_exporter:9187']
```

## exporter.env

```
cd /home/lab10/projeto10/monitoring
vi exporter.env
```

Conteúdo:

```
DATA_SOURCE_NAME=postgresql://postgres:postgrespass@projeto10-patroni1:5432/appdb?sslmode=disable
```

# Script de backup lógico

Um backup lógico simples com pg_dumpall, rodando de dentro do contêiner do líder (ou réplica).

```
cd /home/lab10/projeto10/backup
vi run_backup.sh
```

Conteúdo:

```
#!/bin/bash

BACKUP_DIR="/home/lab10/projeto10/backup/files"
mkdir -p "$BACKUP_DIR"

FILE="$BACKUP_DIR/dump_$(date +%Y%m%d%H%M%S).sql"

echo "Gerando backup lógico em $FILE ..."
docker exec projeto10-patroni1 pg_dumpall -U postgres > "$FILE"

if [ $? -eq 0 ]; then
  echo "Backup concluído com sucesso."
else
  echo "Falha ao gerar backup."
fi
```

Permissão:

```
cd /home/lab10/projeto10/backup
chmod +x run_backup.sh
```

# Script de healthcheck do cluster

O healthcheck mostra o estado do cluster via Patroni e faz um SELECT via HAProxy.

```
cd /home/lab10/projeto10/automation
vi healthcheck.sh
```

Conteúdo:

```
#!/bin/bash

echo "=== Estado do cluster (patronictl list) ==="
docker exec projeto10-patroni1 patronictl list || {
  echo "Falha ao executar patronictl."
  exit 1
}

echo
echo "=== Testando SELECT via HAProxy (appdb) ==="
docker exec projeto10-patroni1 psql -h projeto10-haproxy -p 5000 -U postgres -d appdb -c "SELECT inet_server_addr(), pg_is_in_recovery(), now();" || {
  echo "Falha ao executar SELECT no PostgreSQL."
  exit 1
}
```

Permissão:

```
cd /home/lab10/projeto10/automation
chmod +x healthcheck.sh
```

# Docker Compose com Spilo, HAProxy, App e Monitoramento

```
cd /home/lab10/projeto10
vi docker-compose.yml
```

Conteúdo:

```
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.15
    container_name: projeto10-etcd
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
      - projeto10-net

  projeto10-patroni1:
    image: ghcr.io/zalando/spilo-17:4.0-p2
    container_name: projeto10-patroni1
    depends_on:
      - etcd
    environment:
      SPILO_PROVIDER: local
      SCOPE: projeto10
      ETCD3_HOSTS: projeto10-etcd:2379
      PGUSER_SUPERUSER: postgres
      PGPASSWORD_SUPERUSER: postgrespass
      PGUSER_STANDBY: standby
      PGPASSWORD_STANDBY: standbypass
      SPILO_CONFIGURATION: |
        postgresql:
          connect_address: projeto10-patroni1:5432
          pg_hba:
            - local   all             all                                   trust
            - host    all             all              0.0.0.0/0            trust
            - host    replication     all              0.0.0.0/0            trust
    volumes:
      - ./data/patroni1:/home/postgres/pgdata
    ports:
      - "5435:5432"
      - "8008:8008"
    networks:
      - projeto10-net

  projeto10-patroni2:
    image: ghcr.io/zalando/spilo-17:4.0-p2
    container_name: projeto10-patroni2
    depends_on:
      - etcd
    environment:
      SPILO_PROVIDER: local
      SCOPE: projeto10
      ETCD3_HOSTS: projeto10-etcd:2379
      PGUSER_SUPERUSER: postgres
      PGPASSWORD_SUPERUSER: postgrespass
      PGUSER_STANDBY: standby
      PGPASSWORD_STANDBY: standbypass
      SPILO_CONFIGURATION: |
        postgresql:
          connect_address: projeto10-patroni2:5432
          pg_hba:
            - local   all             all                                   trust
            - host    all             all              0.0.0.0/0            trust
            - host    replication     all              0.0.0.0/0            trust
    volumes:
      - ./data/patroni2:/home/postgres/pgdata
    ports:
      - "5436:5432"
      - "8009:8008"
    networks:
      - projeto10-net

  projeto10-haproxy:
    image: haproxy:2.9
    container_name: projeto10-haproxy
    depends_on:
      - projeto10-patroni1
      - projeto10-patroni2
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    ports:
      - "5000:5000"
    networks:
      - projeto10-net

  app:
    build: ./app
    container_name: projeto10-app
    environment:
      - DB_HOST=projeto10-haproxy
    depends_on:
      - projeto10-haproxy
    ports:
      - "8080:8080"
    networks:
      - projeto10-net

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres_exporter
    env_file:
      - ./monitoring/exporter.env
    ports:
      - "9187:9187"
    networks:
      - projeto10-net

  prometheus:
    image: prom/prometheus
    container_name: projeto10-prometheus
    depends_on:
      - postgres_exporter
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    networks:
      - projeto10-net

  grafana:
    image: grafana/grafana
    container_name: projeto10-grafana
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    networks:
      - projeto10-net

  flyway:
    image: flyway/flyway:10
    container_name: projeto10-flyway
    depends_on:
      - projeto10-haproxy
    command: >
      -url=jdbc:postgresql://projeto10-haproxy:5000/appdb
      -user=postgres
      -password=postgrespass
      -locations=filesystem:/flyway/sql
      migrate
    volumes:
      - ./cicd/sql:/flyway/sql
    networks:
      - projeto10-net

networks:
  projeto10-net:
    driver: bridge
```

# Subindo o ambiente

```
cd /home/lab10/projeto10

docker compose down
rm -rf data/patroni1 data/patroni2 data/etcd

docker compose up -d
docker ps
```![](image-33-1024x271.png)

Confirme o estado do cluster:

```
docker exec projeto10-patroni1 patronictl list
```![](image-34.png)

Você deve ver um Leader running (pode ser o patroni1 ou o patroni2) e um Replica streaming.

# Criando o banco appdb e rodando as migrações

Sempre crio o appdb via HAProxy, independente de quem é o líder:

```
cd /home/lab10/projeto10

docker exec -it projeto10-patroni1 psql -h projeto10-haproxy -p 5000 -U postgres -d postgres -c "CREATE DATABASE appdb;"
```

Se o banco já existir, vai retornar apenas um aviso.

Agora rodo o Flyway:

```
docker compose run --rm flyway
```![](image-35.png)

Verificação:

```
docker exec -it projeto10-patroni2 psql -U postgres -d appdb -c "\\dt"
```![](image-36.png)

A tabela users deve aparecer.

# Testando aplicação e HA

## Healthcheck

```
cd /home/lab10/projeto10
./automation/healthcheck.sh
```![](image-37.png)

Você verá:

- A saída do patronictl list
- Um SELECT mostrando pg_is_in_recovery() e now() via HAProxy

## Criando um usuário via API

```
curl -X POST http://localhost:8080/users/Gustavo
```![](image-38.png)

## Listando usuários

```
curl http://localhost:8080/users
```

# Testando failover com Patroni

Em vez de “puxar o cabo” com docker stop, aqui uso o switchover controlado do Patroni.

Primeiro, vejo quem é líder:

```
docker exec projeto10-patroni1 patronictl list
```

Supondo que a saída mostre:

- Leader: projeto10-patroni2
- Replica: projeto10-patroni1

Posso inverter os papéis assim:

```
docker exec -it projeto10-patroni1 patronictl switchover
```![](image-40.png)

Depois confiro de novo:

```
docker exec projeto10-patroni1 patronictl list
```

O líder deve ter trocado.

Agora testo a aplicação outra vez:

```
curl http://localhost:8080/users
```

A aplicação continua funcionando normalmente, mesmo com a troca de líder, porque ela enxergou apenas projeto10-haproxy:5000.

Se quiser, você pode repetir o switchover invertendo os parâmetros (promovendo de volta o outro nó).

# Rodando backup

```
cd /home/lab10/projeto10/backup
./run_backup.sh

ls -1 /home/lab10/projeto10/backup/files
```

Você terá arquivos dump_YYYYMMDDHHMMSS.sql com o backup lógico completo.

# Monitoramento

- Prometheus: <http://localhost:9090>
- Grafana: <http://localhost:3000> (login padrão: admin / admin)

No Grafana, adicione o Prometheus ( <http://projeto10-prometheus:9090> ) como data source e importe dashboards de PostgreSQL/Patroni.

Você pode acompanhar:

- Lag de replicação
- Número de conexões
- Checkpoints
- Taxa de WAL
- Status dos nós

# Conclusão projeto 10/10

Este Projeto 10/10 fecha a jornada saindo de laboratórios isolados e chegando em uma **plataforma completa de banco de dados**:

- Banco PostgreSQL em alta disponibilidade com **Patroni + etcd**
- **HAProxy** entregando um endpoint único para a aplicação
- Aplicação real (FastAPI) consumindo o cluster
- Migrações de schema versionadas com **Flyway**
- Monitoramento com **Prometheus + Grafana**
- Métricas expostas via **postgres_exporter**
- Rotinas de **backup** e **healthcheck** prontas para automação
- Testes de **failover controlado** via Patroni, mantendo a aplicação online

Ao longo desses dez projetos, a visão deixou de ser “subir um banco” e passou a ser criar **uma plataforma de dados**: alta disponibilidade, automação, observabilidade, proteção de dados e operação contínua.

Agora, com os 10 laboratórios prontos, temos um portfólio prático que mostra não só que sabe usar PostgreSQL, mas que sabe **projetar, operar e evoluir** uma arquitetura completa de banco em produção.

![](ferias2-1024x683.png)
