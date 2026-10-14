---
title: "Projeto 6/10 – Montando um cluster PostgreSQL HA com Patroni, etcd e HAProxy em Docker"
date: 2026-01-26T07:00:00-03:00
draft: false
categories: ["PostgreSQL", "Replicação", "Scripts & Automação"]
tags: ["Postgre SQL", "Projetos"]
cover:
  image: "projeto6_2.png"
---

Este projeto monta um **cluster PostgreSQL altamente disponível** usando três componentes:

- **etcd** como DCS (Distributed Configuration Store)
- **Patroni** controlando bootstrap, failover automático, réplica, saúde, etc.
- **HAProxy** expondo um endpoint único para a aplicação

O objetivo é passar por uma experiência real de cluster HA sem adivinhações: apagar, copiar e colar, subir e validar.

# Limpar o ambiente anterior

Se você já testou o projetos antes, limpe tudo para evitar conflitos.

Esse passo garante que o laboratório suba do zero exatamente como planejado.

```
cd /home/lab06/projeto06 2>/dev/null || true

if [ -f docker-compose.yml ]; then
  docker compose down -v
fi

rm -rf /home/lab06/projeto06
mkdir -p /home/lab06/projeto06
cd /home/lab06/projeto06
```

# Visão geral do ambiente

Este cluster contém 4 serviços:

### etcd

Armazena estado do cluster, líder, configurações e controla o DCS utilizado pelo Patroni.

### Patroni (node2 -> primário)

É o nó responsável pelo bootstrap inicial.

Ele cria o cluster dentro do etcd e inicia o primeiro servidor PostgreSQL como líder.

### Patroni (node1 -> standby)

Esse nó não faz bootstrap, ele **clona automaticamente do líder** seguindo o estado do etcd.

### HAProxy

Entrega um único endpoint na porta **5000**.

A aplicação sempre conecta nesse endpoint, independente de qual nó esteja líder.

# Dockerfile – imagem PostgreSQL 16 + Patroni

Aqui você cria a imagem base que roda Postgres + Patroni com suporte a etcd.

```
cd /home/lab06/projeto06

cat &lt;&lt; 'EOF' > Dockerfile
FROM postgres:16

USER root
RUN apt-get update &amp;&amp; \\
    apt-get install -y python3-pip &amp;&amp; \\
    pip3 install --break-system-packages psycopg2-binary "patroni[etcd]" &amp;&amp; \\
    rm -rf /var/lib/apt/lists/*

USER postgres

EXPOSE 5432 8008

ENTRYPOINT ["patroni"]
CMD ["/etc/patroni.yml"]
EOF
```

# Configurações do Patroni

Crie o diretório de configuração:

```
mkdir -p patroni
```

Cada nó tem sua configuração individual.

O node2 faz o bootstrap.

O node1 só replica.

# Configuração do patroni-node2 (primário)

Este é o **nó que cria o cluster**.

Somente ele possui o bloco bootstrap:.

```
cat << 'EOF' > patroni/patroni-node2.yml
scope: pg-cluster
name: node2

restapi:
  listen: 0.0.0.0:8008
  connect_address: patroni-node2:8008

etcd:
  host: etcd:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 256MB
        wal_level: replica
        hot_standby: "on"
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: patroni-node2:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: replpass
    superuser:
      username: postgres
      password: postgrespass
  parameters:
    wal_keep_size: 64MB
    max_wal_senders: 10
    max_replication_slots: 10
  pg_hba:
    - local   all             all                                   trust
    - local   replication     all                                   trust
    - host    all             all             0.0.0.0/0             trust
    - host    replication     replicator      0.0.0.0/0             trust
EOF
```

# Configuração do patroni-node1 (standby)

Esse nó **não cria nada**. Ele apenas replica do líder.

Aqui está a correção mais crítica: o data_dir fica em um subdiretório /pgdata para evitar permissões ruins herdadas da imagem.

```
cat << 'EOF' > patroni/patroni-node1.yml
scope: pg-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: patroni-node1:8008

etcd:
  host: etcd:2379

postgresql:
  listen: 0.0.0.0:5432
  connect_address: patroni-node1:5432
  data_dir: /var/lib/postgresql/data/pgdata
  authentication:
    replication:
      username: replicator
      password: replpass
    superuser:
      username: postgres
      password: postgrespass
  parameters:
    wal_keep_size: 64MB
    max_wal_senders: 10
    max_replication_slots: 10
  pg_hba:
    - local   all             all                                   trust
    - local   replication     all                                   trust
    - host    all             all             0.0.0.0/0             trust
    - host    replication     replicator      0.0.0.0/0             trust
EOF
```

# Configuração do HAProxy

Este arquivo cria:

- proxy TCP na porta **5000**
- painel web na porta **7000**
- healthcheck usando HTTP do Patroni

