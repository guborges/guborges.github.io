---
title: "DRCP no Oracle: o connection pool dentro do banco"
date: 2026-08-14T10:11:38-03:00
draft: false
categories: ["Oracle DBA"]
tags: ["dba", "Oracle"]
summary: "Configuração, monitoramento e troubleshooting do DRCP no Oracle 19c, com multi-pool e implicit connection pooling do 23ai."
cover:
  image: "ChatGPT-Image-Aug-14-2026-10_11_11-AM.png"
---

Toda aplicação que abre conexão direta no banco paga um preço fixo por conexão. No Oracle, cada conexão dedicada vira um processo de servidor mais uma sessão. Em uma aplicação web que abre, executa uma query curta e fecha, esse custo se repete o tempo todo. Multiplique por milhares de clientes em vários servidores de aplicação e o banco passa a gastar memória e CPU só para sustentar processos que ficam ociosos a maior parte do tempo.

Pool de conexão no middle tier resolve parte disso, mas tem um limite estrutural: cada tier mantém seu próprio mínimo de conexões abertas, e essas conexões não são compartilhadas entre processos ou hosts diferentes. Quando existem vários servidores de aplicação, cada um segurando seu pool, o total de conexões no banco cresce mesmo que poucas estejam ativas ao mesmo tempo.

O DRCP (Database Resident Connection Pool) ataca exatamente esse desperdício. Ele coloca o pool dentro do próprio banco, compartilhado entre todos os clientes. É um recurso antigo, disponível desde o 11.1, presente em Standard e Enterprise Edition, e mesmo assim raramente configurado. Este post mostra como ele funciona, como configurar e monitorar no 19c, o que mudou no 23ai/26ai com multi-pool e implicit connection pooling, e os erros que aparecem quando algo sai do esperado.

## O que o DRCP realmente faz

Um pooled server é a combinação de um processo de servidor (foreground) com uma sessão de banco. Em vez de cada cliente ter o seu, o DRCP mantém um conjunto desses pooled servers prontos e os empresta sob demanda.

No meio do caminho entra o connection broker. O cliente não se conecta direto a um processo de servidor: ele se conecta e autentica no broker, que escolhe um pooled server livre e faz o hand-off da conexão para ele. O processo de background que cumpre esse papel é o CMON. Quando o cliente termina o trabalho e libera a conexão, o pooled server volta para o pool e fica disponível para o próximo.

O ganho direto é a redução de processos de servidor no banco. Em vez de manter um processo por conexão aberta, o banco mantém um processo por conexão ativa. A documentação oficial é explícita sobre o caso de uso: arquiteturas com servidores de aplicação multi-processo e single-threaded, como PHP com Apache, que não conseguem fazer pool no middle tier. Com DRCP, o banco escala para dezenas de milhares de conexões simultâneas sem um processo de servidor para cada uma.

## Quando faz sentido ligar o DRCP

O DRCP não serve para qualquer carga. Ele rende quando o número de conexões ativas é bem menor que o número de conexões abertas. Quanto maior essa diferença, maior o benefício.

A documentação oficial lista as condições onde ele se encaixa:

- muitas conexões de cliente precisam ser sustentadas com uso mínimo de memória
- as aplicações são parecidas e podem compartilhar ou reusar sessões (mesmas credenciais, mesmo schema)
- o cliente pega a conexão, trabalha por um período curto e libera
- não há necessidade de afinidade de sessão entre requisições
- existem múltiplos processos e múltiplos hosts do lado do cliente

Se a aplicação segura a conexão aberta o tempo todo e fica processando, o DRCP puro não ajuda muito, porque nunca chega o sinal de liberação que permitiria reusar o processo. Esse cenário específico só passou a ter solução no 23ai, com implicit connection pooling, que vamos ver mais adiante.

## Configuração no 19c

O DRCP já vem instalado. Existe um pool padrão chamado `SYS_DEFAULT_CONNECTION_POOL`, que nasce inativo. Tudo é gerenciado pelo pacote `DBMS_CONNECTION_POOL`, e você precisa estar como SYSDBA para iniciar ou parar o pool.

Primeiro, verifique o estado atual:

```sql
SELECT connection_pool, status, maxsize FROM dba_cpool_info;

CONNECTION_POOL                STATUS     MAXSIZE
------------------------------ ---------- ----------
SYS_DEFAULT_CONNECTION_POOL    INACTIVE          40
```

