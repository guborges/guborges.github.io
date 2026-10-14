---
title: "Projeto 8/10 – Pipeline CI/CD para migrações de banco com Flyway e GitHub Actions"
date: 2026-03-23T07:14:33-03:00
draft: false
categories: ["Linux & Infra", "PostgreSQL", "Scripts & Automação"]
tags: ["Postgre SQL", "Projetos"]
cover:
  image: "projeto8.png"
---

Este é o oitavo laboratório da minha série prática de Database Engineering.

Depois de trabalhar com backup, tuning, monitoração, replicação e alta disponibilidade, agora é a hora de tratar o banco como o que ele realmente é: **código**.

No mundo moderno, nenhuma mudança de schema deve ser feita “na mão”, muito menos direto em produção. Alterações precisam ser versionadas, revisadas, testadas e aplicadas sempre pelo mesmo processo previsível, auditável e repetível. Para isso, usei **Flyway** como ferramenta de versionamento e **GitHub Actions** como pipeline de CI/CD.

Este projeto cria um fluxo completo:

- Migrações versionadas (V1, V2, V3)
- Banco PostgreSQL efêmero para testes
- Validação automática das migrações
- Execução controlada via pipeline
- Auditoria completa com tabela de histórico
- Automação local + GitHub Actions

A meta não é só aplicar migrações é aplicar **com garantia, controle e governança**, exatamente como times de engenharia maduros fazem.

# **Visão geral da arquitetura**

A estrutura do laboratório segue este formato:

```text
/home/lab08/lab08-cicd-migrations
├── sql/               # Migrações Flyway
├── docker/            # Banco de testes do Flyway
└── .github/workflows/ # Pipeline CI/CD
```

Essa separação mantém tudo organizado:

- **o código do banco fica em sql**,
- **o ambiente de execução fica em docker**,
- **e a automação vive dentro da pasta workflows**.

# **Criando o ambiente completo**

A seguir está a criação completa do projeto em ambiente UBUNTU.

Cada comando faz exatamente uma parte da construção e está explicado antes da execução.

## **Criando a estrutura de diretórios**

Essas pastas são o esqueleto do projeto:

```text
mkdir -p /home/lab08/lab08-cicd-migrations/sql
mkdir -p /home/lab08/lab08-cicd-migrations/docker
mkdir -p /home/lab08/lab08-cicd-migrations/.github/workflows
```

/sql: onde ficam as migrações versionadas.

/docker: onde roda o PostgreSQL temporário.

/workflows: onde mora a pipeline do GitHub Actions.

Separar assim facilita manutenção, testes e CI/CD.

## **Criando as migrações Flyway**

O Flyway usa a convenção:

```text
V1__descricao.sql
V2__descricao.sql
V3__descricao.sql
```

Cada arquivo representa uma evolução incremental no schema.

### **Criando a migração V1 (criação da tabela principal)**

```text
cat << 'EOF' > /home/lab08/lab08-cicd-migrations/sql/V1__create_tables.sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
EOF
```

Essa é a base do sistema: criamos a tabela customers.

### **Criando a migração V2 (índice)**

```text
cat << 'EOF' > /home/lab08/lab08-cicd-migrations/sql/V2__add_indexes.sql
CREATE INDEX idx_customers_email ON customers(email);
EOF
```

Índices são parte crítica da performance e devem ser versionados como qualquer outro objeto.

### **Criando a migração V3 (nova coluna)**

```text
cat << 'EOF' > /home/lab08/lab08-cicd-migrations/sql/V3__add_column_status.sql
ALTER TABLE customers
ADD COLUMN status TEXT DEFAULT 'active';
EOF
```

Aqui evoluímos o schema adicionando um campo sem quebrar compatibilidade.

## **Criando o ambiente de testes com Docker**

Essa etapa cria o PostgreSQL efêmero onde o Flyway vai validar as migrações.

### **Criando o docker-compose.yml**

```text
cat << 'EOF' > /home/lab08/lab08-cicd-migrations/docker/docker-compose.yml
services:
  postgres:
    image: postgres:16
    container_name: pg_flyway
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=Senha123
      - POSTGRES_DB=appdb
    ports:
      - "5432:5432"
EOF
```

