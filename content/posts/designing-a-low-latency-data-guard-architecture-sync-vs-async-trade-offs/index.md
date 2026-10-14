---
title: "Designing a Low-Latency Data Guard Architecture: SYNC vs ASYNC Trade-offs"
date: 2026-02-13T08:57:37-03:00
draft: true
categories: ["Uncategorized"]
---

## Resumo executivo

Synchronous redo transport (SYNC) existe para você conseguir **RPO 0** (zero data loss) quando o primary cai do nada. O custo é simples e inevitável: **latência de rede entra no caminho do COMMIT**. O evento que vai “gritar” no AWR/ASH é quase sempre **log file sync**, porque ele mede exatamente o tempo do COMMIT esperando o LGWR fazer o trabalho e responder.

Asynchronous redo transport (ASYNC) existe para você proteger dados **sem colocar a rede no caminho do COMMIT**. O COMMIT responde assim que o redo local foi escrito, e o primary não espera o standby acusar recebimento. Isso mantém throughput, mas você abre a porta para **perda pequena de dados** (RPO > 0) dependendo de lag e do instante da falha.

O ponto menos entendido (e o mais útil) é que SYNC não é “uma coisa só”. Em Maximum Availability você escolhe entre:

- **SYNC/AFFIRM**: espera confirmação de escrita em disco no standby redo log (proteção mais forte, custo maior).
- **SYNC/NOAFFIRM (FastSync)**: espera só o redo chegar no standby e iniciar I/O; não espera persistir em disco (custo menor, janela pequena de risco em falhas simultâneas).

Se você precisa de RPO 0 e o standby está longe demais para SYNC “direto”, sua melhor carta é **Far Sync**: um hop síncrono curto (perto do primary) para manter latência aceitável e um hop assíncrono longo para o standby remoto. Oracle descreve isso explicitamente como forma de reduzir impacto no commit e ainda permitir zero-data-loss failover para o destino terminal.

O artigo abaixo te dá: base técnica do redo transport, impacto no commit, trade-offs de RPO/RTO, topologias recomendadas, exemplos de configuração e um playbook direto de decisão.

## Contexto e objetivo

Data Guard resolve dois problemas diferentes ao mesmo tempo: **disaster recovery** e **alta disponibilidade**. O seu desenho precisa responder, sem enrolação, a duas perguntas:

1. Qual é o **RPO** aceitável (quanto dado você pode perder na pior falha)?
2. Qual é o **RTO** aceitável (quanto tempo você pode ficar fora)?

Essas duas metas determinam o modo de proteção (Maximum Performance / Availability / Protection) e, por consequência, o comportamento do transporte (SYNC vs ASYNC) e onde você paga a conta (latência/throughput vs risco de perda).

O objetivo específico aqui é desenhar uma arquitetura **low-latency**: manter commit rápido no primary, sem “matar” a aplicação, e ainda assim entregar o nível de proteção exigido. O segredo é entender onde o Data Guard adiciona espera (ou não adiciona) e como medir isso com dados, não com opinião.

## Redo transport de ponta a ponta: LGWR, standby redo log e o custo da rede

O Data Guard não replica “blocos de dados”. Ele replica **redo** (change vectors). O redo é gerado no primary, gravado nos online redo logs locais e usado para manter as réplicas sincronizadas.

O caminho físico do redo no destino também é objetivo: o redo recebido via redo transport é gravado no **standby redo log (SRL)** por um processo **RFS**, e o ciclo de “encher e arquivar” é espelhado no standby. Isso vale tanto para SYNC quanto para ASYNC.

Dois detalhes que viram incidente em produção:

- O SRL precisa ter **tamanho pelo menos igual** ao maior online redo log do source, e Oracle recomenda manter todos do mesmo tamanho por simplicidade.
- O SRL precisa ter **pelo menos um grupo a mais** que o redo log do source **por thread** (RAC: por thread). Se você não acompanha isso, o standby pode ficar temporariamente “fora de sincronia” após log switch e isso pode impedir zero data loss ou até derrubar primary em Maximum Protection em cenários específicos.

Agora o ponto central de latência:

### O commit e o “log file sync”