Antes de iniciar, ajuste os parâmetros para a sua carga. O `CONFIGURE_POOL` define tudo de uma vez:

```sql
BEGIN
  DBMS_CONNECTION_POOL.CONFIGURE_POOL(
    pool_name          => 'SYS_DEFAULT_CONNECTION_POOL',
    minsize            => 4,
    maxsize            => 50,
    incrsize           => 2,
    inactivity_timeout => 120,
    max_think_time     => 60);
END;
/
```

Se quiser alterar um único parâmetro sem mexer no resto, use o `ALTER_PARAM`:

```sql
EXEC DBMS_CONNECTION_POOL.ALTER_PARAM('SYS_DEFAULT_CONNECTION_POOL','MAXSIZE','80');
```

Com os parâmetros definidos, inicie o pool:

```sql
EXEC DBMS_CONNECTION_POOL.START_POOL;
```

Os parâmetros que mais importam, com os valores padrão documentados:

| Parâmetro | Padrão | O que controla |
| --- | --- | --- |
| `minsize` | 4 | Mínimo de pooled servers mantidos no pool |
| `maxsize` | 40 | Máximo de pooled servers no pool |
| `incrsize` | 2 | Quantos servers o pool cria quando não há livre na hora do request |
| `inactivity_timeout` | 300 | TTL de uma sessão ociosa no pool, em segundos. Serve para encolher o pool |
| `max_think_time` | 120 | Tempo máximo de inatividade do cliente após pegar um server, sem transação aberta. Estourou, o server é liberado e a conexão do cliente é terminada |
| `max_txn_think_time` | igual ao `max_think_time` | Mesma ideia, mas com transação aberta. Pode ser maior |
| `max_use_session` | 500000 | Ciclos de uso antes de o pooled server ser reciclado |
| `max_lifetime_session` | 86400 | Tempo de vida máximo de um pooled server, em segundos |

O `max_think_time` merece atenção. Ele é uma proteção contra o cliente que pega um pooled server e some, segurando o recurso. Mas se você colocar um valor agressivo demais para uma aplicação que tem pausas legítimas entre chamadas, o banco vai derrubar essas conexões e o cliente recebe erro. É um parâmetro para calibrar com o comportamento real da aplicação, não chutar.

### Do lado do cliente

Configurar o pool no servidor não muda nada sozinho. O cliente precisa pedir explicitamente uma conexão pooled, e isso é feito com `SERVER=POOLED` na string de conexão.

Easy Connect:

```bash
sqlplus appuser/senha@dbhost:1521/appsvc:POOLED
```

Entrada no tnsnames.ora:

```bash
APPDB_POOL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = appsvc)
      (SERVER = POOLED)))
```

JDBC:

```bash
jdbc:oracle:thin:@//dbhost:1521/appsvc:POOLED
```

Se você esquecer o `SERVER=POOLED`, a conexão funciona, mas vai como dedicada e o DRCP não entra na jogada. O pool fica ativo no banco e ninguém usa.

## Connection class e purity: onde o reuso acontece de verdade

Esse é o ponto que separa um DRCP que funciona de um que parece configurado mas não reusa nada.

Para o DRCP reaproveitar uma sessão, ele precisa saber que duas requisições podem compartilhar o mesmo estado de sessão. Isso é controlado por dois conceitos: connection class e purity. A connection class agrupa conexões que podem reusar os mesmos pooled servers. A purity diz se a sessão pode trazer estado de uso anterior (`SELF`) ou precisa começar limpa (`NEW`).

Quando a connection class está definida, sessões e servers são reusados, e isso aparece como número alto de hits no monitoramento. Quando não está, cada requisição tende a virar uma sessão nova, o que joga fora boa parte do ganho.

Aqui mora uma pegadinha clássica. O SQL*Plus não define purity por padrão e, por isso, não reusa sessões DRCP. Se você testar o DRCP só com SQL*Plus e olhar as estatísticas esperando ver reuso, vai se assustar com o número de misses. Não é o pool que está quebrado, é a ferramenta de teste que não pede reuso. Os drivers de aplicação (JDBC, python-oracledb, OCI) é que definem connection class e purity adequadamente.

## Monitoramento

Toda a observabilidade do DRCP está em um punhado de views. A primeira parada é o `DBA_CPOOL_INFO`, que mostra configuração e status. A coluna do nome do pool ali se chama `CONNECTION_POOL`, não `POOL_NAME`.

```sql
SELECT connection_pool, status, maxsize FROM dba_cpool_info;
```

