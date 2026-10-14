---
title: "Sem titulo"
date: 2026-03-06T13:49:25-03:00
draft: true
categories: ["Uncategorized"]
---

Oracle Data Guard continua sendo a principal tecnologia de alta disponibilidade e disaster recovery do ecossistema Oracle. Com o lançamento do **Oracle AI Database 26ai**, a Oracle trouxe melhorias significativas que impactam diretamente o dia a dia do DBA: novas views de monitoramento, saída JSON no DGMGRL, APIs PL/SQL via `DBMS_DG`, suporte a Data Guard por PDB e integração simplificada com ambientes híbridos (on-premises + OCI).

Neste artigo, vamos explorar as views mais importantes para monitoramento de ambientes Data Guard no 26ai — desde as views clássicas até as **4 novas views introduzidas nesta versão** — com exemplos práticos de SQL que podem ser utilizados em health checks, scripts automatizados e troubleshooting.

🔧 SQL Interativo com FreeSQL

Algumas queries abaixo incluem editores interativos do **FreeSQL**, onde você pode executar SQL diretamente em uma instância real do Oracle AI Database 26ai. As views V$ de Data Guard exigem uma configuração DG ativa — os embeds demonstram a sintaxe SQL no ambiente hospedado do FreeSQL. Faça login com seu Oracle SSO para ter acesso completo de leitura e escrita.

#### Índice