Por que isso é importante?

- Evita usar um banco real
- Permite testar migrações sem risco
- Garante que cada execução começa “zerada”
- Simula o mesmo ambiente que o GitHub Actions vai usar

# **Subindo o banco de testes**

Agora vamos subir o PostgreSQL:

```text
cd /home/lab08/lab08-cicd-migrations/docker
docker compose up -d
```

Verificar:

```text
docker ps | grep pg_flyway
```

Se aparecer a porta 5432 mapeada → tudo certo.

# **Instalando o Flyway**

```text
cd /opt
sudo wget <https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.17.0/flyway-commandline-10.17.0-linux-x64.tar.gz>
sudo tar -xvf flyway-commandline-10.17.0-linux-x64.tar.gz
sudo mv flyway-10.17.0 flyway
sudo ln -s /opt/flyway/flyway /usr/local/bin/flyway
```

Teste:

```text
flyway -v
```

# **Executando as migrações (TESTE REAL DO PROJETO)**

Aqui é onde tudo se valida de verdade.

Use sempre caminho absoluto:

```text
flyway \\
  -url=jdbc:postgresql://localhost:5432/appdb \\
  -user=admin \\
  -password=Senha123 \\
  -locations=filesystem:/home/lab08/lab08-cicd-migrations/sql \\
  migrate
```

Se você seguiu tudo certinho, verá:

```text
Successfully validated 3 migrations
Successfully applied 3 migrations
```

Essa é a prova que o projeto está funcionando.

# **Validando no PostgreSQL**

```text
docker exec -it pg_flyway psql -U admin -d appdb -c "\\dt"
docker exec -it pg_flyway psql -U admin -d appdb -c "\\d customers"
```

Você verá:

- Tabela customers
- Tabela flyway_schema_history
- Coluna status

# **Criando o pipeline CI/CD (GitHub Actions)**

Essa etapa deixa tudo automatizado.

Ao enviar uma migração para o GitHub, a pipeline:

- sobe um PostgreSQL 16 temporário
- instala o Flyway
- aplica todas as migrações
- falha se algo estiver errado
- marca a versão como válida

### **Criando o arquivo flyway.yml**

```text
cat << 'EOF' > /home/lab08/lab08-cicd-migrations/.github/workflows/flyway.yml
name: Flyway Migrations

on:
  push:
    branches:
      - main
    paths:
      - "sql/**"

jobs:
  migrate:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        ports:
          - 5432:5432
        env:
          POSTGRES_USER: admin
          POSTGRES_PASSWORD: Senha123
          POSTGRES_DB: appdb
        options: >-
          --health-cmd="pg_isready -U admin"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install Flyway
        run: |
          sudo apt-get update
          sudo apt-get install -y wget tar default-jre
          wget <https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.17.0/flyway-commandline-10.17.0-linux-x64.tar.gz>
          tar -xvf flyway-commandline-10.17.0-linux-x64.tar.gz
          sudo mv flyway-10.17.0 /opt/flyway
          sudo ln -s /opt/flyway/flyway /usr/local/bin/flyway

      - name: Run Flyway migrations
        run: |
          flyway \\
            -url=jdbc:postgresql://localhost:5432/appdb \\
            -user=admin \\
            -password=Senha123 \\
            -locations=filesystem:./sql \\
            migrate
EOF
```

# **Fluxo final de trabalho**

Depois que tudo está configurado:

1. Você cria uma nova migração em sql/.
2. Faz git add . && git commit && git push.
3. O GitHub Actions:
   - sobe banco efêmero
   - instala Flyway
   - valida e aplica migrações
   - falha se algo estiver errado

Se a pipeline ficar verde --> a versão está aprovada.

Este projeto mostra como tratar o banco de dados como código de verdade com versionamento, validação automática e CI/CD.

Nada é aplicado manualmente, nada foge do controle, e toda mudança deixa rastros claros no histórico.

Com Flyway + GitHub Actions, migrações passam a ser previsíveis, auditáveis e seguras exatamente o que se espera de um ambiente profissional.