A view que diz se o pool está bem dimensionado é a `V$CPOOL_STATS`:

```sql
SELECT num_requests, num_hits, num_misses, num_waits FROM v$cpool_stats;

NUM_REQUESTS   NUM_HITS NUM_MISSES  NUM_WAITS
------------ ---------- ---------- ----------
      100310      99990         40       1055
```

A leitura é direta. Misses baixo significa que sessões e servers estão sendo reusados, ou seja, a connection class está fazendo o trabalho dela. Waits alto significa que o `maxsize` é pequeno demais para a carga: requisições estão esperando um pooled server livre. No exemplo acima, pouco mais de 1% das requisições precisou esperar, o que é saudável. Se esse número sobe, aumente o `maxsize`.

Para ver isso quebrado por connection class:

```sql
SELECT cclass_name, num_requests, num_hits, num_misses FROM v$cpool_cc_stats;
```

Para ver quem está conectado no pool agora:

```sql
SELECT username, program, machine FROM v$cpool_conn_info;
```

A partir do 21c existe também a `V$AUTHPOOL_STATS`, com estatísticas dos servidores de autenticação do DRCP:

```sql
SELECT num_srvs, num_busy, num_free, num_waiters FROM v$authpool_stats;
```

Tem um detalhe de multitenant que pega muita gente. Com DRCP configurado no nível do CDB (o padrão), a `V$CPOOL_STATS` só retorna dados quando consultada do CDB root como SYS. Consultada de dentro de uma PDB, ela volta zero linhas. Se você está olhando de dentro da PDB e não vê estatística nenhuma, não é que o DRCP não está sendo usado, é o escopo da view. Com per-PDB DRCP a história muda, e aí cada PDB enxerga as suas.

## Dimensionamento na prática

O `maxsize` é o parâmetro central, e a lógica de dimensionamento parte de uma ideia simples: o pool precisa ter pooled servers suficientes para o pico de conexões ativas simultâneas, não para o total de conexões abertas.

O exemplo da AWS para RDS Oracle ilustra bem. Em um cenário com 2000 sessões vindas dos tiers de aplicação, mas com no máximo 10% ativas ao mesmo tempo, eles configuraram o pool com `maxsize` de 200. As 2000 sessões compartilham 200 pooled servers. O resto da memória que seria gasta com 1800 processos ociosos simplesmente não é alocada.

O sinal de que você errou para baixo é o `num_waits` subindo na `V$CPOOL_STATS`. Uma consulta útil para acompanhar isso, juntando configuração e estatística:

```sql
SELECT cpi.connection_pool, cpi.status, cs.num_waits
FROM   dba_cpool_info cpi, v$cpool_stats cs
WHERE  cpi.connection_pool = cs.pool_name;
```

Waits subindo de forma consistente é o gatilho para aumentar o `maxsize`. O caminho contrário, pool muito maior que a carga, desperdiça memória mas não quebra nada.

## O que mudou no 23ai e no 26ai

Por anos o DRCP teve uma limitação incômoda: só existia um pool por instância, o `SYS_DEFAULT_CONNECTION_POOL`. Todas as aplicações daquele banco dividiam a mesma configuração. Duas novidades mudaram isso.

### Multi-pool DRCP

A partir do 23ai dá para criar múltiplos pools nomeados, cada um com a sua configuração. Você isola, por exemplo, um pool enxuto para o OLTP e um pool separado para relatórios, sem que um interfira no outro.

```sql
BEGIN
  DBMS_CONNECTION_POOL.ADD_POOL(
    pool_name => 'report_pool',
    minsize   => 2,
    maxsize   => 20);
END;
/

EXEC DBMS_CONNECTION_POOL.START_POOL('report_pool');
```

O `SYS_DEFAULT_CONNECTION_POOL` continua existindo e sendo o pool padrão. Os pools novos são geridos pelos mesmos procedimentos de sempre: `start_pool`, `stop_pool`, `configure_pool`, `alter_param`, `restore_defaults`. Para direcionar a aplicação ao pool certo, a string de conexão precisa do `POOL_NAME` junto com `SERVER=POOLED`:

```bash
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost)(PORT = 1521))
  (CONNECT_DATA =
    (SERVICE_NAME = appsvc)
    (SERVER = POOLED)
    (POOL_NAME = report_pool)))
```