1. [Identificando o Papel do Banco de Dados](#s1)
2. [Monitorando Apply Lag e Transport Lag](#s2)
3. [Status dos Processos Data Guard](#s3)
4. [Status do Transporte de Redo](#s4)
5. [Archive Logs Recebidos e Aplicados](#s5)
6. [Detecção de Gaps de Archive](#s6)
7. [[26ai] V$DG_BROKER_CONFIG — Visão Completa da Configuração](#s7)
8. [[26ai] V$DG_BROKER_PROPERTY — Propriedades do Broker via SQL](#s8)
9. [[26ai] V$DG_BROKER_ROLE_CHANGE — Histórico de Role Changes](#s9)
10. [DGMGRL com Saída JSON no 26ai](#s10)
11. [PREPARE DATABASE FOR DATA GUARD](#s11)
12. [AWR Automático no Active Data Guard](#s12)
13. [Tabela de Referência Rápida](#resumo)

### Primeiro, vamos confirmar que estamos no Oracle 26ai

Execute a query abaixo diretamente no FreeSQL para ver o banner da versão 26ai:

▶ FreeSQL — Executar no Oracle AI Database 26ai INTERATIVO

<https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DlFISszLSy2KTyvNyVFwC/L3VShTKUstKs7MzwMAN3qKriEAAAA=&code_language=SQL&code_format=false>

## 1. Identificando o Papel do Banco de Dados

Antes de qualquer análise em um ambiente Data Guard, o primeiro passo é confirmar se estamos conectados ao **PRIMARY** ou ao **STANDBY**, e qual é o modo de operação atual.

```
SELECT
    name                    AS db_name,
    db_unique_name          AS db_unique_name,
    database_role           AS database_role,
    open_mode               AS open_mode,
    protection_mode         AS protection_mode,
    switchover_status       AS switchover_status
FROM
    v$database;
```

▶ FreeSQL — Consultar V$DATABASE no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAIK8xNxUBcdghZSkeBBTBywI5JTmZRaWpiKLJZYkJiUWp8YX5edAhfILUvPic/NToNyCovyS1OSSzHxkweLyzJLkjPyy1KL44pLEktJiLrcgf1+FMhWYeQDHMa9AjAAAAA==&code_language=SQL&code_format=false

### Saída esperada no PRIMARY:

| DB_NAME | DB_UNIQUE_NAME | DATABASE_ROLE | OPEN_MODE | PROTECTION_MODE | SWITCHOVER_STATUS |
| --- | --- | --- | --- | --- | --- |
| CDB1 | cdb1 | PRIMARY | READ WRITE | MAXIMUM PERFORMANCE | TO STANDBY |

### Saída esperada no STANDBY:

| DB_NAME | DB_UNIQUE_NAME | DATABASE_ROLE | OPEN_MODE | PROTECTION_MODE | SWITCHOVER_STATUS |
| --- | --- | --- | --- | --- | --- |
| CDB1 | cdb1_stby | PHYSICAL STANDBY | READ ONLY WITH APPLY | MAXIMUM PERFORMANCE | TO PRIMARY |

O que observar

`OPEN_MODE = READ ONLY WITH APPLY` confirma que o **Active Data Guard** está ativo. `SWITCHOVER_STATUS = TO STANDBY` (no primary) ou `TO PRIMARY` (no standby) indica que o ambiente está pronto para switchover. No 26ai, a `V$DG_BROKER_CONFIG` ganhou colunas como `SWITCHOVER_READY` e `FAILOVER_READY`, permitindo validação direta sem usar DGMGRL.

## 2. Monitorando Apply Lag e Transport Lag

A métrica mais crítica em Data Guard é o **apply lag** — o atraso entre o que foi gerado no primary e o que foi aplicado no standby. O **transport lag** indica o atraso no envio dos redo logs.

```
SELECT
    name,
    value,
    unit,
    time_computed,
    datum_time
FROM
    v$dataguard_stats
WHERE
    name IN ('transport lag', 'apply lag',
             'apply finish time', 'estimated startup time')
ORDER BY
    name;
```

▶ FreeSQL — Consultar V$DATAGUARD_STATS no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAIK8xNxUHTCrLDGnFMoszcssgbBKMnNT45PzcwtKS1JTuNyC/H0VylRSEksS00sTi1Lii0sSS4q5/INcXIMUnCLBhgEAQG1gNlwAAAA=&code_language=SQL&code_format=false

### Saída esperada (standby sincronizado):

| NAME | VALUE | UNIT | TIME_COMPUTED |
| --- | --- | --- | --- |
| apply finish time | +00 00:00:00 | day(2) | 06-MAR-2026 10:30:15 |
| apply lag | +00 00:00:00 | day(2) | 06-MAR-2026 10:30:15 |
| estimated startup time | 12 | second | 06-MAR-2026 10:30:15 |
| transport lag | +00 00:00:00 | day(2) | 06-MAR-2026 10:30:15 |

Interpretação

`apply lag = +00 00:00:00` → standby 100% sincronizado. Valores acima de 5 minutos podem indicar gargalo de I/O, rede ou contention no apply. O `estimated startup time` mostra quanto tempo o standby levaria para ficar disponível após ativação. **Nota:** `V$DATAGUARD_STATS` retorna linhas apenas quando executada no **standby**.

## 3. Status dos Processos Data Guard — V$DATAGUARD_PROCESS

No 26ai, `V$DATAGUARD_PROCESS` é a view recomendada para verificar todos os processos envolvidos na replicação, substituindo parte do uso da antiga `V$MANAGED_STANDBY`.

```
SELECT
    pid,
    name,
    role,
    action,
    client_role,
    sequence#,
    block#,
    active_agents,
    known_agents
FROM
    v$dataguard_process
ORDER BY
    name;
```

▶ FreeSQL — Consultar V$DATAGUARD_PROCESS no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAIKCzBQdMCMvMTcVwirKz4GyEpNLMvPzIOzknMzUvJJ4hGRxamFpal5yqjKXW5C/r0KZSkpiSWJ6aWJRSnxBUX5yanExl3+Qi2uQglMk2HAAsLHTXnUAAAA=&code_language=SQL&code_format=false

### Saída esperada no STANDBY:

| PID | NAME | ROLE | ACTION | CLIENT_ROLE | SEQUENCE# |
| --- | --- | --- | --- | --- | --- |
| 25 | DMON | Broker Monitor | IDLE |  | 0 |
| 30 | MRP0 | Recovery | APPLYING_LOG |  | 487 |
| 35 | PR00 | Recovery Worker | APPLYING_LOG |  | 487 |
| 40 | RFS | Log Transport | RECEIVING | PRIMARY | 487 |

Processos-chave

**MRP0** com `APPLYING_LOG` → apply ativo e funcionando. **RFS** com `RECEIVING` → redo sendo recebido do primary em tempo real. **DMON** → processo do Broker. Se o MRP0 não existir na lista, o apply está parado e precisa ser reiniciado.

## 4. Status do Transporte de Redo — V$ARCHIVE_DEST

O transporte de redo é o elo crítico entre primary e standby. Problemas aqui significam standby desatualizado.

```
SELECT
    dest_id,
    dest_name,
    status,
    type,
    recovery_mode,
    synchronization_status,
    synchronized,
    gap_status
FROM
    v$archive_dest_status
WHERE
    status <> 'INACTIVE'
    AND type <> 'LOCAL';
```

▶ FreeSQL — Consultar V$ARCHIVE_DEST no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAIKU1OKS+MwUHQQnLzE3FcItLkksKS2GsEsqC1IRijLzEksy8/O43IL8fRXKVBKLkjMyy1LjQVJc4R6uQa5QvQo2dgrqnn6OziGeYa7qXI5+LgpB/uF+ob4KNrYKpgAzfpRmhwAAAA==&code_language=SQL&code_format=false

### Erros comuns no campo ERROR:

- `ORA-12541: TNS: no listener` — listener do standby está down
- `ORA-12170: TNS: connect timeout` — problema de rede entre primary e standby
- `ORA-16058: standby database instance not available` — standby não está em mount/open

## 5. Archive Logs Recebidos e Aplicados — V$ARCHIVED_LOG

Para verificar quais logs foram recebidos e aplicados no standby, identificando possíveis lacunas:

```
SELECT
    thread#,
    MAX(sequence#) AS last_seq_received,
    MAX(CASE WHEN applied = 'YES' THEN sequence# END) AS last_seq_applied,
    MAX(sequence#) -
        MAX(CASE WHEN applied = 'YES' THEN sequence# END) AS gap
FROM
    v$archived_log
WHERE
    resetlogs_change# = (SELECT resetlogs_change# FROM v$database)
GROUP BY
    thread#
ORDER BY
    thread#;
```

| THREAD# | LAST_SEQ_RECEIVED | LAST_SEQ_APPLIED | GAP |
| --- | --- | --- | --- |
| 1 | 487 | 487 | 0 |

Interpretação

`GAP = 0` → todos os logs recebidos foram aplicados. Gap maior que zero indica logs pendentes (pode ser normal durante carga alta). Para ambientes RAC com múltiplas threads, cada thread deve ser verificada independentemente.

## 6. Detecção de Gaps de Archive — V$ARCHIVE_GAP

Quando há falha no transporte de redo, podem surgir gaps (lacunas) nos archive logs do standby:

```
SELECT
    thread#,
    low_sequence#,
    high_sequence#,
    (high_sequence# - low_sequence# + 1) AS missing_logs
FROM
    v$archive_gap;
```

Ação necessária

Se retornar linhas, existem logs faltando no standby. Ações corretivas: (1) verificar o status da destination no primary, (2) copiar manualmente os archives faltantes se necessário, (3) forçar um log switch no primary: `ALTER SYSTEM ARCHIVE LOG CURRENT;`. Se retornar vazio — sem gaps, transporte saudável.

## 7. V$DG_BROKER_CONFIG — Visão Completa da Configuração EXPANDIDA NO 26ai

No Oracle 26ai, `V$DG_BROKER_CONFIG` foi expandida com colunas essenciais que antes só estavam disponíveis via DGMGRL:

```
SELECT
    database,
    dataguard_role,
    status,
    severity,
    switchover_ready,
    failover_ready,
    transport_mode,
    current_state
FROM
    v$dg_broker_config;
```

▶ FreeSQL — Consultar V$DG_BROKER_CONFIG no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAIKUxJLEpMTiVB04L700sSglvig/BypWXJJYUloMZaeWpRZlllRCeeWZJckZ+UCh+KLUxBSoaFpiZg66WElRYl5xQX5RSXxufkoql1uQv69CmUpKenxSUX42UGlyfl5aZjoAjaE8fJcAAAA=&code_language=SQL&code_format=false

### Saída esperada:

| DATABASE | DATAGUARD_ROLE | STATUS | SEVERITY | SWITCHOVER_READY | FAILOVER_READY | TRANSPORT_MODE |
| --- | --- | --- | --- | --- | --- | --- |
| cdb1 | PRIMARY | 0 | SUCCESS | YES | UNKNOWN | -N/A- |
| cdb1_stby | PHYSICAL STANDBY | 0 | SUCCESS | YES | YES | ASYNC |

Por que isso é importante

Antes do 26ai, para saber se um standby estava `SWITCHOVER_READY` ou `FAILOVER_READY`, era necessário executar `SHOW DATABASE` no DGMGRL. Agora é possível monitorar via SQL puro, integrando com ferramentas como Zabbix, Grafana, OEM ou scripts Python.

## 8. V$DG_BROKER_PROPERTY — Propriedades do Broker via SQL NOVA NO 26ai

Nova view no 26ai que expõe **todas as propriedades do Broker** diretamente via SQL — sem necessidade do DGMGRL:

```
SELECT
    member,
    dataguard_role,
    property,
    value
FROM
    v$dg_broker_property
WHERE
    property IN (
        'LogXptMode',
        'ApplyLagThreshold',
        'TransportLagThreshold',
        'FastStartFailoverTarget',
        'FastStartFailoverThreshold',
        'NetTimeout',
        'RedoCompression',
        'StandbyFileManagement'
    )
ORDER BY
    member, property;
```

▶ FreeSQL — Consultar V$DG_BROKER_PROPERTY no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/21PSwrCMBDd5xRZCFXwElojCv1ALKirEsmQFpNOmKaF3t7Skq6c1Zv3GeY9RCbSivF5HLgP0HHBWgVlBkW6JrSwcp7QA4Vp3UZlB2BXWeZ83GlTfwi/QHU0sedNSLFl+L3g+yWXZGhePuSoIVkvJSfv7ZQpUzUEfYNWR6Ei1fUeKfwTCwhV6wCHEBkJGlN0fjb2LXbJgZXyIiQ/v2O37Z8fFEiiDvcAAAA=&code_language=SQL&code_format=false

### Saída esperada:

| MEMBER | DATAGUARD_ROLE | PROPERTY | VALUE |
| --- | --- | --- | --- |
| cdb1 | PRIMARY | LogXptMode | ASYNC |
| cdb1 | PRIMARY | NetTimeout | 300 |
| cdb1 | PRIMARY | RedoCompression | DISABLE |
| cdb1_stby | PHYSICAL STANDBY | ApplyLagThreshold | 0 |
| cdb1_stby | PHYSICAL STANDBY | FastStartFailoverThreshold | 30 |
| cdb1_stby | PHYSICAL STANDBY | StandbyFileManagement | AUTO |

Vantagens operacionais

Auditoria de configuração automatizada via SQL, comparação entre membros da configuração e detecção de inconsistências sem precisar logar no DGMGRL. A view tem colunas como `scope` e `valid_role` que indicam quais propriedades se aplicam a cada contexto.

## 9. V$DG_BROKER_ROLE_CHANGE — Histórico de Role Changes NOVA NO 26ai

Nova view que armazena o histórico dos últimos **10 role changes** (switchovers e failovers) da configuração:

```
SELECT
    event,
    old_primary,
    new_primary,
    TO_CHAR(begin_time, 'DD-MON-YYYY HH24:MI:SS') AS begin_time,
    TO_CHAR(end_time, 'DD-MON-YYYY HH24:MI:SS')   AS end_time,
    EXTRACT(SECOND FROM (end_time - begin_time))    AS duration_secs
FROM
    v$dg_broker_role_change
ORDER BY
    begin_time DESC;
```

▶ FreeSQL — Consultar V$DG_BROKER_ROLE_CHANGE no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAILUstS8Eh0wMz8nJb6gKDM3sagSIpCXWo4qkJSanpkXX5KZmwrhp+algHlcbkH+vgplKinp8UlF+dmpRfFF+Tmp8ckZiXnpqVz+QS6uQQpOkUjaFVxcg50BOxFPyoYAAAA=&code_language=SQL&code_format=false

### Saída esperada:

| EVENT | OLD_PRIMARY | NEW_PRIMARY | BEGIN_TIME | END_TIME | DURATION_SECS |
| --- | --- | --- | --- | --- | --- |
| Switchover | cdb1_stby | cdb1 | 29-JAN-2026 16:43:32 | 29-JAN-2026 16:43:45 | 13 |
| Switchover | cdb1 | cdb1_stby | 15-JAN-2026 08:10:05 | 15-JAN-2026 08:10:20 | 15 |

Por que isso é importante

Rastreabilidade completa de quem era primary e quando houve mudança de papel. Em casos de Fast-Start Failover, o motivo da transição também é registrado. Ideal para auditorias e Root Cause Analysis (RCA) pós-incidente.

## 10. DGMGRL com Saída JSON no 26ai 26ai

O Data Guard Broker CLI (DGMGRL) agora suporta saída em formato **JSON**, facilitando integração com ferramentas de automação como Ansible, Terraform e pipelines CI/CD:

```
# Na linha de comando do sistema operacional:
$ dgmgrl -outputformat json sys/password@cdb1 "SHOW CONFIGURATION"

# Parse com jq:
$ dgmgrl -outputformat json sys/password@cdb1 "SHOW CONFIGURATION" | jq '.members'
```

Benefício para automação

Isso elimina a necessidade de parsear texto livre com regex frágeis. Agora é possível integrar o resultado diretamente com `jq`, Python, PowerShell ou qualquer ferramenta que processe JSON — capturando dados estruturados como role, transport lag, apply lag e target lag.

## 11. PREPARE DATABASE FOR DATA GUARD 26ai

O 26ai introduziu o comando `PREPARE DATABASE FOR DATA GUARD` no Broker, que automatiza a preparação do primary para criação do standby:

```
DGMGRL> PREPARE DATABASE FOR DATA GUARD
        WITH DB_UNIQUE_NAME IS cdb1_stby
        DB_RECOVERY_FILE_DEST IS "+FRA"
        DB_RECOVERY_FILE_DEST_SIZE IS "400G"
        BROKER_CONFIG_FILE1 IS "+DATA/CDB1/dg1.dat"
        BROKER_CONFIG_FILE2 IS "+FRA/CDB1/dg2.dat";
```

Esse comando configura automaticamente: archivelog mode, force logging, standby redo logs, broker config files e demais parâmetros necessários. Isso reduz drasticamente os passos manuais e o potencial de erro humano durante o setup do Data Guard.

## 12. AWR Automático no Active Data Guard 26ai

No 26ai, os snapshots AWR no standby são coletados automaticamente e transportados para o primary — sem necessidade de configuração manual do RMF (Remote Management Framework). Os snapshots são coletados a cada hora com retenção de 8 dias.

```
-- Verificar snapshots AWR no standby
SELECT
    snap_id,
    dbid,
    instance_number,
    TO_CHAR(begin_interval_time, 'DD-MON-YYYY HH24:MI') AS snap_begin,
    TO_CHAR(end_interval_time, 'DD-MON-YYYY HH24:MI')   AS snap_end
FROM
    dba_hist_snapshot
ORDER BY
    snap_id DESC
FETCH FIRST 5 ROWS ONLY;
```

Os relatórios AWR podem ser gerados diretamente no standby usando `@?/rdbms/admin/awrrpt.sql`, exatamente como no primary. Isso é uma melhoria enorme para troubleshooting de performance em bancos standby.

### Bônus: Informações da instância no FreeSQL

Use o editor abaixo para consultar informações da instância Oracle 26ai:

▶ FreeSQL — Consultar V$INSTANCE no 26ai INTERATIVO

https://freesql.com/embedded/?layout=vertical&compressed_code=H4sIAFv+qmkC/wt29XF1DuFSAILMvOKSxLzk1Pi8xNxUHbBQRn5xCRK3LLWoODM/Lz6tNCcHIgLUUVJaDGGnJJYkJiUWp8YjC8INLcrPSeVyC/L3VShTgQkCALs2xht9AAAA&code_language=SQL&code_format=false

## 📋 Tabela de Referência Rápida

| View | Finalidade | 26ai? |
| --- | --- | --- |
| V$DATABASE | Papel do banco, modo de operação, protection mode | — |
| V$DATAGUARD_STATS | Apply lag, transport lag (somente no standby) | — |
| V$DATAGUARD_PROCESS | Status de todos os processos DG (MRP, RFS, DMON) | — |
| V$ARCHIVE_DEST | Configuração e erros das destinations | — |
| V$ARCHIVE_DEST_STATUS | Status de sincronização e gaps por destination | — |
| V$ARCHIVED_LOG | Logs recebidos/aplicados por thread e sequence | — |
| V$ARCHIVE_GAP | Gaps de archive logs no standby | — |
| V$DG_BROKER_CONFIG | Configuração completa do Broker (colunas expandidas) | EXPANDIDA |
| V$DG_BROKER_PROPERTY | Propriedades do Broker via SQL | NOVA |
| V$DG_BROKER_ROLE_CHANGE | Histórico de switchovers e failovers | NOVA |

## Conclusão

O Oracle AI Database 26ai representa um salto significativo na operação e monitoramento de ambientes Data Guard. As novas views (`V$DG_BROKER_PROPERTY`, `V$DG_BROKER_ROLE_CHANGE`) e as expansões na `V$DG_BROKER_CONFIG` eliminam a dependência do DGMGRL para muitas atividades de monitoramento, permitindo integração direta com ferramentas como Zabbix, Grafana, OEM e scripts de automação.

Combinando isso com a saída JSON do DGMGRL, o AWR automático no standby, as APIs `DBMS_DG` (com mais de 30 novas funções) e o suporte a Data Guard por PDB, o 26ai entrega um ecossistema de alta disponibilidade mais automatizado, observável e preparado para operações modernas de DevOps e cloud híbrida.

Essas consultas podem — e devem — ser incorporadas nos seus health checks diários, scripts de monitoramento e dashboards operacionais para garantir que seu ambiente Data Guard esteja sempre saudável e pronto para qualquer cenário de falha.

*Powered by [FreeSQL](https://freesql.com) — SQL interativo no Oracle AI Database 26ai.*