Quando uma sessão faz COMMIT, ela espera o LGWR: CPU, submissão de I/O, espera de I/O e retorno para “postar” as sessões. Esse tempo inteiro aparece como **log file sync**.

O que confunde muita gente é que log file sync pode explodir por efeito cascata: um único log write lento (log file parallel write) pode colocar milhares de sessões esperando no log file sync. Oracle descreve exatamente esse efeito.

### Onde a rede entra

Em **SYNC**, o primary não conclui o envio (e depende do modo, não conclui o commit “protegido”) até receber **acknowledgement** do destino. Oracle define SYNC como uso de modo síncrono e define NET_TIMEOUT como o quanto o **LGWR** espera por esse ACK antes de derrubar a sessão e registrar erro.

Em termos práticos: se você coloca SYNC em um link com RTT alto, jitter e perda intermitente, você está convidando o COMMIT a pagar essa conta. Esse custo vai aparecer em:

- aumento de **log file sync** no primary
- aumento de response time do destino SYNC medido por **V$REDO_DEST_RESP_HISTOGRAM**

## SYNC vs ASYNC, AFFIRM vs NOAFFIRM e FORCE LOGGING: o que muda de verdade

### Definições objetivas (sem marketing)

**AFFIRM vs NOAFFIRM** não é “detalhe”. É o contrato do ACK do destino:

- **AFFIRM**: o destino só confirma depois de escrever o redo no SRL.
- **NOAFFIRM**: o destino confirma antes de escrever no SRL.

Defaults importam: se você não especificar, o padrão é **AFFIRM quando SYNC** e **NOAFFIRM quando ASYNC**.

**Maximum Availability** (o modo que a maioria quer) deixa isso explícito:

- Você não comita até o redo estar no online redo local **e**, dependendo da configuração, ou:
  - o redo foi recebido e o I/O do SRL foi iniciado e o ACK voltou, ou
  - o redo foi recebido e escrito no SRL e o ACK voltou.
- Se o primary não recebe ACK de pelo menos um standby sincronizado, ele se comporta como Maximum Performance para preservar disponibilidade até recuperar a condição.

**Maximum Performance** é direto: COMMIT confirma quando o redo local foi escrito; envio para standby ocorre de forma assíncrona em relação ao commit, então o primary não paga o tempo de rede/ACK.

**Maximum Protection** adiciona a regra “sem brincadeira”: se não conseguir ACK do standby, o primary **shutdown** ao invés de continuar com transações “desprotegidas”. Oracle ainda recomenda mínimo de dois standbys para evitar que uma falha isolada derrube o primary nesse modo.

### O impacto no commit (por que SYNC sente e ASYNC quase não sente)

Oracle descreve o comportamento do ASYNC sem rodeio: ele reconhece sucesso do commit assim que o write local termina e **não espera** o standby acusar recebimento; isso evita impacto no desempenho do primary, mas não garante que todo redo committed esteja no standby a qualquer instante.

E Oracle também descreve o contraste do SYNC: o primary espera confirmação do standby e, em SYNC/AFFIRM, isso envolve “recebido e escrito em disco (SRL)” antes de sinalizar commit success.

### Tabela comparativa: SYNC vs ASYNC (visão de decisão)

A tabela abaixo é um resumo operacional baseado no que Oracle define em proteção modes e em como SYNC/ASYNC + AFFIRM/NOAFFIRM se comportam na prática.

| Dimensão | SYNC (SYNC/AFFIRM ou SYNC/NOAFFIRM) | ASYNC (tipicamente ASYNC/NOAFFIRM) |
| --- | --- | --- |
| RPO | Pode ser **0** se houver standby sincronizado; em SYNC/NOAFFIRM existe janela pequena de risco (falhas simultâneas) | **Não garante 0**; pode haver perda pequena dependendo do lag e do instante da falha |
| Impacto de latência no COMMIT | **Sim**: rede entra no caminho do commit (direto ou indiretamente via ACK) | **Minimiza**: commit confirma no write local; não espera ACK do standby |
| Requisitos de rede | RTT baixo, jitter controlado, link previsível; NET_TIMEOUT vira parâmetro crítico | Bandwidth e estabilidade ajudam, mas o primary não “trava” esperando |
| Complexidade operacional | Maior: tuning de timeout, observabilidade de response time, risco de stalls | Menor para o primary; foco vai para monitorar lag e apply |
| Use case típico | OLTP crítico com exigência de “zero data loss” local/metro; Far Sync quando distância é grande | Proteção DR com foco em performance; replicação para regiões distantes; múltiplas cópias |