Vale registrar que multi-pool é uma mudança server-side. A documentação oficial confirma que clientes antigos, de release 12.1 e 19c, conseguem se conectar a um banco 26ai e usar os recursos de multi-pool.

### Implicit connection pooling

Essa é a novidade que cobre o caso que o DRCP puro não resolvia: a aplicação que segura a conexão aberta durante toda a sua vida e fica ociosa boa parte do tempo, sem nunca chamar um close que sinalize que o server pode ser reusado.

O implicit connection pooling faz o banco detectar sozinho quando a sessão está implicitamente sem estado e devolver o pooled server para o pool, de forma transparente. Quando a aplicação faz a próxima chamada, um server livre é alocado de volta para ela e ela continua sem perceber a troca. Nenhuma mudança de código é necessária, só de string de conexão.

O controle é feito pelo `POOL_BOUNDARY`, que tem dois valores:

- `STATEMENT`: o server volta para o pool quando a sessão fica implicitamente sem estado
- `TRANSACTION`: o server volta quando a transação termina com commit ou rollback, ou quando a sessão fica sem estado

Para ser considerada implicitamente sem estado, a sessão precisa satisfazer todas estas condições: todos os cursores abertos foram lidos até o fim, não há transação ativa, não há LOBs temporários, não há tabelas temporárias globais com linhas e não há tabelas temporárias privadas abertas. Se qualquer uma falha, a sessão é considerada com estado e não é liberada.

Na string de conexão fica assim, no formato Easy Connect:

```bash
dbhost:1521/appsvc:POOLED?pool_boundary=transaction&pool_connection_class=myapp
```

A escolha do boundary depende do comportamento da aplicação. Se as sessões raramente ficam sem estado mas você quer forçar a liberação, `TRANSACTION` é a opção mais segura. Se você sabe que as sessões ficam sem estado entre statements, `STATEMENT` aproveita mais. A purity nesse modo é `SELF` por padrão, o que permite reuso da memória de sessão. Adicionar `POOL_PURITY=NEW` força a recriação do estado a cada uso, mais limpo e mais caro.

### Per-PDB DRCP

Desde o 21c o DRCP pode ser configurado por PDB, controlado pelo parâmetro `ENABLE_PER_PDB_DRCP`. O padrão é `FALSE`, ou seja, DRCP no nível do CDB root. Com `TRUE`, cada administrador de PDB gerencia o seu próprio pool:

```sql
ALTER SYSTEM SET enable_per_pdb_drcp=TRUE SCOPE=SPFILE;
-- exige reinício da instância
```

Um ponto que vale saber antes de tentar: os processos de broker são de propriedade do root e compartilhados entre os pools das PDBs, mesmo no modo per-PDB. E enquanto o `ENABLE_PER_PDB_DRCP` estiver `FALSE`, tentar gerenciar pool de dentro de uma PDB dá erro, como veremos no troubleshooting.

## Troubleshooting: os erros que aparecem de verdade

Estes são erros documentados ou reportados publicamente, não hipóteses.

ORA-12520: TNS:listener could not find available handler for requested type of server. Aparece quando o cliente tenta conectar com `SERVER=POOLED` mas o pool está parado. O listener não tem handler do tipo pooled para entregar. Confirme com `SELECT status FROM dba_cpool_info` e dê `START_POOL` se estiver `INACTIVE`.

ORA-03113: end-of-file on communication channel. Pode aparecer em uma sessão pooled que ficou ociosa além do `inactivity_timeout` e foi fechada para encolher o pool. Do ponto de vista do cliente, a conexão simplesmente morreu. Se isso acontece com sessões que deveriam estar legitimamente ociosas, o `inactivity_timeout` ou o `max_think_time` está curto demais para o padrão da aplicação.

ORA-24459: OCISessionGet() timed out waiting for pool to create new connections. Esse foi reportado em produção com driver godror em um 19c. O cliente pediu uma conexão, o pool não tinha pooled server livre e não conseguiu criar um novo a tempo. É o sintoma clássico de `maxsize` pequeno para o pico de carga, ou de o `incrsize` não dar conta da velocidade de chegada das requisições. Olhe o `num_waits` na `V$CPOOL_STATS` e considere subir o `maxsize`.

ORA-56500 e ORA-56501: Pool not found e Pool startup failed. Tipicamente um nome de pool errado no `start_pool` ou `configure_pool`. Confira o nome exato com `DBA_CPOOL_INFO` antes de chamar o procedimento.

