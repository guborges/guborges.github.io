---
title: "Projeto 5/10 – Replicação Streaming do PostgreSQL (Primary e Replica)"
date: 2026-01-07T07:00:00-03:00
draft: false
categories: ["PostgreSQL", "Replicação", "Scripts & Automação"]
tags: ["Postgre SQL", "Projetos"]
cover:
  image: "projeto5_ge.png"
---

Este é o quinto projeto da série de dez exercícios práticos voltados para Database Engineering. Depois de trabalhar com automação, monitoramento, backup e tuning, o próximo passo é alta disponibilidade.

Aqui montei, usando apenas Docker, um ambiente simples de replicação física em streaming entre dois servidores PostgreSQL:

- um servidor primary
- um servidor standby (replica)

A ideia é que você consiga reproduzir tudo apenas copiando e colando os comandos, sem precisar ter psql instalado no host – só Docker.

Objetivos do projeto

- Subir dois servidores PostgreSQL: primary e replica
- Habilitar os parâmetros necessários para streaming replication
- Criar o usuário de replicação
- Gerar um base backup do primary para inicializar o standby
- Ajustar o primary_conninfo no base backup
- Liberar a replicação no pg_hba.conf do primary
- Validar a replicação em tempo real
- Promover a réplica a primary usando pg_ctl no usuário correto dentro do container

Arquitetura do ambiente

Todo o projeto fica em:

```text
/home/lab05/projetos-dbengine/projeto05
```

Estrutura de diretórios:

```text
projeto05/
├── primary/
│   ├── docker-compose.yml
│   └── init.sql
├── replica/
│   └── docker-compose.yml
└── basebackup/
```

Os dois serviços rodam em containers separados:

- pg_primary exposto na porta 5433 do host
- pg_replica exposto na porta 5434 do host

Ambos compartilham uma rede Docker chamada pg_net, para que a réplica enxergue o primary pelo hostname pg_primary.

Criando diretórios e rede Docker

Crie a estrutura do projeto e a rede:

```text
mkdir -p /home/lab05/projetos-dbengine/projeto05
cd /home/lab05/projetos-dbengine/projeto05

mkdir -p primary replica basebackup

docker network create pg_net
```

A rede pg_net será usada nos dois docker-compose.yml.

Configurando o Primary

O primary precisa:

- Usar um wal_level compatível com replicação
- Ter max_wal_senders maior que 0
- Manter WAL suficiente em disco (wal_keep_size)
- Escutar na rede (listen_addresses=*)
- Criar o usuário replicator com permissão de replicação

Arquivo primary/docker-compose.yml

```text
cat << 'EOF' > primary/docker-compose.yml
services:
  pg_primary:
    image: postgres:16
    container_name: pg_primary
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=Senha123
      - POSTGRES_DB=appdb
      - POSTGRES_HOST_AUTH_METHOD=trust
    ports:
      - "5433:5432"
    volumes:
      - ./primary_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - ../basebackup:/basebackup
    command:
      - "postgres"
      - "-c"
      - "wal_level=replica"
      - "-c"
      - "max_wal_senders=10"
      - "-c"
      - "wal_keep_size=64MB"
      - "-c"
      - "listen_addresses=*"
    networks:
      - pg_net

networks:
  pg_net:
    external: true
EOF
```

Pontos importantes:

- POSTGRES_HOST_AUTH_METHOD=trust: simplifica o acesso no laboratório
- Volume ../basebackup:/basebackup: mapeia o diretório de base backup para dentro do container
- Parâmetros:
  - wal_level=replica: habilita informações necessárias para replicação física
  - max_wal_senders=10: permite até 10 processos enviando WAL
  - wal_keep_size=64MB: garantia mínima de WAL mantido
  - listen_addresses=*: escuta em todas as interfaces
- Uso da rede externa pg_net para falar com a futura réplica

Arquivo primary/init.sql

Esse script:

- Cria o usuário de replicação
- Cria uma tabela de teste
- Insere uma linha inicial para validar replicação depois

