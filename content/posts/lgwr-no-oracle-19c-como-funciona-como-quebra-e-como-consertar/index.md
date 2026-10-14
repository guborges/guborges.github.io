---
title: "LGWR no Oracle 19c: como funciona, como quebra e como consertar"
date: 2026-03-30T07:00:00-03:00
draft: false
categories: ["Oracle DBA", "SQL & PL/SQL"]
tags: ["Oracle", "Script Oracle"]
cover:
  image: "lgwr.png"
---

O LGWR é o processo que define quanto tempo cada COMMIT leva no seu banco. Quando ele atrasa, toda a aplicacao sente. Não importa se as queries estão rapidas ou se o storage esta com I/O bom nos dashboards. Se o LGWR não da conta, o commit trava e o usuario reclama.

Este post cobre o funcionamento interno do LGWR no 19c, como diagnosticar problemas de commit latency e o que fazer para resolver.

## O cenário

> Oracle 19c (19.8+), standalone ou RAC, ASM com disk groups separados para DATA e FRA. Workload OLTP com taxa de commits entre 200 e 2000 por segundo.

O quadro classico: a aplicação fica lenta. O AWR mostra "log file sync" no topo dos wait events. O time de storage jura que o I/O esta normal. O DBA abre o relatorio e nao sabe por onde comecar.

## Como o LGWR funciona

Quando uma sessão faz COMMIT, ela copia o redo record para o redo log buffer na SGA e posta o LGWR via semaforo. O LGWR pega o conteudo do buffer e grava nos online redo log files em disco.

O LGWR grava nestas situacoes:

- Um processo faz COMMIT ou ROLLBACK
- Passaram 3 segundos desde a ultima escrita
- O redo log buffer atingiu 1/3 da capacidade ou 1MB
- O DBWn precisa gravar dirty buffers (o redo tem que ir primeiro, por causa do write-ahead logging)
- Log switch

Depois de gravar, o LGWR sinaliza a sessao de volta. O tempo total dessa espera e o wait event "log file sync". Na pratica, e a commit latency da sua aplicacao.

Uma confusão comum: o DBWn nao tem nada a ver com a durabilidade do commit. Quem garante que a transação foi persistida e o LGWR. O DBWn grava dirty buffers em outro ritmo, independente. Se o banco cair, o SMON usa o redo (gravado pelo LGWR) para refazer as alterações na recovery.

## LGWR workers (LGnn) no 19c

A partir do 12c, o LGWR pode delegar a escrita fisica para workers auxiliares (LG00, LG01, etc.). Eles gravam o redo no disco e postam as sessoes de volta.

```
$ ps -ef | grep ora_lg
oracle  14346     1  0 11:10 ?  00:00:00 ora_lgwr_mydb
oracle  14350     1  0 11:10 ?  00:00:00 ora_lg00_mydb
oracle  14354     1  0 11:10 ?  00:00:00 ora_lg01_mydb
```

O parametro `_use_single_log_writer` controla isso:

- `ADAPTIVE` (default no 19c): Oracle decide sozinho quando ativar ou desativar os workers
- `TRUE`: forca modo single, so o LGWR trabalha
- `FALSE`: forca modo scalable, sempre com workers

O LGWR trace registra as transicoes:

```
kcrfw_slave_adaptive_updatemode: scalable->single group0=375 all=384 delay=144
Adaptive scalable LGWR disabling workers
```

No 19c ha bugs conhecidos em que o modo adaptativo causa problemas. Se voce vir wait events como "LGWR any worker group" ou "LGWR worker group ordering" no AWR, faz sentido testar `_use_single_log_writer=TRUE`. Mas faca isso com suporte do Oracle. E um parametro oculto.

## O problema: log file sync alto

O "log file sync" não é um evento de I/O direto. E o tempo total que a sessão espera pelo LGWR. As etapas desse ciclo:

1. Acordar o LGWR (se estava idle)
2. LGWR coleta o redo e emite o I/O
3. Esperar o I/O completar (este e o "log file parallel write")
4. Pos-processamento
5. LGWR posta a sessao de volta
6. Sessao acorda

Se o gargalo esta na etapa 3, o problema é I/O. Se está nas etapas 5 e 6, o problema é CPU scheduling. Essa distinção é o que separa um diagnostico util de um chute.

## Diagnostico

### Comparar log file sync com log file parallel write

```
SELECT event,
       total_waits,
       ROUND(time_waited_micro/1000) AS total_ms,
       ROUND(time_waited_micro/total_waits/1000, 2) AS avg_wait_ms
FROM   v$system_event
WHERE  event IN ('log file sync', 'log file parallel write')
ORDER BY event;
```

