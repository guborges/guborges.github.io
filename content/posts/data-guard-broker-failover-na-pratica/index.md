---
title: "Data Guard Broker: FAILOVER na prática"
date: 2025-12-23T17:33:19-03:00
draft: false
categories: ["Data Guard", "Oracle DBA"]
tags: ["Data Guard", "Oracle"]
cover:
  image: "failover.png"
---

Esse é o segundo post da sequência. No primeiro eu falei só da diferença entre failover e switchover. Agora é mão na massa: como eu executo um failover usando Data Guard Broker, com os checkpoints que eu realmente olho quando quero fazer isso com segurança.

Eu estou usando o lab que vem nos treinamentos do meu mentor. Isso facilita demais para estudar e repetir o procedimento quantas vezes eu quiser, porque o ambiente já vem montado do jeito certo para treinar cenário de incidentes, sem risco e sem depender de infra real.

Vou manter os nomes dos bancos exatamente como eu uso no meu lab:

- primário: AUTO_U_BR
- standby: AUTO_U_US

A ideia aqui é promover o standby e deixar o ambiente pronto para o próximo passo, que é reintegrar o antigo primário com reinstate.

## Antes de qualquer comando: o que eu quero garantir

Failover não é só rodar comando. É tomar decisão com base em estado do ambiente. Antes de promover o standby eu valido:

- o Broker está consistente e enxergando os membros corretamente
- o standby está saudável e com apply ok
- flashback está habilitado no standby, porque eu quero reinstate depois
- FRA tem espaço, porque flashback sem espaço vira problema

Se isso não estiver redondo, eu não avanço.

## Eu começo pelos logs, não pelo DGMGRL

Primeira regra: failover sem log aberto é pedir para tomar susto.

Eu deixo dois terminais no standby: um para executar e outro só para acompanhar alert.log e o log do Broker.

```text
tail -200f /u01/app/oracle/diag/rdbms/auto_u_us/AUTOUS/trace/alert_AUTOUS.log
tail -100f /u01/app/oracle/diag/rdbms/auto_u_us/AUTOUS/trace/drcAUTOUS.log
```

Se eu ainda tiver acesso ao primário antes da falha, eu deixo o log dele pronto também:

```text
tail -200f /u01/app/oracle/diag/rdbms/auto_u_br/AUTOBR/trace/alert_AUTOBR.log
tail -100f /u01/app/oracle/diag/rdbms/auto_u_br/AUTOBR/trace/drcAUTOBR.log
```

## Simulando a falha do primário no lab

No lab eu simulo perda total do primário do jeito mais “real”: queda abrupta. Eu faço power off na VM do AUTO_U_BR.

A ideia aqui é simples: simular aquele cenário clássico de queda de host, queda de energia ou travamento geral, onde o primário não responde mais.

Em produção, isso vira uma validação operacional: antes do failover, você tem que ter certeza que o primário está realmente fora, para não virar confusão depois.

## Validando o estado no Broker (ponto de decisão)

Com o primário indisponível, eu conecto no Broker a partir do standby:

```text
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW CONFIGURATION LAG;

SHOW DATABASE VERBOSE 'AUTO_U_US';
SHOW DATABASE VERBOSE 'AUTO_U_BR';
```

O que eu procuro aqui:

- standby AUTO_U_US com status bom
- primário AUTO_U_BR como not reachable, error ou equivalente
- lag conhecido, para eu entender onde estou pisando

Se o broker ainda estiver mostrando o primário como ok por alguns minutos, eu não forço no impulso. Eu espero o monitoramento convergir.

## Flashback: eu confiro aqui porque o “eu do futuro” agradece

Se eu quero reinstate depois, eu preciso de flashback habilitado no standby antes do failover.

```text
SELECT flashback_on FROM v$database;
SHOW PARAMETER db_recovery_file_dest;
```

Se não estiver ativo:

```text
ALTER DATABASE FLASHBACK ON;
```

Esse passo muda totalmente o pós-evento, porque abre caminho para reintegrar o antigo primário sem rebuild do zero.

## Executando o failover

Com o standby validado, eu executo o failover via Broker:

```text
FAILOVER TO 'AUTO_U_US';
```

A partir daqui eu acompanho os logs e volto no broker para enxergar a transição.

## Confirmando que virou primário de verdade

No Broker:

```text
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW CONFIGURATION LAG;

SHOW DATABASE VERBOSE 'AUTO_U_US';
SHOW DATABASE VERBOSE 'AUTO_U_BR';
```

No SQL:

```text
SELECT name, open_mode, database_role FROM v$database;
```

O esperado é:

- database_role como PRIMARY
- open_mode como READ WRITE

## A parte que derruba aplicação mesmo com banco ok: serviço e conectividade

Depois do failover, eu valido conexão. No lab eu faço um teste simples:

```text
sqlplus sh/oracle@srvora3:1521/AUTOUS_SVC
```

E se eu quiser validar escrita:

```text
CREATE TABLE x AS SELECT * FROM dba_objects;
SELECT COUNT(1) FROM x;
```

Em produção, aqui é onde você valida o que a aplicação realmente usa (service, SCAN, tnsnames, endpoint).

Failover resolve o problema do serviço voltar. Mas o ambiente ainda fica sem redundância.

No próximo post eu fecho o ciclo: como reintegrar o AUTO_U_BR como Physical Standby usando REINSTATE, aproveitando flashback e evitando rebuild com RMAN.