```text
cat << 'EOF' > primary/init.sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica123';

CREATE TABLE IF NOT EXISTS teste (
    id   integer PRIMARY KEY,
    txt  text
);

INSERT INTO teste (id, txt) VALUES (1, 'linha inicial no primary');
EOF
```

Subindo o Primary

```text
cd /home/lab05/projetos-dbengine/projeto05/primary
docker compose up -d
```

Verifique se está rodando:

```text
docker ps | grep pg_primary
```

Teste rápido de conexão (usando psql dentro do container):

```text
docker exec -it pg_primary \\
  psql -U admin -d appdb -c "SELECT version();"
```

Gerando o Base Backup do Primary

Agora vamos criar uma cópia física do cluster do primary para usar como diretório de dados da réplica.

Esse comando roda dentro do pg_primary e grava o base backup em /basebackup/replica_data (que é mapeado para o host em basebackup/replica_data):

```text
docker exec -it pg_primary \\
  pg_basebackup \\
    -D /basebackup/replica_data \\
    -Fp \\
    -Xs \\
    -P \\
    -R \\
    -U replicator
```

O que cada opção faz:

- D /basebackup/replica_data: destino do base backup
- Fp: formato “plain” (cópia de diretório)
- Xs: inclui os WAL necessários via streaming
- P: mostra progresso
- R: cria:
  - standby.signal : faz o Postgres subir em modo standby
  - postgresql.auto.conf com primary_conninfo

Do lado do host, você pode inspecionar:

```text
ls -R /home/lab05/projetos-dbengine/projeto05/basebackup
```

Ajustando o primary_conninfo no base backup (no host)

O -R ajuda, mas a linha gerada de primary_conninfo pode não estar com o host e a senha do jeito que queremos. Vamos ajustar diretamente no arquivo do base backup, ainda no host, antes de subir o container da réplica.

Exiba a linha atual:

```text
cd /home/lab05/projetos-dbengine/projeto05

grep primary_conninfo basebackup/replica_data/postgresql.auto.conf
```

Agora sobrescreva para uma versão explícita apontando para o hostname pg_primary:

```text
sed -i "s/^primary_conninfo.*/primary_conninfo = 'host=pg_primary port=5432 user=replicator password=replica123'/" basebackup/replica_data/postgresql.auto.conf
```

E confira:

```text
grep primary_conninfo basebackup/replica_data/postgresql.auto.conf
```

Deve aparecer:

```text
primary_conninfo = 'host=pg_primary port=5432 user=replicator password=replica123'
```

Isso garante que, quando o container pg_replica subir usando esse diretório, ele já sabe exatamente como conectar no primary para puxar os WAL.

Configurando a Replica

Agora sim, com o base backup preparado e o primary_conninfo corrigido, vamos configurar a réplica.

Arquivo replica/docker-compose.yml

```text
cat << 'EOF' > replica/docker-compose.yml
services:
  pg_replica:
    image: postgres:16
    container_name: pg_replica
    environment:
      - POSTGRES_HOST_AUTH_METHOD=trust
    ports:
      - "5434:5432"
    volumes:
      - ../basebackup/replica_data:/var/lib/postgresql/data
    networks:
      - pg_net

networks:
  pg_net:
    external: true
EOF
```

Aqui:

- O volume ../basebackup/replica_data vira o PGDATA da réplica
- Esse diretório já tem:
  - standby.signal → ativa modo standby
  - postgresql.auto.conf com primary_conninfo apontando para pg_primary

Subindo a Replica

```text
cd /home/lab05/projetos-dbengine/projeto05/replica
docker compose up -d
```

Verifique:

```text
docker ps | grep pg_replica
```

Confirme se está em recovery:

```text
docker exec -it pg_replica \\
  psql -U admin -d appdb -c "SELECT pg_is_in_recovery();"
```

Se retornar t, a réplica está em modo standby esperando WAL do primary.

Liberando replicação no pg_hba.conf do primary

Agora falta dizer para o primary que o usuário replicator pode se conectar para fazer replicação.

Primeiro, descubra o PGDATA no primary:

```text
docker exec -it pg_primary bash -c 'echo $PGDATA'
```

Geralmente será /var/lib/postgresql/data.