Se o avg_wait_ms de "log file parallel write" esta proximo do "log file sync", o gargalo e I/O puro. Se "log file parallel write" esta baixo (abaixo de 5ms) e "log file sync" esta alto (acima de 15ms), o problema esta no CPU scheduling ou na sinalizacao entre processos.

### Verificar taxa de commits

```
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('user commits',
                'user rollbacks',
                'redo writes',
                'redo size',
                'redo wastage',
                'redo buffer allocation retries');
```

Se user commits por segundo passa de 500, a aplicação provavelmente comita com frequencia excessiva. Cada commit exige um round-trip ao LGWR.

Outro indicador: se redo wastage esta alto em relacao ao redo size, os commits sao pequenos e frequentes, deixando espaço desperdiçado nos blocos de redo.

### Verificar o que o LGWR esta fazendo agora

```
SELECT sid,
       event,
       state,
       seconds_in_wait
FROM   v$session
WHERE  program LIKE '%LGWR%'
   OR  program LIKE '%LG0%';
```

Se o LGWR esta em "log file parallel write", está esperando I/O. Se esta em "rdbms ipc message", esta idle (comportamento normal entre escritas). Se esta em "LGWR wait for redo copy", tem contenção no buffer.

### Procurar escritas lentas no trace

O LGWR registra automaticamente escritas acima de 500ms:

```
$ grep "Warning: log write" $ORACLE_BASE/diag/rdbms/$DB_NAME/$INSTANCE/trace/*lgwr*.trc

Warning: log write elapsed time 608ms, size 23848KB
```

Se voce encontrar multiplos warnings no mesmo intervalo, o storage esta com problemas intermitentes. O time de infra vai dizer que "o I/O medio esta normal", mas o problema sao os picos, não a média.

## Solução

### Caso 1: I/O lento (log file parallel write acima de 10ms)

Mover redo logs para storage mais rapido. Em ASM, colocar em disk group dedicado com SSD ou NVMe.

Nao colocar redo em RAID-5 ou RAID-6. A penalidade de escrita com paridade causa picos de latencia. RAID-1 ou RAID-10 sao as opcoes corretas para redo.

Verificar o filesystemio_options:

```
SHOW PARAMETER filesystemio_options;
```

O valor recomendado e SETALL (habilita async e direct I/O). Sem isso, o LGWR pode estar fazendo buffered I/O, adicionando latencia desnecessaria.

Garantir que os redo logs não dividem disco com datafiles ou archive logs. Parece óbvio, mas acontece mais do que deveria.

### Caso 2: commits excessivos

Se a taxa de commits esta alta e o "log file parallel write" esta normal, o problema é volume. Cada commit força uma escrita sincrona.

A correção certa e na aplicação: agrupar operações e comitar em batch. Em PL/SQL, o Oracle ja otimiza: dentro de um bloco, o "log file sync" so acontece no final, nao a cada COMMIT.

Se a aplicação não pode ser alterada e perder dados em caso de crash e tolerável:

```
ALTER SESSION SET COMMIT_WAIT = NOWAIT;<br>ALTER SESSION SET COMMIT_LOGGING = BATCH;
```

O `COMMIT_WAIT` aceita `WAIT` (default), `NOWAIT` e `FORCE_WAIT`. O `NOWAIT` retorna sem esperar o LGWR confirmar a escrita. Se o banco cair nesse intervalo, o commit pode ser perdido.

Use apenas para cargas de dados em massa. Nunca em transações financeiras. A documentacao oficial do 19c (Oracle Database Reference, COMMIT_WAIT) avisa explicitamente que isso viola o D do ACID.

Um detalhe que a documentação nao deixa claro mas que foi confirmado por Frits Hoogland: quando você seta COMMIT_LOGGING=BATCH, o Oracle silenciosamente muda o comportamento do wait tambem. O LGWR simplesmente não recebe o sinal de commit e so grava na proxima passagem de 3 segundos. Tenha isso em mente.

### Caso 3: CPU starvation no LGWR

Se o LGWR não consegue CPU, ele atrasa para gravar e para postar as sessões de volta. Verifique:

```
$ vmstat 5 10
```

Se a coluna `r` (run queue) esta consistentemente acima do numero de CPUs, ha contenção.

No 19c, o LGWR roda com prioridade normal (SCHED_OTHER) por padrao. A partir do 21c, ele ja vem incluso no `_high_priority_processes`. No 19c, você pode adicionar:

```
ALTER SYSTEM SET "_high_priority_processes"='LMS*|VKTM|LGWR' SCOPE=SPFILE;
```

Requer restart. Depois, confirme:

```
$ ps -eo pid,class,pri,nice,args | grep ora_lgwr<br>26058 RR  41    - ora_lgwr_mydb
```

O `RR` indica Round-Robin scheduling, com prioridade elevada. Cuidado: nao coloque LGWR na prioridade mais alta (RT) sem entender as consequências. Um processo RT pode monopolizar a CPU e bloquear interrupções de I/O que o proprio LGWR precisa para funcionar.

Esse parametro e oculto. Consulte o Oracle Support antes (MOS Doc ID 1373500.1).

## Validação

Depois de qualquer mudança, valide com um AWR de pelo menos 30 minutos em horario de pico:

```
SELECT snap_id,
       event_name,
       ROUND(time_waited_micro/total_waits/1000, 2) AS avg_ms
FROM   dba_hist_system_event
WHERE  event_name IN ('log file sync', 'log file parallel write')
AND    snap_id BETWEEN &amp;snap_antes AND &amp;snap_depois
ORDER BY snap_id, event_name;
```

Metas razoaveis: "log file sync" abaixo de 5ms em SSD/NVMe, abaixo de 10ms em HDD. "log file parallel write" abaixo de 2ms em SSD, abaixo de 10ms em HDD.

## Troubleshooting

### Log file switch (checkpoint incomplete)

Sessoes esperando nesse evento. O LGWR precisa trocar de redo log group, mas o DBWn nao terminou de gravar os dirty buffers do grupo anterior.

Causa mais comum: redo log groups pequenos demais. A recomendacao e ter switches a cada 15-20 minutos.

```
SELECT ROUND(24*60/COUNT(*)) AS mins_entre_switches
FROM   v$log_history
WHERE  first_time > SYSDATE - 1;
```

Se o resultado for menor que 10, aumente o tamanho dos redo log groups.

### Log file sync intermitente em RAC 19c

Picos de "log file sync" apos upgrade para 19c, com mensagens "warning: log write broadcast wait time xxx ms" no trace do LGWR. Causa: bug de redo write broadcast em RAC. MOS Doc ID 2699312.1. Aplique o patch indicado ou teste `_use_single_log_writer=TRUE` como workaround.

### LGWR travado em log file parallel write

O LGWR fica preso em "log file parallel write" por segundos. Sessoes acumulam em "log file sync". O banco parece congelado.

Causa: storage com latencia alta. Em Exadata, pode ser flash disk com falha (MOS Doc ID 2638456.1). Fora de Exadata, rode `iostat -x 5` e verifique await e %util dos devices dos redo logs.

## Erros comuns

| Situacao | O que da errado | O que fazer |
| --- | --- | --- |
| LOG_BUFFER muito grande | Threshold de 1/3 fica alto, LGWR acumula redo demais por escrita | Deixar o Oracle auto-tunar |
| Redo logs em RAID-5 | Picos de latencia por penalidade de paridade | Migrar para RAID-1 ou RAID-10 |
| COMMIT a cada INSERT em loop | Centenas de commits/segundo, cada um esperando LGWR | Comitar em batch |
| `_use_single_log_writer` alterado sem necessidade | Pode degradar performance em workloads que se beneficiam do modo scalable | Deixar em ADAPTIVE, exceto se houver problema documentado |
| Redo logs no mesmo disco que datafiles | Contencao de I/O entre LGWR e DBWn | Isolar redo em storage dedicado |

## Boas praticas

- Manter pelo menos 3-4 redo log groups com 2 members multiplexados em discos separados
- Dimensionar redo logs para switches a cada 15-20 minutos
- Isolar redo em storage dedicado
- Monitorar "log file sync" e "log file parallel write" no AWR continuamente, nao so quando ha reclamacao
- Em RAC 19c, acompanhar patches de LGWR nas MOS notes de known issues

Nunca trace o LGWR com SQL Trace (10046) em producao sem absoluta necessidade. O overhead pode piorar o problema e, em casos extremos, causar hang. Use ASH ou Snapper para coletar dados sem impacto.

Parametros ocultos (_use_single_log_writer, _high_priority_processes) nao sao suportados oficialmente. Alteracoes devem ser feitas com orientacao do Oracle Support e documentadas. Erros nesses parametros podem causar corrupcao ou degradacao severa.

`COMMIT_WAIT=NOWAIT` viola a durabilidade ACID. Use apenas quando perda de dados em caso de crash e aceitavel e documentada.

O LGWR define a latencia de commit do banco. Diagnosticar problemas com ele exige separar I/O de CPU, olhar os wait events corretos e saber quando o problema esta na aplicacao (commits demais) e nao no banco. As queries e procedimentos deste post cobrem os cenarios mais frequentes em ambientes Oracle 19c.
