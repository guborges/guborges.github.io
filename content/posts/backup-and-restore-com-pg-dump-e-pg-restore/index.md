---
title: "Backup e Restore com PG_DUMP e PG_RESTORE"
date: 2025-10-06T11:36:53-03:00
draft: false
categories: ["Backup & Recovery", "PostgreSQL"]
tags: ["Postgre SQL"]
cover:
  image: "ChatGPT-Image-Oct-6-2025-11_34_39-AM.png"
---

Realizar o backup em um banco de dados é uma tarefa essencial para qualquer DBA, muitas vezes podendo ser a diferença entre salvar dados críticos ou enfrentar consequências desastrosas no cenário profissional. Essa prática assegura que as informações estejam protegidas e possam ser recuperadas em situações de perda de dados, falhas no sistema ou, para não esquecer, o famoso e temido "update sem where".

Para tornar essa atividade mais fácil e até mesmo prazerosa, vou apresentar a vocês um passo a passo detalhado sobre como utilizar essa poderosa ferramenta de backup e recuperação.

Requisitos necessários para o tutorial:

- PostgreSQL instalado.
- Uma base de dados para nos divertirmos.

PG_DUMP

O utilitário do PostgreSQL designado para a realização de cópias do banco, mesmo quando este está em uso, é o pg_dump. Este aplicativo efetua a cópia de segurança de um banco de dados específico. Para realizar o backup de toda a instância ou de objetos globais compartilhados por todos os bancos de dados dessa instância, como funções de banco de dados, roles e espaços de tabelas, é aconselhável empregar o pg_dumpall ( é uma outra funcionalidade, qual não abordarei nesse artigo ).

Neste artigo, demonstrarei a forma mais básica e compreensível para realizar um backup.

Mas gostaria de mencionar um parâmetro que encontrei mais desafiador de compreender e que considero um dos mais importantes nesse comando.

A opção "-f" no Pg_dump nos proporciona a capacidade de escolher o tipo de saída a ser utilizado, e ela é subdividida da seguinte maneira:

```sql
 -F  --format=c|d|t|p 
		
-Fp - Este é o formato legível do backup, possibilitando edições manuais.
Utilidade: Ideal para uma análise manual do conteúdo do backup ou para realizar modificações pontuais. Entretanto, este formato tende a ocupar mais espaço em comparação com outros formatos binários.

-Fc - Este formato é compacto e binário, sendo mais eficiente em termos de espaço quando comparado ao formato de texto.
Utilidade: Recomendado para economizar espaço de armazenamento e facilitar a transferência, especialmente em ambientes que lidam com grandes bancos de dados.

-Fd - Produz um conjunto de arquivos em um diretório, com cada tabela armazenada em um arquivo separado.
Utilidade: Útil quando é necessário uma granularidade fina para restaurar ou manipular partes específicas do backup, proporcionando maior flexibilidade no processo de recuperação. 
```

---

Passo 1 - Criando seu Backup

Prefiro começar explicando a sintaxe do comando, pois acredito ser a abordagem mais eficaz para compreender as ações que estou realizando.

```sql
pg_dump [option] [dbname] > [ Arquivo backup ]
```

Executando o comando da seguinte maneira para realizar um backup do meu banco de dados chamado "db4", basta alterar o nome do [dbname] para o banco desejado e o [arquivo de backup] para o nome de sua preferência. No meu caso:

```sql
pg_dump -h localhost -U postgres -W -F t db4 > db4.tar
```

Resultado:

```sql
[postgres@localhost backup]$ psql
psql (13.13)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 db4       | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 db5       | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 db6       | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 dbgustavo | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 dump      | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 postgres  | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 repmgr    | repmgr   | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
 template0 | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 teste     | postgres | UTF8     | pt_BR.UTF-8 | pt_BR.UTF-8 |
(10 rows)

postgres=# \c db4
You are now connected to database "db4" as user "postgres".
db4=# \dt
Did not find any relations.
db4=#
db4=# create table gbteste (id numeric);
CREATE TABLE
db4=#
db4=# insert into gbteste values (1);
INSERT 0 1
db4=#
db4=#
db4=# exit
[postgres@localhost backup]$ pg_dump -h localhost -U postgres -W -F t db4 > db4.tar
Password:
[postgres@localhost backup]$ ll
total 8
-rw-r--r--. 1 postgres postgres 6656 dez 31 03:11 db4.tar
[postgres@localhost backup]$ 
```

Pronto, fizemos a criação de uma tabela, inserimos dados e realizamos o backup da mesma.

apenas de curiosidade, como defini o formato como -F t ( tar ) o arquivo de backup fica formatado dessa forma, impossibilitando a leitura.

Passo 2 - Realizando a restauração da base com o PG_RESTORE

É importante observar que o PG_RESTORE não pode ser empregado com todos os tipos de backups criados.

Agora que possuímos um backup de nossa base, que tal realizarmos uma restauração?

Vamos simular que, sem querer, eu executei um drop na tabela "gbteste" do banco de dados "db4":

Pronto, minha tabela desapareceu 😟 e agora?

Identifiquei a falha, vamos proceder com a restauração.

Podemos realizar o restore de duas formas, 1- o restore completo do banco de dados ou 2 - o restore apenas da tabela perdida, o que muda, é apenas 1 parâmetro no pg_restore que irei mostrar.

Podemos realizar, retornando apenas 1 única tabela, dessa forma:

```sql
pg_restore -U postgres -t gbteste -Ft -d db4 < db4.tar --verbose
```
ou retornando o banco inteiro ( removendo o -t )

```sql
pg_restore -U postgres -Ft -d db4 < db4.tar --verbose
```
Conclusão:

Neste processo de backup e restauração com o PostgreSQL, destacamos a importância do comando pg_dump para criar backups eficientes, permitindo escolher entre diferentes formatos. Ao enfrentar situações inesperadas, o pg_restore desempenha um papel crucial na restauração dos dados, resgatando a integridade do banco de dados. É fundamental compreender a sintaxe e os parâmetros desses comandos para garantir uma gestão eficaz dos dados, seja na prevenção de perdas ou na recuperação de informações cruciais.
