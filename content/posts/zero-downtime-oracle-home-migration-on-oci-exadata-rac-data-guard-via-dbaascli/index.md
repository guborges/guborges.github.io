---
title: "Zero-Downtime Oracle Home Migration on OCI Exadata RAC + Data Guard via DBAASCLI"
date: 2025-12-23T17:34:07-03:00
draft: false
categories: ["Cloud", "Oracle DBA", "Patching & Upgrades"]
tags: ["Cloud", "Oracle", "Projetos"]
cover:
  image: "dbhomedbaascli.png"
---

## O que é esse processo

Aqui a ideia é simples:

- Você já tem um **novo Oracle Home** criado no OCI (ex: RU 19.27).
- Seu banco ainda está rodando no **Oracle Home antigo** (ex: RU 19.25).
- O que vamos fazer é **mover o banco (RAC) para iniciar usando o novo Home**, via `dbaascli`.
- Depois, no Primary, rodar **Datapatch** para casar dicionário com o novo RU.

Isso não copia datafile, não mexe em ASM, não recria DB.  
É troca do **Oracle Home usado pelo start do banco**.

## Variáveis do exemplo (troque pelos seus nomes)

Banco:

- `instanciagustavo`

Nodes:

- `node1 node2 node3 node4`

Oracle Home antigo (exemplo):

- `/u02/app/oracle/product/19.0.0.0/dbhome_3` (19.25)

Oracle Home novo (exemplo):

- `/u02/app/oracle/product/19.0.0.0/dbhome_4` (19.27)

# Parte 1 – Standby (aplicar primeiro)

## Etapa 1 – Confirmar estado atual do banco e do Oracle Home em uso

Nesta etapa iremos **confirmar em qual Oracle Home o banco está rodando agora**, e validar informações básicas antes de qualquer mudança.

```text
dbaascli database getDetails --dbname instanciagustavo
```

O que você procura aqui:

- Oracle Home atual
- Nodes do cluster
- Status do banco
- Informações que confirmam que você está mexendo no banco certo

## Etapa 2 – Rodar prereqs por node (sem mover nada)

Nesta etapa iremos **executar somente validações** para garantir que o node consegue mover o banco para o novo Oracle Home.

```text
dbaascli database move --dbname instanciagustavo \
  --oracleHome /u02/app/oracle/product/19.0.0.0/dbhome_4 \
  --skipDatapatch \
  --nodeList node1 \
  --executePrereqs
```

Repita para `node2`, `node3`, `node4`.

O que isso faz:

- valida se o novo home existe e está OK
- valida dependências no node
- valida que o dbaascli consegue operar o recurso do banco naquele node

O que isso **não faz**:

- não para instância
- não muda Oracle Home
- não aplica patch

## Etapa 3 – Garantir Apply sincronizado e parar Apply

Nesta etapa iremos **garantir que o standby está sincronizado** e depois **parar o apply**, para não ter redo sendo aplicado durante a troca do Oracle Home.

Checklist rápido:

- apply lag zerado ou dentro do aceitável
- MRP rodando
- standby saudável

Depois:

- parar apply (do jeito que você usa no seu padrão operacional)

## Etapa 4 – Executar o move do banco para o novo Oracle Home (node a node)

Nesta etapa iremos **realizar o move do banco para o novo Oracle Home**, que já contém a versão/RU desejada (ex: 19.27).  
Aqui é onde o banco passa a **subir usando o novo home**.

```text
dbaascli database move --dbname instanciagustavo \
  --oracleHome /u02/app/oracle/product/19.0.0.0/dbhome_4 \
  --skipDatapatch \
  --nodeList node1 \
  --waitForCompletion false
```

Repita para `node2`, `node3`, `node4`.

O que acontece na prática:

- o recurso do banco no CRS passa a apontar pro novo home
- as instâncias daquele node são reiniciadas controladamente para subir usando os binaries do novo home
- ao final, aquele node já está com o banco rodando no home novo

Por que `--waitForCompletion false`:

- você dispara por node e acompanha o progresso/logs sem ficar travado no terminal
- se um node demorar mais, você não fica preso no comando do outro

## Etapa 5 – Validar listener e network/admin após o move

Nesta etapa iremos **garantir que as configurações de rede e listener estão coerentes com o novo Oracle Home**, porque isso é o tipo de coisa que “não dá erro”, mas te quebra depois.

Validar:

- se o diretório do banco em `network/admin` existe no novo home  
  Exemplo esperado:
  - antigo: `/.../dbhome_3/network/admin/instanciagustavo`
  - novo: `/.../dbhome_4/network/admin/instanciagustavo`

E validar listener no node principal (grid / listener que serve o banco):

- listener apontando para o home correto
- serviços do banco registrando normalmente

## Etapa 6 – Subir Apply e confirmar archive aplicando

Nesta etapa iremos **retomar o apply** e validar que o standby voltou a aplicar archive normalmente.

Checklist:

- MRP rodando
- apply lag evoluindo/zerando
- archive chegando e sendo aplicado
- sem erro em alert log referente a serviço/listener/home

# Parte 2 – Primary (depois dos standbys)

## Etapa 7 – Confirmar Home atual no Primary

Nesta etapa iremos **confirmar o Oracle Home atual do Primary** antes do move.

```text
dbaascli database getDetails --dbname instanciagustavo
```

## Etapa 8 – Rodar prereqs por node no Primary

Nesta etapa iremos **validar node a node** antes de mover.

```text
dbaascli database move --dbname instanciagustavo \
  --oracleHome /u02/app/oracle/product/19.0.0.0/dbhome_4 \
  --skipDatapatch \
  --nodeList node1 \
  --executePrereqs
```

Repetir para `node2,node3,node4`.

## Etapa 9 – Move do banco no Primary para o novo Oracle Home (node a node)

Nesta etapa iremos **mover o banco para o novo home** (RU 19.27) no Primary.

```text
dbaascli database move --dbname instanciagustavo \
  --oracleHome /u02/app/oracle/product/19.0.0.0/dbhome_4 \
  --skipDatapatch \
  --nodeList node1 \
  --waitForCompletion false
```

Repetir para `node2,node3,node4`.

Resultado esperado:

- Primary rodando com binaries do home novo

## Etapa 10 – Executar Datapatch no Primary

Nesta etapa iremos **atualizar o dicionário do banco** para refletir os SQL patches da RU 19.27 do novo Oracle Home.  
Essa etapa é o que fecha o ciclo da atualização.

```text
dbaascli database runDatapatch --dbname instanciagustavo --waitForCompletion false
```

O que isso faz:

- aplica SQL patches no dicionário
- sincroniza `dba_registry_sqlpatch` com o novo Oracle Home

## Etapa 11 – Validações finais (o que eu confiro sempre)

Nesta etapa iremos **confirmar que o ambiente ficou estável e coerente** após a troca do home.

Validações que não negocio:

- Banco rodando em todos os nodes
- Serviços registrando via listener
- `dba_registry_sqlpatch` consistente
- Objetos inválidos sob controle
- Standbys aplicando archive
- apply lag dentro do esperado

## Conclusão

Ao final desse processo, o banco **instanciagustavo** passa a executar utilizando o **novo Oracle Home**, que neste cenário corresponde à RU **19.27**. A troca do Oracle Home foi realizada sem recriação do banco, sem movimentação de datafiles e sem impacto estrutural em ASM ou Data Guard.

O ambiente agora se encontra com:

- Binaries do Oracle Database alinhados à versão 19.27
- Dicionário atualizado via Datapatch no Primary
- Standbys operando e aplicando archive normalmente
- Oracle Home antigo ainda presente no filesystem, permitindo validações adicionais antes de qualquer descomissionamento

A partir desse ponto, o banco está efetivamente rodando na nova versão, com o ciclo de atualização concluído.  
Qualquer ajuste posterior passa a ocorrer já sobre o Oracle Home atualizado, mantendo consistência entre binaries, dicionário e operação do ambiente.