Adicione uma linha liberando replicação de qualquer IP (ambiente de laboratório):

```text
docker exec -it pg_primary bash -c "echo \\"host    replication     replicator      0.0.0.0/0       trust\\" >> \\$PGDATA/pg_hba.conf"
```

Confirme no final do arquivo:

```text
docker exec -it pg_primary bash -c "tail -n 5 \\$PGDATA/pg_hba.conf"
```

Recarregue a configuração:

```text
docker exec -it pg_primary \\
  psql -U admin -d appdb -c "SELECT pg_reload_conf();"
```

Esse passo é essencial: sem ele, a réplica não consegue abrir a conexão de replicação, mesmo com primary_conninfo correto.

Verificando o wal receiver na Replica

Verifique se o processo responsável por receber WAL está ativo no standby:

```text
docker exec -it pg_replica \\
  psql -U admin -d appdb -c "SELECT status, conninfo FROM pg_stat_wal_receiver;"
```

Saída esperada (como você já viu):

```text
 status   | conninfo
----------+------------------------------------------------------------------
 streaming | user=replicator ... host=pg_primary port=5432 ...
```

Isso confirma que a réplica está conectada no primary e recebendo WAL.

Verificando sessões de replicação no Primary

No primary, veja se ele enxerga a réplica como cliente de replicação:

```text
docker exec -it pg_primary \\
  psql -U admin -d appdb -c "
SELECT pid,
       usename,
       application_name,
       state,
       sync_state
FROM pg_stat_replication;
"
```

Esperado:

- Uma linha com:
  - usename = replicator
  - state = streaming

Se isso apareceu, a replicação física está estabelecida.

Testando a Replicação de Dados

Agora vamos validar que as alterações no primary chegam na réplica.

Inserindo dados no primary

```text
docker exec -it pg_primary \\
  psql -U admin -d appdb -c "INSERT INTO teste VALUES (2, 'linha replicada para o standby');"
```

E mais uma linha:

```text
docker exec -it pg_primary \\
  psql -U admin -d appdb -c "INSERT INTO teste VALUES (3, 'teste depois de configurar pg_hba.conf');"
```

Consultando na réplica

```text
docker exec -it pg_replica \\
  psql -U admin -d appdb -c "SELECT * FROM teste ORDER BY id;"
```

Saída esperada:

```text
 id |                           txt
----+-------------------------------------------------
  1 | linha inicial no primary
  2 | linha replicada para o standby
  3 | teste depois de configurar pg_hba.conf
```

Se essa consulta mostrar as linhas inseridas no primary, a replicação física em streaming está funcionando de ponta a ponta.

Promovendo a Replica a Primary

Para simular falha do primary, podemos promover a réplica para assumir o papel de primary.

Importante: dentro do container, o Postgres roda como usuário postgres, então o pg_ctl precisa ser chamado com esse usuário.

Comando de promoção

```text
docker exec -u postgres pg_replica pg_ctl -D /var/lib/postgresql/data promote
```

Verificando o estado após promoção

```text
docker exec -it pg_replica \\
  psql -U admin -d appdb -c "SELECT pg_is_in_recovery();"
```

Esperado:

```text
f
```

A antiga réplica agora:

- Virou primary
- Parou de aplicar WAL vindo do antigo primary
- Passou a aceitar escrita diretamente

Neste projeto, montei e validei um ambiente completo de replicação física streaming do PostgreSQL em Docker, com uma sequência totalmente reprodutível:

- Organização do projeto em /home/lab05/projetos-dbengine/projeto05
- Configuração do primary com parâmetros de WAL e usuário de replicação
- Geração de base backup com pg_basebackup direto no container
- Ajuste do primary_conninfo no base backup apontando para pg_primary
- Configuração da réplica usando esse diretório como PGDATA
- Liberação de acesso de replicação no pg_hba.conf do primary
- Validação da replicação via pg_stat_wal_receiver, pg_stat_replication e consultas na tabela de teste
- Promoção da réplica utilizando pg_ctl com o usuário postgres

Esse setup manual mostra exatamente o que ferramentas de HA (como Patroni, Stolon, etc.) automatizam em produção.
