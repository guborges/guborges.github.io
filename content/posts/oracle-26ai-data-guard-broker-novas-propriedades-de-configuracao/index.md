---
title: "Oracle 26ai Data Guard Broker: Novas Propriedades de Configuração"
date: 2026-02-02T07:00:00-03:00
draft: false
categories: ["Data Guard", "Oracle 26Ai"]
tags: ["Data Guard", "Oracle"]
cover:
  image: "brokerconfig.png"
---

Estava montando um ambiente de Data Guard com Oracle 26ai aqui no lab e quando rodei um `SHOW CONFIGURATION VERBOSE` no DGMGRL me deparei com propriedades que não existiam no Oracle 19c.

Resolvi comparar a documentação oficial das duas versões e encontrei funcionalidades novas que resolvem problemas reais de produção.

Vou mostrar algumas propriedades novas do Data Guard Broker no Oracle 26ai, validadas contra a documentação oficial da Oracle.

## Comparação: Oracle 19c vs Oracle 26ai

Aqui está o output real do `SHOW CONFIGURATION VERBOSE` nas duas versões:

### Oracle 19c

```
Configuration - DG_AUTO
  Protection Mode: MaxPerformance
  Members:
  AUTO_U_BR - Primary database
    AUTO_U_US - Physical standby database
    
  Properties:
    FastStartFailoverThreshold      = '30'
    OperationTimeout                = '30'
    TraceLevel                      = 'USER'
    FastStartFailoverLagLimit       = '30'
    CommunicationTimeout            = '180'
    ObserverReconnect               = '0'
    FastStartFailoverAutoReinstate  = 'TRUE'
    FastStartFailoverPmyShutdown    = 'TRUE'
    BystandersFollowRoleChange      = 'ALL'
    ObserverOverride                = 'FALSE'
    ExternalDestination1            = ''
    ExternalDestination2            = ''
    PrimaryLostWriteAction          = 'CONTINUE'
    ConfigurationWideServiceName    = 'AUTO_CFG'

Fast-Start Failover:  Disabled
Configuration Status: SUCCESS
```

### Oracle 26ai

```
Configuration - DG_CARECA
  Protection Mode: MaxPerformance
  Members:
  careca   - Primary database
    CARECADG - Physical standby database
    
  Properties:
    BystandersFollowRoleChange      = 'ALL'
    CommunicationTimeout            = '180'
    ConfigurationSimpleName         = 'DG_CARECA'
    ConfigurationWideServiceName    = 'careca_CFG'
    DrainTimeout                    = '0'
    ExternalDestination1            = ''
    ExternalDestination2            = ''
    FastStartFailoverAutoReinstate  = 'TRUE'
    FastStartFailoverLagGraceTime   = '0'
    FastStartFailoverLagLimit       = '30'
    FastStartFailoverLagType        = 'APPLY'
    FastStartFailoverPmyShutdown    = 'TRUE'
    FastStartFailoverThreshold      = '30'
    ObserverOverride                = 'FALSE'
    ObserverPingInterval            = '0'
    ObserverPingRetry               = '0'
    ObserverReconnect               = '0'
    OperationTimeout                = '30'
    PrimaryDatabaseCandidates       = ''
    PrimaryLostWriteAction          = 'CONTINUE'
    TraceLevel                      = 'USER'

Fast-Start Failover:  Disabled
Configuration Status: SUCCESS
```

Repare que apareceram 7 propriedades novas que não existem no Oracle 19c.

## 1. PrimaryDatabaseCandidates

Esta propriedade permite restringir quais databases podem se tornar primary durante switchover ou failover.

Segundo a documentação oficial:

> "The PrimaryDatabaseCandidates configuration property allows the restriction of switchover and failover to a list of databases. This property is a list of DB_UNIQUE_NAMEs of the databases to which Data Guard broker will allow switchover and failover when FSFO is disabled."

Fonte: Oracle Database 26ai Data Guard Broker Properties

### Sintaxe

```
DGMGRL> EDIT CONFIGURATION SET PROPERTY PrimaryDatabaseCandidates = 'DB1,DB2,DB3';
```

### Exemplo da Documentação Oracle

Cenário: 6 databases na configuração (db1, db2, db3, b4, b5, b6)

```
DGMGRL> EDIT CONFIGURATION SET PROPERTY PrimaryDatabaseCandidates = 'db1,db2,db3';
Property "primarydatabasecandidates" updated
```

Resultado:

- Switchover/Failover permitido APENAS para db1, db2 ou db3
- Tentativa de switchover/failover para b4, b5 ou b6 será rejeitada
- Erro retornado: ORA-16529

### ORA-16529

Quando você tenta fazer switchover ou failover para um database que não está na lista de candidatos:

```
ORA-16529: Switchover and failover operations are restricted to databases 
           specified in the PrimaryDatabaseCandidates configuration property
```

Fonte: Oracle Database 26ai Error Messages

### Casos de Uso

1. Multi-standby com propósitos diferentes
2. Standby de reporting que nunca deve virar primary
3. Standby de compliance/auditoria
4. Cascaded standby que não pode ser promovido diretamente

### Comportamento com Fast-Start Failover

Segundo a documentação:

> "Note that, when FSFO is enabled, this property does not add any new restrictions since Data Guard Broker already has a property FastStartFailoverTarget that lists which databases can be new primary databases."

Ou seja, quando FSFO está habilitado, a propriedade `FastStartFailoverTarget` é que controla.

### Adição Automática do Primary

A documentação menciona:

```
DGMGRL> EDIT CONFIGURATION SET PROPERTY PrimaryDatabaseCandidates = 'DB2';
Warning: ORA-16580: The primary database was added to property PrimaryDatabaseCandidates
Property "primarydatabasecandidates" updated
```