### Fast Sync (SYNC/NOAFFIRM): o meio termo útil

Oracle nomeia explicitamente **Fast Sync** como **SYNC NOAFFIRM**, com objetivo de reduzir round-trip total (ACK na memória) e diminuir impacto de synchronous transport no primary. E também alerta: isso pode introduzir uma exposição muito pequena a data loss se primary e standby falharem quase simultaneamente antes do I/O do standby completar.

### FORCE LOGGING: integridade de dados antes de performance “bonita”

FORCE LOGGING não é sobre latência, é sobre **não quebrar o standby** com operações NOLOGGING. Oracle é clara: o modo padrão (permitir certas cargas nonlogged) **não é apropriado** quando existe standby porque pode deixar dados “faltando” no standby e exigir correção manual. FORCE LOGGING impede cargas nonlogged, mas pode deixar o load mais lento pois o dado precisa ir para redo.

Se você está perseguindo low-latency de commit, usar NOLOGGING para “reduzir redo” pode parecer tentador. Em Data Guard, isso vira risco operacional. FORCE LOGGING é a linha de base de consistência.

## RPO/RTO, topologias e padrões: single standby, múltiplos destinos, Far Sync, FSFO e Broker

Aqui entra arquitetura. Ajustar parâmetro resolve parte do problema; topologia resolve o resto.

### Single standby (o básico)

- **Primary → Standby (SYNC)**: bom para RPO 0 quando a rede é “perto” e previsível; ruim quando RTT é alto.
- **Primary → Standby (ASYNC)**: performance do primary preservada; risco de perda pequena (RPO > 0).

### Multiple standby (um local + um remoto)

Oracle descreve benefício de multi-standby: manter um standby próximo (para proteção forte e failover rápido) e outro remoto (para desastre regional).

Esse padrão resolve o dilema clássico: você usa SYNC para o standby local e ASYNC para o remoto.

### Far Sync (o padrão mais eficiente para “RPO 0 em longa distância”)

Oracle descreve Far Sync assim:

- Você cria uma instância Far Sync **próxima do primary** para minimizar impacto no commit (por menor latência).
- Em falha do primary, assumindo Far Sync sincronizado, Far Sync e o standby terminal coordenam “final redo shipment” e fazem **zero-data-loss failover**.
- O Far Sync recebe redo em modo que atende a exigência de zero data loss e **encaminha em ASYNC** para o destino terminal distante.

Em português claro: você paga a latência SYNC só no trecho curto (metro) e empurra o trecho longo para ASYNC.

### Cascading standbys (alternativa útil, com restrições)

Oracle também descreve cascata: um standby “cascading” recebe redo do primary e repassa para até 30 destinos terminais, podendo ser em real-time ou non-real-time.

Atenções:

- Só **physical standby** faz cascata.
- **Real-time cascading** requer licença de **Active Data Guard** (segundo a documentação).
- Existe risco de proteção ligeiramente menor no destino terminal pelo “segundo hop” (Oracle aponta que o segundo hop cria potencial de perda adicional no terminal se um outage impedir redo de chegar).

### Fast-Start Failover (FSFO) e o risco real de split brain

FSFO é Broker automatizando failover: quando habilitado, o broker decide se failover é necessário e inicia failover para o target standby automaticamente.

O risco operacional aqui não é “achar bonito”. É **partição de rede** e duas bases assumindo primary se você fizer disable/force errado em cenário de desconexão. A documentação traz “Caution” explícito: em certas condições de lost connectivity ao desabilitar FSFO com FORCE, pode haver dois bancos assumindo primary.

### Diagramas de topologia

```
Site C - Remote DRSite B - MetroSite A - PrimarySYNC (AFFIRM ou NOAFFIRM)ASYNCPrimary DBStandby LocalStandby RemotoShow code
```

```
Remote - DRMetro - Far SyncSite A - PrimarySYNC (curto)ASYNC (longo)Primary DBFar Sync InstanceTerminal StandbyShow code
```