ORA-56515: DRCP Operation not allowed from a Pluggable Database. Aparece ao tentar `ADD_POOL` ou gerenciar pool de dentro de uma PDB com `ENABLE_PER_PDB_DRCP` ainda em `FALSE`. Nesse modo o gerenciamento é só pelo root. Para gerenciar por PDB, ligue o `ENABLE_PER_PDB_DRCP` e reinicie.

ORA-56504, ORA-56505 e ORA-56507: exceções do `ALTER_PARAM`, para nome de parâmetro inválido, valor inválido e falha ao alterar a configuração. Quase sempre é digitação errada no nome ou valor do parâmetro.

## Riscos e cuidados antes de levar para produção

O DRCP muda o modelo de conexão, e isso tem implicações que vale mapear antes.

Vazamento de estado entre requisições é o risco mais sério. Se a aplicação depende de estado de sessão acumulado (variáveis de package, configurações de NLS, contexto) e a purity permite reuso, uma requisição pode pegar uma sessão com resíduo de outra. Defina connection class e purity de forma consciente. Aplicações que não toleram esse compartilhamento precisam de purity `NEW`, pagando o custo de sessão limpa.

Timeouts agressivos derrubam conexões legítimas. Já cobri no `max_think_time` e no `inactivity_timeout`, mas reforço porque é a causa mais comum de gente abandonar o DRCP achando que ele é instável. Calibre com o comportamento real, não com o padrão.

O connection broker é um ponto central no fluxo. Toda conexão pooled passa por ele para autenticação e hand-off. Em cargas de altíssima concorrência de conexão, o dimensionamento de brokers e de conexões por broker (via `CONNECTION_BROKERS`) entra na conta.

Ao parar o pool, o `STOP_POOL` aceita um `draintime` que controla por quanto tempo ele espera as conexões em uso terminarem antes de derrubar. Parar pool sem pensar no drain em horário de carga vai cortar sessões ativas.

## Validação técnica

O que está confirmado na documentação oficial da Oracle: a definição de pooled server como processo de servidor mais sessão, o papel do connection broker, os casos de uso recomendados, todos os parâmetros e seus padrões do `DBMS_CONNECTION_POOL` (release 26 do PL/SQL Packages Reference), o comportamento das views de monitoramento e a interpretação de hits, misses e waits, a pegadinha do SQL*Plus não setar purity, o comportamento da `V$CPOOL_STATS` retornar zero linhas de dentro da PDB em DRCP de nível CDB, o multi-pool e o implicit connection pooling do 23ai com `POOL_BOUNDARY`, e o `ENABLE_PER_PDB_DRCP` do 21c. As fontes são o Administrator's Guide, o PL/SQL Packages Reference e o technical brief oficial de DRCP.

O que é prática de mercado validada em fontes externas confiáveis: a abordagem de dimensionar o `maxsize` pelo pico de conexões ativas (exemplo da AWS para RDS Oracle), e as strings de conexão com `SERVER=POOLED` em Easy Connect, tnsnames e JDBC (ORACLE-BASE e documentação de drivers).

O que são erros reais e rastreáveis, não hipóteses: ORA-24459 reportado em produção com godror em 19c, e ORA-12520, ORA-03113 e ORA-56515 documentados em fontes públicas com o contexto que os dispara.

O que varia por ambiente: os valores ideais de `maxsize`, `minsize`, `inactivity_timeout` e `max_think_time` dependem inteiramente do padrão de carga da sua aplicação. Os números deste post são ponto de partida e exemplo, não recomendação universal. Meça com a `V$CPOOL_STATS` na sua carga antes de fixar valores.

## Conclusão

O DRCP é um caso raro de recurso que está ali, instalado, licenciado em qualquer edição, e que a maioria dos ambientes nunca liga. Para aplicações com muitas conexões curtas e poucas ativas ao mesmo tempo, especialmente em arquiteturas com vários servidores de aplicação que não compartilham pool entre si, ele corta processos de servidor e memória de forma significativa.

A configuração é simples. O que separa um DRCP útil de um inútil é entender connection class e purity, dimensionar o `maxsize` pela carga ativa real e monitorar misses e waits na `V$CPOOL_STATS`. No 23ai e no 26ai o recurso ficou bem mais flexível, com pools nomeados por aplicação e o implicit connection pooling, que finalmente cobre o caso das conexões longas e ociosas sem exigir mudança de código. Se você tem uma carga que se encaixa no perfil, vale medir o ganho antes de assumir que precisa de mais memória no servidor.