O Broker automaticamente adiciona o database primary atual à lista de candidatos se você não incluí-lo.

## 2. DrainTimeout

Esta propriedade substitui a opção `WAIT` do comando SWITCHOVER, que foi deprecated no Oracle 26ai.

Segundo a documentação oficial:

> "The DrainTimeout configuration property specifies the number of seconds you want to wait for sessions to drain before the broker proceeds with switchover, including switchovers performed as part of a database rolling upgrade using DBMS_ROLLING."

Fonte: Oracle Database 26ai Data Guard Broker Properties

### Sintaxe

```
DGMGRL> EDIT CONFIGURATION SET PROPERTY DrainTimeout = 120;
```

### Valores Possíveis

- `0` - Não espera sessions drenarem (padrão)
- `N` - Espera N segundos
- `CONFIGURED` - Usa o tempo configurado no Clusterware (RAC)

### Nota Importante da Documentação

> "This property replaces the value specified for the WAIT option of SWITCHOVER because that option has been deprecated in 26ai. Note that if the WAIT option is specified for a SWITCHOVER command, the specified wait value will override the value specified for the DrainTimeout property."

Ou seja, se você ainda usar `SWITCHOVER TO db WAIT 60`, o valor 60 tem precedência sobre DrainTimeout.

### Quando Usar

- DrainTimeout = 0: Manutenção de emergência
- DrainTimeout = 60-300: Manutenção programada
- DrainTimeout = CONFIGURED: RAC com Clusterware

## 3. FastStartFailoverLagGraceTime

Esta propriedade define um período de graça antes do Broker bloquear FSFO por causa de LAG elevado.

Segundo a documentação oficial:

> "The FastStartFailoverLagGraceTime configuration property specifies the maximum amount of time (in seconds) that can pass before the lag limit (FastStartFailoverLagLimit) is reached when the primary database requests permission to move to the lagging state."

Fonte: Oracle Database 26ai Data Guard Broker Properties

### Sintaxe

```
DGMGRL> EDIT CONFIGURATION SET PROPERTY FastStartFailoverLagGraceTime = 300;
```

### Como Funciona

1. Primary pede permissão para entrar em estado "lagging"
2. Se receber permissão ANTES do lag limit ser atingido, primary não trava
3. Se lag limit for atingido, primary trava até receber permissão OU até FastStartFailoverThreshold expirar
4. Se lag limit for atingido, FSFO não é permitido

### Valores

- `0` - Sem período de graça (padrão)
- `N` - N segundos de período de graça (mínimo: 5 segundos)

### Aplicação

Usado quando FSFO está habilitado E a configuração está em modo MaxPerformance.

## 4. FastStartFailoverLagType

Esta propriedade define qual tipo de LAG considerar para FSFO.

### Valores

- `APPLY` - Considera Apply Lag
- `TRANSPORT` - Considera Transport Lag

Esta propriedade aparece no Oracle 26ai mas não está detalhada na seção de propriedades da documentação 26ai. Aparece nas views de configuração.

## 5. ConfigurationSimpleName

Nome simplificado da configuração para identificação.

Esta propriedade é automaticamente definida pelo Broker baseada no nome da configuração criada.

## 6. ObserverPingInterval

Intervalo de ping do Observer em segundos.

Valor padrão: 0 (automático)

## 7. ObserverPingRetry

Número de tentativas de ping antes de declarar falha.

Valor padrão: 0 (automático)

## Validação no Seu Ambiente

Para confirmar se seu Oracle Database tem essas propriedades:

```
DGMGRL> SHOW CONFIGURATION VERBOSE;
```

Ou consulte a view:

```
SELECT DISTINCT property 
     FROM v$dg_broker_property 
     WHERE property IN ('PrimaryDatabaseCandidates', 
                        'DrainTimeout',
                        'FastStartFailoverLagGraceTime',
                        'FastStartFailoverLagType',
                        'ConfigurationSimpleName',
                        'ObserverPingInterval',
                        'ObserverPingRetry')
     ORDER BY property;
```

## Resumo das Propriedades Novas

| Propriedade | Função | Valor Padrão | Documentação |
| --- | --- | --- | --- |
| PrimaryDatabaseCandidates | Restringe quais DBs podem virar primary | '' (todos permitidos) | Oracle Docs 26ai |
| DrainTimeout | Tempo para drenar sessions no switchover | 0 | Oracle Docs 26ai |
| FastStartFailoverLagGraceTime | Período de graça antes de bloquear FSFO | 0 | Oracle Docs 26ai |
| FastStartFailoverLagType | Tipo de LAG a considerar (APPLY/TRANSPORT) | APPLY | View v$dg_broker_property |
| ConfigurationSimpleName | Nome simplificado da configuração | (auto) | View v$dg_broker_property |
| ObserverPingInterval | Intervalo de ping do Observer | 0 | View v$dg_broker_property |
| ObserverPingRetry | Tentativas de ping do Observer | 0 | View v$dg_broker_property |

## Referências

1. Oracle AI Database 26ai New Features - Data Guard https://docs.oracle.com/en/database/oracle/oracle-database/26/nfcoa/data_guard.html
2. Oracle Data Guard Broker Properties - 26ai https://docs.oracle.com/en/database/oracle/oracle-database/26/dgbkr/oracle-data-guard-broker-properties.html
3. Oracle Database 26ai Error Messages https://docs.oracle.com/en/error-help/db/ora-16529/
4. Oracle Database 19c Data Guard Broker Properties (comparação) https://docs.oracle.com/en/database/oracle/oracle-database/19/dgbkr/oracle-data-guard-broker-properties.html