## Configuração e medição: parâmetros, Broker e observabilidade prática

### Exemplos de configuração (SPFILE/params)

A base de Data Guard passa por `LOG_ARCHIVE_DEST_n` com atributos corretos. Oracle documenta SYNC/ASYNC, NET_TIMEOUT, REOPEN, COMPRESSION, DB_UNIQUE_NAME, VALID_FOR e recomenda especificar VALID_FOR para continuar enviando redo após role transition.

Exemplo de **Maximum Availability com SYNC/AFFIRM** (ilustrativo):

```
sqlCopyALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(PRIM,STBY)' SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_DEST_1=
'LOCATION=USE_DB_RECOVERY_FILE_DEST
 VALID_FOR=(ALL_LOGFILES,ALL_ROLES)
 DB_UNIQUE_NAME=PRIM' SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
'SERVICE=STBY
 SYNC AFFIRM
 NET_TIMEOUT=10
 REOPEN=5
 VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
 DB_UNIQUE_NAME=STBY' SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE SCOPE=BOTH;
```

Notas importantes suportadas pela documentação:

- **NET_TIMEOUT** em SYNC controla quanto tempo o LGWR bloqueia esperando ACK; se não chega, a sessão é encerrada e erro é registrado.
- Oracle recomenda especificar NET_TIMEOUT em SYNC para controlar stall máximo por falha de redo transport.
- Se você não especifica AFFIRM/NOAFFIRM, o default muda conforme SYNC/ASYNC.

Exemplo de **Maximum Availability com FastSync (SYNC/NOAFFIRM)**:

```
sqlCopyALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
'SERVICE=STBY
 SYNC NOAFFIRM
 NET_TIMEOUT=10
 REOPEN=5
 VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
 DB_UNIQUE_NAME=STBY' SCOPE=BOTH;
```

Por que isso reduz impacto? Porque o ACK vem sem esperar escrita no SRL; Oracle descreve exatamente esse trade-off e a janela de risco em falhas simultâneas.

Exemplo de **Maximum Performance com ASYNC/NOAFFIRM**:

```
sqlCopyALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
'SERVICE=STBY
 ASYNC NOAFFIRM
 REOPEN=5
 VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
 DB_UNIQUE_NAME=STBY' SCOPE=BOTH;
```

Isso preserva performance do primary porque o envio é assíncrono em relação ao commit.

### TNS e listener: o básico que evita “log transport stuck”

A documentação de redo transport reforça que o `SERVICE` usa um Oracle Net service name resolvível e recomenda **dedicated server** no connect descriptor para redo transport.

Exemplo simples de `tnsnames.ora` (modelo):

```
iniCopySTBY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = stby-host)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = stby_dg)
      (SERVER = DEDICATED)
    )
  )
```

### Data Guard Broker: quando você quer consistência operacional

DGMGRL existe para você gerenciar config de forma centralizada; Oracle define isso como alternativa ao Cloud Control.

O ponto-chave para este artigo: com broker você configura redo transport mode de forma declarativa. A documentação diz que você usa `LogXptMode` ou `RedoRoutes` para setar **SYNC, ASYNC ou FASTSYNC**.

Exemplo mínimo (snippets):

```
sqlCopy-- no SQL*Plus
ALTER SYSTEM SET DG_BROKER_START=TRUE SCOPE=BOTH;
```

```
textCopy-- no DGMGRL
DGMGRL> CREATE CONFIGURATION dgconf AS PRIMARY DATABASE IS PRIM CONNECT IDENTIFIER IS PRIM;
DGMGRL> ADD DATABASE STBY AS CONNECT IDENTIFIER IS STBY MAINTAINED AS PHYSICAL;
DGMGRL> ENABLE CONFIGURATION;

-- modo de proteção e transporte
DGMGRL> EDIT CONFIGURATION SET PROTECTION MODE AS MAXAVAILABILITY;
DGMGRL> EDIT DATABASE PRIM SET PROPERTY LogXptMode='FASTSYNC';  -- SYNC/NOAFFIRM
DGMGRL> EDIT DATABASE STBY SET PROPERTY LogXptMode='FASTSYNC';
```

Se você habilitar FSFO, a doc mostra o comando e que ele pode operar “Zero Data Loss Mode” dependendo da configuração.

