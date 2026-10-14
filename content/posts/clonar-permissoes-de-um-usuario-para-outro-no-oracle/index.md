---
title: "Clonar permissões de um usuário para outro no Oracle"
date: 2025-10-23T08:00:54-03:00
draft: false
categories: ["Oracle DBA", "Scripts & Automação"]
tags: ["Oracle", "Script Oracle"]
cover:
  image: "postclone.png"
---

Esta semana passei por uma situação recorrente no dia a dia. Sempre que precisava clonar permissões de um usuário para outro no Oracle eu fazia na mão, consultando dba_role_privs, dba_sys_privs, dba_tab_privs e dba_col_privs e etc etc.... Para eliminar retrabalho, escrevi um script simples que gera todos os GRANTs necessários em um arquivo .sql sem executar nada.

## O que o script gera

- Roles
- System privileges
- Object privileges
- Column privileges em formato agregado por objeto

O resultado é um arquivo .sql pronto para aplicar no usuário de destino.

## Como usar

1. Salve o conteúdo como `clone_permissoes.sql`
2. Execute no SQL*Plus com privilégios adequados   
   **sqlplus / as sysdba @clone_permissoes.sql USUARIO_ORIGEM USUARIO_DESTINO**
3. O script cria o arquivo `clone_USUARIO_ORIGEM_to_USUARIO_DESTINO.sql`
4. Revise e execute os GRANTs desse arquivo conforme a necessidade do ambiente

## Script

```text
-- +-------------------------------------------------------------------------------------------+
-- | Objetivo   : Clonar usuario Origem x Destino                                              |
-- | Criador    : Gustavo Borges Evangelista                                                   |
-- | Exemplo    : @clone_permissoes.sql usuarioOrigem usuarioDestino                           |
-- | Arquivo    : clone_permissoes.sql                                                         |
-- +-------------------------------------------------------------------------------------------+

-- clone_permissoes.sql (gera os GRANTs; não executa)
SET PAGES 0 LINES 500 FEEDBACK OFF HEADING OFF VERIFY OFF ECHO OFF SERVEROUTPUT OFF TERMOUT OFF TRIMSPOOL ON

COLUMN SRC_U NEW_VALUE SRC_U
COLUMN DST_U NEW_VALUE DST_U

-- Define variáveis (NÃO está em spool ainda)
SELECT COALESCE((SELECT username FROM dba_users WHERE username = UPPER('&amp;1')), UPPER('&amp;1')) SRC_U FROM dual;
SELECT COALESCE((SELECT username FROM dba_users WHERE username = UPPER('&amp;2')), UPPER('&amp;2')) DST_U FROM dual;

-- começa a gravar o arquivo
SPOOL clone_&amp;SRC_U._to_&amp;DST_U..sql

---------------
-- ROLES
---------------
SELECT 'GRANT "'||granted_role||'" TO "'||'&amp;DST_U'||'";' FROM dba_role_privs WHERE grantee='&amp;SRC_U' ORDER  BY granted_role;

---------------
-- SYSTEM PRIVS
---------------
SELECT 'GRANT '||privilege||' TO "'||'&amp;DST_U'||'";' FROM   dba_sys_privs WHERE  grantee='&amp;SRC_U' ORDER  BY privilege;

---------------
-- OBJECT PRIVS
---------------
SELECT 'GRANT '||privilege||' ON "'||owner||'"."'||table_name||'" TO "'||'&amp;DST_U'||'";' FROM   dba_tab_privs WHERE  grantee='&amp;SRC_U' ORDER  BY owner, table_name, privilege;

---------------
-- COLUMN PRIVS
---------------
WITH cols AS (
  SELECT privilege, owner, table_name,
         '"'||REPLACE(column_name,'"','""')||'"' AS col_q
  FROM   dba_col_privs
  WHERE  grantee='&amp;SRC_U'
)
SELECT 'GRANT '||privilege||' ('||
       LISTAGG(col_q, ', ') WITHIN GROUP (ORDER BY col_q)||
       ') ON "'||owner||'"."'||table_name||'" TO "'||'&amp;DST_U'||'";'
FROM cols
GROUP BY privilege, owner, table_name
ORDER BY owner, table_name, privilege;

SPOOL OFF
SET TERMOUT ON
```

## Exemplo

```text
sqlplus / as sysdba @clone_permissoes.sql UsuarioORIGEM UsuarioDESTINO
```

Saída gerada no arquivo

```text
GRANT "DBA" TO "UsuarioDESTINO";
GRANT CREATE SESSION TO "UsuarioDESTINO";
GRANT UNLIMITED TABLESPACE TO "UsuarioDESTINO";
```

## Notas rápidas

- O script apenas gera os comandos e não altera nada no banco
- O nome do arquivo usa os nomes efetivos dos usuários resolvidos em dba_users
- Se preferir registrar cabeçalho, ajuste para sua padronização de encoding e auditoria
- Caso precise incluir quotas, profiles, proxy connect ou políticas específicas, é possível estender o mesmo padrão acrescentando consultas e formatação dos comandos correspondentes

Objetivo simples. Menos tempo copiando permissão na mão e mais tempo resolvendo o que importa.
