---
title: "Projeto 7/10 – Capacity Planning de PostgreSQL com Prometheus, Exporters e cAdvisor"
date: 2026-02-16T07:00:00-03:00
draft: false
categories: ["Linux & Infra", "PostgreSQL", "Scripts & Automação"]
tags: ["Postgre SQL", "Projetos"]
cover:
  image: "projeto7_3.png"
---

Este é o sétimo laboratório da série de 10 projetos práticos de Database Engineering.

O objetivo é montar um ambiente completo capaz de coletar métricas reais de PostgreSQL e dos containers para análise de capacidade: CPU, memória, conexões, tamanho do banco.

Tudo funciona apenas copiando e colando os comandos abaixo.

# Objetivo do projeto

- Subir um ambiente PostgreSQL com monitoramento via Prometheus
- Coletar métricas de containers (CPU / RAM) via **cAdvisor**
- Coletar métricas internas do PostgreSQL via **postgres_exporter**
- Coletar métricas de host (CPU / RAM / disco) via **node_exporter**
- Organizar consultas PromQL
- Visualizar métricas no Prometheus
- Capturar dados reais para capacity planning
- Gerar curvas de CPU, memória, conexões e crescimento da base

# Estrutura do laboratório

Execute:

```
mkdir -p /home/lab07/projeto07/prometheus
mkdir -p /home/lab07/projeto07/queries
mkdir -p /home/lab07/projeto07/notebooks
```

Estrutura final:

```
/home/lab07/projeto07
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── queries/
│   ├── cpu.promql
│   ├── memory.promql
│   ├── connections.promql
│   ├── wal.promql
│   ├── size.promql
└── notebooks/
    └── analise-capacidade.ipynb (opcional)
```

# docker-compose.yml completo

```
cat << 'EOF' > /home/lab07/projeto07/docker-compose.yml
version: '3.8'

services:

  postgres:
    image: postgres:16
    container_name: pg_primary
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: SenhaForte123
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - ./pgdata:/var/lib/postgresql/data

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    pid: "host"
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro'
    ports:
      - "9100:9100"
    restart: always

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres_exporter
    environment:
      DATA_SOURCE_NAME: "postgres://admin:SenhaForte123@postgres:5432/appdb?sslmode=disable"
    ports:
      - "9187:9187"
    depends_on:
      - postgres

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: always

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - postgres_exporter
      - node_exporter
      - cadvisor
EOF
```

# Configuração do Prometheus

```
cat << 'EOF' > /home/lab07/projeto07/prometheus/prometheus.yml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'postgres_exporter'
    static_configs:
      - targets: ['postgres_exporter:9187']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
EOF
```

# Subindo o laboratório

```
cd /home/lab07/projeto07
docker compose up -d
docker ps
```

Acesse:

- Prometheus → http://localhost:9090
- cAdvisor → http://localhost:8080

# Criando os arquivos PromQL

## CPU

```
cat << 'EOF' > /home/lab07/projeto07/queries/cpu.promql
rate(container_cpu_usage_seconds_total{container_label_com_docker_compose_service="postgres"}[5m])
EOF
```

## Memória

```
cat << 'EOF' > /home/lab07/projeto07/queries/memory.promql
container_memory_usage_bytes{container_label_com_docker_compose_service="postgres"}
EOF
```

## Conexões simultâneas

```
cat << 'EOF' > /home/lab07/projeto07/queries/connections.promql
pg_stat_activity_count
EOF
```

## Tamanho do banco

```
cat << 'EOF' > /home/lab07/projeto07/queries/size.promql
pg_database_size_bytes{datname="appdb"}
EOF
```

# Validando as métricas no Prometheus

Abra:

**http://localhost:9090**

No menu superior clique em **Query**.

## Verificando se todos os serviços estão UP

Digite:

```
up
```

Você deve ver:

![](image-20.png)

# Coletando as métricas

## CPU do PostgreSQL

```
rate(container_cpu_usage_seconds_total{container_label_com_docker_compose_service="postgres"}[5m])
```

**Graph →** mostra curva de uso de CPU do container.

![](image-21-1024x477.png)
![](image-25.png)

## Memória do PostgreSQL

```
container_memory_usage_bytes{container_label_com_docker_compose_service="postgres"}
```![](image-22-1024x469.png)
![](image-26.png)

## Conexões simultâneas

```
pg_stat_activity_count
```![](image-23-1024x481.png)

Você pode abrir várias sessões para ver o número subindo:

```
docker exec -it pg_primary psql -U admin -d appdb
```

## Tamanho do banco

```
pg_database_size_bytes{datname="appdb"}
```![](image-24-1024x486.png)

Com o tempo, você vê a linha crescendo no gráfico.

Exemplo típico:

```
dia 1 → 12.3 GB
dia 2 → 12.5 GB
dia 3 → 12.9 GB
dia 4 → 13.1 GB
```

Crescimento médio: **+0,25 GB/dia**

# Criando carga para deixar os gráficos bonitos

```
docker exec -it pg_primary psql -U admin -d appdb
```

Dentro do SQL:

```
CREATE TABLE teste(id serial, nome text);
INSERT INTO teste(nome)
SELECT md5(random()::text)
FROM generate_series(1,100000);
```

Resultados imediatos:

- CPU sobe
- WAL dispara
- tamanho do banco aumenta
- cAdvisor reflete o uso do container

Ótimo para tirar imagens do artigo.

# Como usar isso para Capacity Planning

Com dias ou semanas de coleta, você consegue:

- **CPU**
  - identificar horários críticos
  - detectar tendência de aumento
- **Memória**
  - saber se o container precisa de mais RAM
- **Conexões**
  - prever necessidade de pgBouncer
- **Tamanho da base**
  - calcular projeções de 3/6/12 meses
- **WAL**
  - ajustar retenção
  - prever impacto em replicação

Exemplo de projeção baseada no crescimento diário:

- 3 meses → +22 GB
- 6 meses → +45 GB
- 12 meses → +90 GB

Este projeto entrega tudo o que você precisa para montar uma pipeline de métricas para PostgreSQL:

- PostgreSQL
- Prometheus
- Node Exporter
- Postgres Exporter
- cAdvisor
- Consultas PromQL essenciais

Este laboratório montou a base completa de observabilidade necessária para entender de forma clara como um PostgreSQL se comporta no dia a dia. Com PostgreSQL, Prometheus, Node Exporter, Postgres Exporter e cAdvisor trabalhando juntos, agora é possível visualizar CPU, memória, conexões e crescimento da base de forma totalmente transparente.

A partir daqui, qualquer análise de capacidade deixa de ser “achismo” e passa a ser feita com dados reais.  
Com algumas horas de coleta já é possível identificar padrões, horários críticos e tendências; com alguns dias ou semanas, é possível projetar o comportamento da carga para os próximos meses.