### Como medir latência, lag e impacto no commit (sem achismo)

**Métricas mínimas que você precisa coletar:**

- redo gerado (por segundo)
- commits por segundo
- **log file sync** (tempo e distribuição)
- transport lag e apply lag
- response time do destino SYNC
- gap detectado/resolvido

#### Log file sync e distribuição

A própria documentação de troubleshooting conecta log file sync diretamente ao trabalho do LGWR e mostra como usar histogramas para entender a “cauda” (picos).

Exemplos práticos:

```
sqlCopy-- visão rápida de waits (desde startup)
SELECT event, total_waits, time_waited/100 time_waited_s, average_wait/1000 avg_ms
FROM v$system_event
WHERE event IN ('log file sync','log file parallel write');
```

```
sqlCopy-- perfil de distribuição do log file sync
SELECT wait_time_milli, wait_count
FROM v$event_histogram
WHERE event = 'log file sync'
ORDER BY wait_time_milli;
```

Para AWR (modelo):

```
sqlCopy-- AWR: média de log file sync por snapshot (exemplo)
SELECT
  s.begin_interval_time,
  e.total_waits_delta,
  ROUND(e.time_waited_micro_delta/1000 / NULLIF(e.total_waits_delta,0), 3) AS avg_ms
FROM dba_hist_system_event e
JOIN dba_hist_snapshot s
  ON s.snap_id = e.snap_id AND s.instance_number = e.instance_number
WHERE e.event_name = 'log file sync'
ORDER BY s.begin_interval_time;
```

#### Lag no standby (transport/apply)

O caminho mais direto é `V$DATAGUARD_STATS`. A doc descreve apply lag/transport lag, e que os cálculos dependem de dados recebidos periodicamente do source (se não muda, suspeite de falta de recebimento).

```
sqlCopySELECT name, value, time_computed, datum_time
FROM v$dataguard_stats
WHERE name LIKE '%lag%';
```

#### Response time do destino SYNC

Use `V$REDO_DEST_RESP_HISTOGRAM`. A doc explica colunas, que serve para identificar performance issues em SYNC e para tuning de `NET_TIMEOUT`, e até fornece queries (min/max).

```
sqlCopySELECT frequency, duration
FROM v$redo_dest_resp_histogram
WHERE dest_id = 2 AND frequency > 1
ORDER BY duration;
```

```
sqlCopySELECT max(duration) AS max_s, min(duration) AS min_s
FROM v$redo_dest_resp_histogram
WHERE dest_id = 2 AND frequency > 1;
```

#### Estado e erros do transporte

- `V$ARCHIVE_DEST` e `V$ARCHIVE_DEST_STATUS` mostram destino, modo e status em runtime.
- `V$DATAGUARD_STATUS` expõe mensagens recentes sobre redo transport/log apply (espelho do alert/trace).

```
sqlCopySELECT dest_id, status, destination, error
FROM v$archive_dest_status
WHERE status <> 'INACTIVE';
```

```
sqlCopySELECT facility, severity, message_num, timestamp, message
FROM v$dataguard_status
ORDER BY timestamp DESC
FETCH FIRST 30 ROWS ONLY;
```

#### DGMGRL: sanity check rápido

Um comando que vira rotina:

```
textCopyDGMGRL> SHOW CONFIGURATION;
DGMGRL> SHOW CONFIGURATION LAG;
DGMGRL> SHOW DATABASE STBY;
```

E para FSFO, a doc manda você validar com `SHOW FAST_START FAILOVER`.

#### ADRCI: alert log com filtro (operacional)

Exemplo de uso (modelo):

```
textCopyadrci> show homes
adrci> set home diag/rdbms/prim/PRIM
adrci> show alert -tail 200
```

Use isso junto com `V$DATAGUARD_STATUS` para cruzar erro com momento exato.

## Testes e simulações, mitigations, riscos e checklist de implementação

### Como medir SYNC vs ASYNC em laboratório (metodologia que presta)

A metodologia é: **mudar uma variável por vez** e medir as mesmas métricas.

1. Baseline em ASYNC/NOAFFIRM
2. Trocar para SYNC/AFFIRM
3. Trocar para SYNC/NOAFFIRM (FastSync)
4. Introduzir latência/jitter artificial e repetir

