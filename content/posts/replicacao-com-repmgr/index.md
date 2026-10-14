---
title: "Replicação com REPMGR"
date: 2025-10-04T13:00:29-03:00
draft: false
categories: ["PostgreSQL", "Replicação"]
tags: ["Postgre SQL"]
cover:
  image: "ChatGPT-Image-Oct-6-2025-10_53_18-AM.png"
---

Em cenários onde cada segundo é crucial, uma interrupção inesperada pode ter um impacto significativo nos negócios. Ninguém deseja ser pego de surpresa, não é mesmo? É aí que entra o Repmgr, o super-herói dos clusters de servidores PostgreSQL. Trata-se de um conjunto de ferramentas de código aberto que simplifica a administração de replicação e torna os failovers uma tarefa mais gerenciável.

Em poucas palavras, o Repmgr é a carta na manga para manter os servidores PostgreSQL sempre em funcionamento, garantindo que os negócios continuem sem interrupções!

**Pré-requisitos para seguir este tutorial:**

- Sistema operacional Linux CentOS 8 ou superior.
- PostgreSQL 13 instalado em dois ambientes.

Como configurar? Vamos lá

1. Instalando o REPMGR no servidor Prmary e Standby

No servidor primary, instale o REPMGR

```sql
yum -y install repmgr13*
```

No servidor standby

```sql
yum -y install repmgr13*
```

**2. Configuração e Parametrização dos Ambientes:**

Nesta fase, procederemos com a configuração dos parâmetros essenciais para a replicação no PostgreSQL.

No arquivo de configuração do PostgreSQL, localizado em /var/lib/pgsql/13/data/postgresql.conf, realize as seguintes edições nos parâmetros indicados:

```sql
archive_command = 'cp %p /var/lib/pgsql/archive/%'
archive_mode = on
hot_standby = on
listen_addresses = '*'
max_replication_slots = 10
max_wal_senders = 5
shared_preload_libraries = 'repmgr'
wal_level = replica 
```

Estas modificações visam otimizar o ambiente para suportar a replicação eficiente no PostgreSQL. Lembre-se de reiniciar o PostgreSQL para aplicar as alterações.

Prossigamos para a próxima etapa assim que essas configurações estiverem concluídas e verificadas.

**3. Criação do Usuário para Gerenciamento da Replicação:**

No ambiente PRIMARY, é necessário criar um usuário e conceder permissões de superusuário para gerenciar a replicação, neste caso, o usuário repmgr.

**3.1. Criando o Usuário:**

Utilize o seguinte comando SQL para criar o usuário repmgr:

```sql
postgres=# create user repmgr;

postgres=# create database repmgr owner repmgr; 
```

**3.2. Concedendo Permissões de Superusuário:**

Após criar o usuário, conceda as permissões necessárias:

```sql
postgres=# alter user repmgr with superuser
```

Volte para o arquivo pg_hba.conf e conceda acesso ao usuário.

Por motivos de estudo ou laboratório, criarei o usuário com permissões de acesso de qualquer local. Evite realizar esta ação em ambientes produtivos!

Acesse o arquivo em /var/lib/pgsql/13/data/pg_hba.conf

adicione a seguinte linha de configuração

```
host    repmgr          repmgr          0.0.0.0/0               trust
host    replication     repmgr          <ip_replica>/32         trust
```

**4. Configuração do Arquivo REPMGR.CONF:**

A próxima etapa é a criação do arquivo repmgr.conf no nó primário. Embora possa ser criado em qualquer local do sistema operacional, optei por colocá-lo no meu diretório de dados do PostgreSQL, consolidando meus arquivos de configuração em um único local.

Vamos definir os parâmetros mínimos para garantir o funcionamento, lembrando que este arquivo de configuração pode ser mais extenso conforme necessário.

**Diretório de Dados:** /var/lib/pgsql/13/data/

```sql
touch repmgr.conf

vi repmgr.conf
```

- Dentro deste arquivo, iremos editar as seguintes configurações.

```sql
node_id=1

node_name=Master

conninfo='user=repmgr password=1 host=192.168.0.106 dbname=repmgr port=5432 connect_timeout=2'

data_directory='/var/lib/pgsql/13/data/'

failover=automatic

promote_command='/usr/pgsql-13/bin/repmgr standby promote -f /var/lib/pgsql/13/data/repmgr.conf --log-to-file'

follow_command='/usr/pgsql-13/bin/repmgr standby follow -f /var/lib/pgsql/13/data/repmgr.conf --log-to-file --upstream-node-id=%n'
```

**5. Registrando o Servidor como Primary (Master):**

Perfeito! Após realizar todas as configurações iniciais, vamos indicar ao nosso PostgreSQL quem está no controle, estabelecendo claramente qual é o nosso banco principal.

```sql
/usr/pgsql-13/bin/repmgr  -f  /var/lib/pgsql/13/data/repmgr.conf  primary register
```

funcionando!

**6. Configuração do REPMGR.CONF no Servidor Standby:**

Proceda à criação do arquivo repmgr.conf no servidor standby (Passo 4).

No entanto, ao contrário do Master, incluiremos as seguintes configurações.

```sql
 node_id=2

node_name=Standby

conninfo='user=repmgr password=1 host=192.168.0.107 dbname=repmgr port=5432 connect_timeout=2'

data_directory='/var/lib/pgsql/13/data/'

failover=automatic

promote_command='/usr/pgsql-13/bin/repmgr standby promote -f /var/lib/pgsql/13/data/repmgr.conf --log-to-file'

follow_command='/usr/pgsql-13/bin/repmgr standby follow -f /var/lib/pgsql/13/data/repmgr.conf --log-to-file --upstream-node-id=%n'
```

**7. Registrando o Servidor Standby:**

Agora que configuramos o servidor principal, é hora de registrar o servidor standby para receber os dados do master.

Inicialmente, realizaremos o clone entre os ambientes.

```sql
/usr/pgsql-13/bin/repmgr -h 192.168.0.106 -U repmgr -d repmgr -f /var/lib/pgsql/13/data/repmgr.conf standby clone -F
```

Registre o seu servidor standby para informar ao PostgreSQL quem ele é.

```sql
/usr/pgsql-13/bin/repmgr -h 192.168.0.106 -U repmgr -d repmgr -f /var/lib/pgsql/13/data/repmgr.conf standby register -F
```

No cluster MASTER, verifique se as alterações foram devidamente aceitas.

```sql
/usr/pgsql-13/bin/repmgr  -f  /var/lib/pgsql/13/data/repmgr.conf cluster  show
```

E aqui está: nossos bancos de dados MASTER e STANDBY estão em comunicação.

Para demonstrar, efetuei o acesso em ambos os ambientes. No servidor master, criei um banco de dados chamado GUSTAVO e executei a consulta correspondente no servidor STANDBY.

Em resumo, agora estamos com a replicação do PostgreSQL funcionando redondinha! Desde a instalação do Repmgr até a criação de bancos e consultas, cobrimos tudo. Fica claro que o Repmgr é um aliado poderoso para garantir que nossos dados estejam sempre disponíveis e em sintonia. Então, se a ideia é otimizar o desempenho e a confiabilidade do PostgreSQL, o Repmgr é o caminho certo. É mais do que uma ferramenta, é a tranquilidade de saber que nossos sistemas estão rodando de forma segura.