```
mkdir -p haproxy
```
```
cat << 'EOF' > haproxy/haproxy.cfg
global
    maxconn 100

defaults
    log global
    mode tcp
    timeout client  30s
    timeout connect 4s
    timeout server  30s

frontend pgsql
    bind *:5000
    default_backend pgsql_back

backend pgsql_back
    option httpchk
    http-check expect status 200
    server node1 patroni-node1:5432 check port 8008
    server node2 patroni-node2:5432 check port 8008 backup

frontend stats
    bind *:7000
    mode http
    stats uri /stats
    stats refresh 10s
EOF
```

# Docker Compose do cluster

O etcd precisa do parâmetro --enable-v2=true.

Sem isso, o Patroni trava em “waiting on etcd”.

```
cat << 'EOF' > docker-compose.yml
services:

  etcd:
    image: quay.io/coreos/etcd:v3.5.15
    container_name: etcd
    command: >
      /usr/local/bin/etcd
      --data-dir=/etcd-data
      --name=etcd0
      --listen-client-urls=http://0.0.0.0:2379
      --advertise-client-urls=http://etcd:2379
      --listen-peer-urls=http://0.0.0.0:2380
      --initial-advertise-peer-urls=http://etcd:2380
      --initial-cluster=etcd0=http://etcd:2380
      --initial-cluster-token=etcd-cluster-1
      --initial-cluster-state=new
      --enable-v2=true
    ports:
      - "2379:2379"
    networks:
      - patroni-net

  patroni-node1:
    build:
      context: .
      dockerfile: ./Dockerfile
    container_name: patroni-node1
    volumes:
      - ./patroni/patroni-node1.yml:/etc/patroni.yml
    networks:
      - patroni-net
    depends_on:
      - etcd

  patroni-node2:
    build:
      context: .
      dockerfile: ./Dockerfile
    container_name: patroni-node2
    volumes:
      - ./patroni/patroni-node2.yml:/etc/patroni.yml
    networks:
      - patroni-net
    depends_on:
      - etcd

  haproxy:
    image: haproxy:2.9
    container_name: haproxy
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - "5000:5000"
      - "7000:7000"
    networks:
      - patroni-net
    depends_on:
      - patroni-node1
      - patroni-node2

networks:
  patroni-net:
    driver: bridge
EOF
```

# Subindo o cluster

```
cd /home/lab06/projeto06

docker compose build --no-cache
docker compose up -d

docker ps
```

Você deve ver:

- etcd
- patroni-node1
- patroni-node2
- haproxy

Se ambos os Patronis estiverem "Up", tudo subiu corretamente.

# Validando quem é primário e standby

```
docker exec -it patroni-node2 psql -U postgres -c "SELECT inet_server_addr(), pg_is_in_recovery();"

docker exec -it patroni-node1 psql -U postgres -c "SELECT inet_server_addr(), pg_is_in_recovery();"
```

Esperado:

- node2 -> f (primário)
- node1 -> t (standby)

Se o node1 der erro de “socket missing”, quase sempre é data_dir errado.

# Confirmando a replicação

No primário:

```
docker exec -it patroni-node2 psql -U postgres -c "
  SELECT pid, state, client_addr, sync_state
  FROM pg_stat_replication;
"
```

Se apareceu uma linha, a réplica está conectada.

# Testando a replicação física

No primário:

```
docker exec -it patroni-node2 psql -U postgres -c "
  DROP TABLE IF EXISTS teste_ha;
  CREATE TABLE teste_ha (id int, valor text);
  INSERT INTO teste_ha VALUES (1, 'ok');
"
```

No standby:

```
docker exec -it patroni-node1 psql -U postgres -c "SELECT * FROM teste_ha;"<br><br>
```

Se retornou 1 | ok, está perfeito.

# Testando via HAProxy

```
docker exec -it patroni-node2 bash -c '
  PGPASSWORD=postgrespass psql -h haproxy -p 5000 -U postgres -c "
    SELECT inet_server_addr(), pg_is_in_recovery();
  "
'
```

O HAProxy sempre manda para o líder atual.

# Simulando Failover

Derrube o líder:

```
docker stop patroni-node2
```

Após alguns segundos, o node1 assume.

Confirme:

```
docker exec -it patroni-node1 psql -U postgres -c "
  SELECT inet_server_addr(), pg_is_in_recovery();
"
```

Via HAProxy:

```
docker exec -it patroni-node1 bash -c '
  PGPASSWORD=postgrespass psql -h haproxy -p 5000 -U postgres -c "
    SELECT inet_server_addr(), pg_is_in_recovery();
  "
'
```

Agora o endpoint aponta para o novo primário sem mudança na aplicação.

Pronto!

Este laboratório entrega uma experiência completa e realista de **alta disponibilidade com PostgreSQL**, usando exatamente os mesmos componentes adotados em ambientes corporativos modernos. O Patroni cuida do ciclo de vida do cluster, o etcd mantém o estado distribuído e o HAProxy garante um ponto único de acesso permitindo failover transparente para a aplicação.

Com poucos comandos, você sobe, testa, valida e simula cenários críticos como promoção automática de réplicas.

A grande vantagem deste projeto é que ele mostra, de forma prática, como funciona um cluster HA de verdade: líder, réplica, health checks, failover, recovery e roteamento inteligente.