Para simular impactos de rede, uma abordagem prática é usar traffic shaping no Linux (`tc`) para adicionar delay; existe material técnico demonstrando exatamente isso em contexto de Data Guard SYNC.

Gerador simples de carga de commits:

```
sqlCopy-- tabela
CREATE TABLE t_commit (id NUMBER, payload VARCHAR2(100), ts TIMESTAMP DEFAULT SYSTIMESTAMP);

-- carga: muito commit pequeno (pior caso para log file sync)
DECLARE
  i NUMBER := 0;
BEGIN
  FOR i IN 1..200000 LOOP
    INSERT INTO t_commit(id, payload) VALUES (i, RPAD('x',100,'x'));
    COMMIT;
  END LOOP;
END;
/
```

Variante para controlar taxa (sleep) e simular TPS estável:

```
sqlCopyDECLARE
  i NUMBER;
BEGIN
  FOR i IN 1..60000 LOOP
    INSERT INTO t_commit(id, payload) VALUES (i, 'x');
    COMMIT;
    DBMS_LOCK.SLEEP(0.01); -- 10ms
  END LOOP;
END;
/
```

Como forçar lag (sem destruir o ambiente):

- Apply lag: parar MRP no standby e observar `V$DATAGUARD_STATS` crescer; depois religar e medir catch-up.
- Transport lag: introduzir delay/restrição na rede (tc) ou interromper momentaneamente a conectividade (controlado) e observar `transport lag`.

### Exemplo “antes/depois” com números hipotéticos plausíveis

Cenário: OLTP com 2.500 commits/s, redo ~25 MB/s.

**Baseline (ASYNC/NOAFFIRM)**

- log file sync avg: 1,8 ms
- transport lag: varia entre 0s e 3s
- apply lag: 0s a 2s
- RPO: “quase zero na maior parte do tempo”, mas não garantido

**Mudança para SYNC/AFFIRM com RTT 8 ms**

- log file sync avg sobe para ~10–14 ms (RTT + overhead de standby I/O)
- throughput cai (mais sessões esperando commit)
- response histogram do destino mostra cauda (picos)
- RPO: 0 enquanto standby estiver sincronizado

**Ajuste para SYNC/NOAFFIRM (FastSync) no mesmo RTT**

- log file sync avg cai de ~12 ms para ~9 ms (ACK mais cedo)
- RPO: mantém sem perda na maioria dos casos, mas existe janela curta de risco em falhas simultâneas antes do I/O completar

**Redesenho com Far Sync (metro 1 ms RTT) + standby remoto (40 ms RTT)**

- Primary → Far Sync em SYNC (custo ~1–3 ms)
- Far Sync → remoto em ASYNC (sem colocar 40 ms no commit)
- Você aproxima commit latency do mundo “metro” e ainda consegue zero-data-loss failover para o destino terminal, conforme descrito na doc de Far Sync.

Os números acima são plausíveis porque o custo mínimo de SYNC inclui o ciclo de ACK (sensível a RTT) e, em AFFIRM, inclui a confirmação pós-escrita no SRL; a doc separa exatamente esses dois comportamentos.

### Mitigações (o que realmente reduz latência e instabilidade)

1. **Escolha correta do modo**

- Se você precisa RPO 0 e tem rede boa em metro: SYNC/AFFIRM ou FastSync dependendo do budget de latência e do apetite a risco.
- Se precisa RPO 0 com distância grande: Far Sync.
- Se performance manda e RPO > 0 é aceito: ASYNC.

2. **Timeouts e política de falha**

- Ajustar `NET_TIMEOUT` em SYNC para evitar stall exagerado; a doc alerta que, sem um valor pequeno e não-zero, o primary pode potencialmente stall por mais tempo e recomenda definir um valor controlado.
- Usar `REOPEN` e (quando aplicável) `MAX_FAILURE` para política de reconexão.

3. **Synchronous multi-destination sem “pagar” por todos**

- `DATA_GUARD_SYNC_LATENCY` limita quanto o LGWR espera por destinos síncronos adicionais depois que pelo menos um standby já acusou recebimento. Isso existe justamente para evitar que “o mais lento” degrade o commit quando você tem mais de um SYNC.

