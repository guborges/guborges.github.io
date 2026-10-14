---
title: "Como montar um ambiente completo com Docker, PostgreSQL, Prometheus e Grafana"
date: 2025-11-17T07:43:19-03:00
draft: false
categories: ["Linux & Infra", "PostgreSQL", "Scripts & Automação"]
tags: ["Postgre SQL"]
cover:
  image: "docker-1.png"
---

Este artigo documenta passo a passo a criação de um ambiente de banco de dados utilizando Docker, PostgreSQL, PgAdmin, Prometheus e Grafana. A ideia é montar um stack moderno de observabilidade e administração utilizando apenas containers e um arquivo Docker Compose.

O tutorial foi escrito para ser totalmente reproduzível. Mesmo quem nunca montou um ambiente como este conseguirá acompanhar.

## Preparação do ambiente

O primeiro passo é criar a estrutura de pastas onde vamos organizar tudo.

Execute no seu Linux:

```
mkdir -p ~/pg-stack/monitoring
cd ~/pg-stack
```

Resultado esperado:

```
~/pg-stack/
       |----> monitoring/
```

Tudo será configurado dentro dessa pasta.

## Criando o arquivo docker-compose.yml

Agora dentro da pasta pg-stack, crie o arquivo:

```
vi docker-compose.yml
```

Cole o conteúdo completo:

```
services:
  postgres:
    image: postgres:16
    container_name: postgres16
    restart: always
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=SenhaForteAqui
      - POSTGRES_DB=labdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin4
    restart: always
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@example.com
      - PGADMIN_DEFAULT_PASSWORD=OutraSenhaForte
    ports:
      - "8080:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      - postgres

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    container_name: pg_exporter
    restart: always
    environment:
      DATA_SOURCE_NAME: "postgresql://admin:SenhaForteAqui@postgres16:5432/labdb?sslmode=disable"
    ports:
      - "9187:9187"
    depends_on:
      - postgres

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    ports:
      - "9090:9090"
    depends_on:
      - postgres_exporter

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  pgdata:
  pgadmin_data:
  grafana_data:
```

Salve e saia.

## Criando o arquivo de configuração do Prometheus

Entre na pasta de monitoramento:

```
cd monitoring
vi prometheus.yml
```

Adicione:

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgres'
    static_configs:
      - targets: ['pg_exporter:9187']
```

Depois:

```
cd ..
```

## Subindo todo o ambiente

Dentro da pasta pg-stack:

```
docker compose up -d
```![](image-2.png)

Verifique:

```
docker compose ps
```![](image-3-1024x83.png)

Resultado esperado:

```
postgres16        Up
pgadmin4          Up
pg_exporter       Up
prometheus        Up
grafana           Up
```

## Acessos aos serviços

### PostgreSQL

Porta: 5432  
Usuário: admin

### PgAdmin

http://IP_DO_SERVIDOR:8080  
Email: admin@example.com  
Senha: OutraSenhaForte

### Prometheus

http://IP_DO_SERVIDOR:9090

### Grafana

http://IP_DO_SERVIDOR:3000  
Usuário: admin  
Senha: admin

### Métricas do Exporter

http://IP_DO_SERVIDOR:9187/metrics

## Importando o dashboard de PostgreSQL no Grafana

No Grafana:

1. Dashboards
2. Import
3. Insira o ID **9628**
4. Selecione Prometheus como datasource
5. Importar

![](WhatsApp-Image-2025-11-15-at-18.05.40_79c5c75d-1-1024x513.jpg)

Esse dashboard traz métricas detalhadas de:

- conexões
- autovacuum
- deadlocks
- WAL
- growth de tabelas
- seq scan vs index scan
- hit ratio
- I/O
- locks
- tempo de queries

É um dos modelos mais completos para PostgreSQL + Prometheus.

## Fluxo do ambiente

1. PostgreSQL gera estatísticas internas.
2. Postgres Exporter expõe as métricas.
3. Prometheus coleta as métricas.
4. Grafana exibe as visualizações.
5. Docker Compose orquestra tudo, garantindo rede interna e persistência.

Com Docker e Docker Compose é possível montar um ambiente completo de banco de dados com administração, métricas e visualização em minutos, de forma padronizada e reprodutível. Esse tipo de stack é amplamente utilizado em times de Database Engineering, SRE e Observabilidade.

![](ChatGPT-Image-Oct-4-2025-09_57_50-AM.png)
