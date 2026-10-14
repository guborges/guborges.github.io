---
title: "Data Guard Broker: SWITCHOVER na prática"
date: 2025-12-23T17:32:51-03:00
draft: false
categories: ["Data Guard", "Oracle DBA"]
tags: ["Data Guard", "Oracle"]
cover:
  image: "switchover2.png"
---

Depois de falar sobre conceitos (failover vs switchover) o próximo passo natural é o cenário mais tranquilo:**switchover**.

Aqui não tem desastre, não tem queda de host e não tem improviso. Switchover é troca planejada de papéis, feita com o ambiente saudável, sincronizado e sob controle.

Neste post eu mostro como faço um switchover usando Data Guard Broker, usando o mesmo lab que utilizo nos treinamentos do meu mentor. Esse lab facilita muito o estudo, porque o ambiente já vem preparado para repetir o processo quantas vezes for necessário, sem risco e sem dependência de infraestrutura real.

Os bancos do cenário são:

- AUTO_U_BR
- AUTO_U_US

## Quando faz sentido usar switchover

Switchover não é alternativa a failover. É outra coisa.

Eu uso switchover quando:

- o primário está acessível
- o Data Guard está sincronizado
- não existe pressão de incidente
- a troca faz parte de teste, manutenção ou exercício de DR

Se o primário caiu, não é switchover. Se tem perda de dados envolvida, não é switchover. Aqui a premissa é ambiente saudável.

## Antes de qualquer comando, eu valido o estado do ambiente

Switchover bom começa antes do comando.

Antes de trocar os papéis, eu quero ter certeza de três coisas:

- os bancos estão sincronizados
- não existe gap de redo
- o Broker está enxergando a configuração corretamente

A partir de qualquer nó, conecto no DGMGRL e começo pelo básico:

```text
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW CONFIGURATION LAG;
```

Esses três comandos já dizem muito. Se o lag não estiver zerado ou convergindo, eu não avanço.

Depois eu valido os dois lados individualmente:

```text
VALIDATE DATABASE 'AUTO_U_BR';
VALIDATE DATABASE 'AUTO_U_US';
```

Se o Broker disser que não está pronto para switchover, eu paro e resolvo o motivo. Switchover não aceita “vamos tentar pra ver se vai”.

## Rede e comunicação também entram no checklist

Outra coisa que eu sempre valido é se os bancos conseguem se enxergar corretamente pela rede, do jeito que o Broker espera.

```text
VALIDATE NETWORK CONFIGURATION FOR ALL;
```

Esse comando evita surpresas depois, principalmente em ambientes RAC ou com múltiplos listeners.

## Garantindo que o standby está pronto

Antes de trocar os papéis, eu confiro se o apply está ativo no standby. Se por algum motivo ele estiver pausado, eu não sigo.

```text
SHOW DATABASE VERBOSE 'AUTO_U_US';
```

Se precisar, eu ativo explicitamente pelo Broker:

```text
EDIT DATABASE 'AUTO_U_US' SET STATE='APPLY-ON';
```

A regra aqui é simples: **sem apply consistente, sem switchover**.

## Executando o switchover

Com tudo validado, aí sim vem o comando mais simples de todo o processo.

```text
SWITCHOVER TO 'AUTO_U_US';
```

O que eu gosto no Broker é justamente isso: ele coordena tudo sozinho.

Ele desmonta o primário atual, promove o standby, converte o antigo primário em standby e sobe o apply automaticamente, sem precisar ficar alternando comandos manuais.

## Conferindo o resultado com calma

Depois do switchover, eu não saio rodando comandos em sequência frenética. Eu dou alguns segundos e volto a validar.

```text
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW CONFIGURATION LAG;
```

Depois olho cada banco:

```text
SHOW DATABASE VERBOSE 'AUTO_U_US';
SHOW DATABASE VERBOSE 'AUTO_U_BR';
```

Aqui eu quero ver:

- AUTO_U_US como primary
- AUTO_U_BR como physical standby
- apply rodando normalmente

## Validando no SQL também

Além do Broker, eu sempre confiro direto no banco:

```text
SELECT name, open_mode, database_role FROM v$database;
```

Isso elimina qualquer dúvida sobre o papel real do banco após a troca.

## Teste rápido de conectividade

Switchover pode estar tecnicamente perfeito e ainda assim falhar do ponto de vista da aplicação.

No lab eu faço um smoke test simples para garantir que o serviço está respondendo no novo primário:

```text
sqlplus sh/oracle@srvora3:1521/AUTOUS_SVC
```

Em produção, aqui entra validação de service, SCAN, balanceamento e o que a aplicação realmente usa.

## E se eu quiser voltar para o estado original

Como isso é um cenário planejado, nada impede de fazer o caminho inverso.

```text
SWITCHOVER TO 'AUTO_U_BR';
```

Depois disso, repito os mesmos checks de configuração e lag para garantir que tudo voltou ao estado inicial.

## Uma observação importante sobre licenciamento

Switchover não exige Active Data Guard.

Se o standby estiver em MOUNT aplicando redo, você está dentro do Data Guard básico, que já vem na Enterprise Edition.  
Active Data Guard só entra em cena quando você abre o standby com READ ONLY WITH APPLY.

## O que vem depois na sequência

Com isso, fecha-se o cenário de operação planejada.

Nos próximos posts a sequência fica clara:

- failover: quando tudo dá errado
- reinstate: como recuperar o ambiente depois do desastre

Cada um resolve um problema diferente, e entender isso evita muita dor de cabeça em produção.