4. **Compressão de redo (quando faz sentido)**

- Atributo `COMPRESSION` existe e pode melhorar desempenho em links de baixa banda e alta latência; mas exige licença de Oracle Advanced Compression.
- Use como mitigação para gap resolution e WAN, não como “milagre” para RTT alto em SYNC (que é outra natureza de custo).

5. **Gap handling e operação**

- Oracle descreve que redo gap é detectado e resolvido automaticamente quando a transmissão retoma, e que o tempo para resolver é proporcional ao tamanho do gap e inversamente proporcional ao throughput do link; compressão pode reduzir esse tempo.

6. **Monitoramento e alerting**

- Lag: `V$DATAGUARD_STATS` (transport/apply).
- SYNC response: `V$REDO_DEST_RESP_HISTOGRAM` e `NET_TIMEOUT` tuning.
- Erros: `V$ARCHIVE_DEST_STATUS` + `V$DATAGUARD_STATUS`.

### Riscos e cenários de falha (onde você se machuca)

- **Network partition em SYNC**: se o primary não consegue ACK de pelo menos um standby sincronizado, ele pode operar como Maximum Performance temporariamente (Availability) ou shutdown (Protection).
- **Data loss em ASYNC**: por definição não há garantia de redo committed estar no standby em qualquer instante.
- **Data loss em FastSync (SYNC/NOAFFIRM)**: existe cenário especial de falhas simultâneas antes do I/O do standby completar.
- **Split brain / “dois primaries” com operação errada de FSFO**: a doc do broker alerta explicitamente sobre condições de rede desconectada e uso de FORCE ao desabilitar FSFO que podem levar a duas bases assumindo primary.

### Playbook de decisão (direto por RPO/RTO e latência)

Regra operacional: você escolhe primeiro por **RPO**, depois você otimiza por **latência**.

```
mermaidCopyflowchart TD
  A[Definir RPO/RTO e SLA] --> B{RPO precisa ser 0?}
  B -- "Nao" --> C[Use ASYNC/NOAFFIRM (MaxPerformance)\nMonitore lag e apply]
  B -- "Sim" --> D{Distancia/RTT permite SYNC sem matar o COMMIT?}
  D -- "Sim" --> E{Apetite a risco minimo em falha simultanea?}
  E -- "Zero tolerancia" --> F[SYNC/AFFIRM (MaxAvailability ou MaxProtection)\nTunar NET_TIMEOUT e monitorar response time]
  E -- "Aceita janela minima" --> G[SYNC/NOAFFIRM (FastSync)\nMonitorar e documentar risco]
  D -- "Nao" --> H[Use Far Sync: SYNC curto + ASYNC longo\n(ou local SYNC + remoto ASYNC)]
  H --> I{Quer failover automatico?}
  I -- "Sim" --> J[FSFO com Broker + Observer\nTestar particao de rede e playbooks]
  I -- "Nao" --> K[Failover manual com runbook\nTreinar e medir RTO]
```

Os blocos acima refletem exatamente o comportamento descrito por Oracle para protection modes, SYNC/ASYNC e FastSync, e o papel de Far Sync para reduzir impacto no commit.

### Checklist de implementação (publicável e executável)

- Definir RPO/RTO e escolher protection mode (MaxPerformance / MaxAvailability / MaxProtection).
- Confirmar SRLs: tamanho >= maior ORL e “+1 grupo por thread”; revisar sempre que adicionar redo no primary.
- Habilitar FORCE LOGGING (ou justificar formalmente exceção e plano de correção).
- Configurar `LOG_ARCHIVE_DEST_n` com `DB_UNIQUE_NAME` e `VALID_FOR` em todos os sites.
- Em SYNC, definir `NET_TIMEOUT` e validar `V$REDO_DEST_RESP_HISTOGRAM`.
- Se múltiplos destinos SYNC, avaliar `DATA_GUARD_SYNC_LATENCY`.
- Medir antes/depois: redo MB/s, commits/s, log file sync, transport/apply lag.
- Se usar Broker: padronizar `LogXptMode`/`RedoRoutes`, e só então habilitar FSFO com testes de partição de rede e runbooks.
- Definir alertas: lag thresholds, response time SYNC, erro em dest_status, e correlação com alert log.
