---
title: "Data Guard Broker: entendendo Failover vs Switchover"
date: 2025-12-23T17:32:23-03:00
draft: false
categories: ["Data Guard", "Oracle DBA"]
tags: ["Data Guard", "Oracle"]
cover:
  image: "failoveremergency.png"
---

Uma das dúvidas mais comuns quando se trabalha com Oracle Data Guard em produção é quando usar FAILOVER e quando usar SWITCHOVER. Na teoria a diferença parece simples, mas na prática a decisão envolve impacto em dados, risco operacional e até retrabalho posterior.

Neste artigo eu explico as diferenças entre failover e switchover usando o Data Guard Broker, com foco em cenários reais de produção e não apenas no comando em si.

## Conceito geral

Switchover é uma operação planejada.  
Failover é uma operação de emergência.

Essa definição básica já ajuda, mas não resolve tudo. O que realmente importa é entender o contexto em que cada uma deve ser usada e o que acontece com o ambiente depois da execução.

## O que é Switchover

O switchover é uma troca de papéis entre o primário e o standby de forma controlada e sem perda de dados.

O banco primário é encerrado de forma ordenada, todos os redo logs pendentes são aplicados no standby e, somente depois disso, o standby é promovido a primário.

Características principais do switchover:

- Operação planejada
- Zero perda de dados
- Ambiente permanece íntegro
- O antigo primário volta automaticamente como standby
- Não exige Flashback Database

É o tipo de operação ideal para:

- Testes de DR
- Manutenções planejadas
- Troca de site por necessidade operacional
- Atualizações de infraestrutura

Com o Data Guard Broker, o switchover é simples e previsível:

```text
SWITCHOVER TO 'db_unique_name';
```

O Broker valida o ambiente, coordena a transição e garante que o apply esteja consistente antes da troca.

## O que é Failover

O failover é uma operação de recuperação de desastre. Ele é usado quando o primário não está mais acessível ou não pode ser recuperado em tempo aceitável.

Diferente do switchover, o failover pode causar perda de dados, dependendo do modo de proteção do Data Guard e do último redo aplicado no standby.

Características principais do failover:

- Operação não planejada
- Pode haver perda de dados
- O antigo primário é invalidado
- Exige ações posteriores para reintegrar o ambiente
- Pode depender de Flashback Database para reinstate

É usado em cenários como:

- Queda total do primário
- Corrupção grave
- Falha de storage ou site
- Incidentes que exigem retomada imediata do serviço

No Broker, o comando é direto:

```text
FAILOVER TO 'db_unique_name';
```

Após o failover, o antigo primário passa a aparecer como DISABLED ou NEEDS REINSTATE, o que é esperado.

## Diferença prática entre os dois

A principal diferença não está no comando, mas no impacto posterior.

No switchover:

- O ambiente já sai balanceado
- Não existe necessidade de rebuild
- O standby continua protegido automaticamente

No failover:

- O ambiente perde temporariamente a redundância
- O antigo primário precisa ser tratado
- Pode ser necessário REINSTATE ou RMAN DUPLICATE

Por isso, failover nunca deve ser tratado como alternativa “rápida” ao switchover. Ele é uma medida de emergência, não uma troca operacional.

## O papel do Data Guard Broker

O Broker é o componente que realmente torna essas operações seguras em produção.

Ele:

- Valida pré-requisitos
- Coordena a ordem das ações
- Evita comandos fora de sequência
- Mantém consistência de metadados
- Facilita validações pós-operação

Ignorar o Broker e executar failover ou switchover manualmente aumenta o risco e a complexidade, principalmente em ambientes RAC ou multitenant.

## Erros comuns em produção

Alguns erros aparecem com frequência em ambientes reais:

- Executar failover quando um switchover seria suficiente
- Não validar apply lag antes de um switchover
- Abrir o antigo primário após failover sem reinstate
- Não ter Flashback Database habilitado e depois precisar reconstruir tudo
- Tratar failover como procedimento de rotina

Entender a diferença entre as operações evita todos esses problemas.

## Qual escolher em produção

A decisão pode ser resumida de forma simples:

Use switchover quando o primário está acessível e o cenário é planejado.  
Use failover apenas quando o primário não pode ser utilizado e o foco é restaurar o serviço o mais rápido possível.

Depois de um failover, o trabalho não termina. É necessário reintegrar o ambiente, seja via REINSTATE ou rebuild, para voltar a ter proteção.

## Conclusão

Failover e switchover resolvem problemas diferentes e não devem ser tratados como operações equivalentes.

O Data Guard Broker existe exatamente para tornar essas decisões mais seguras e executáveis em produção, reduzindo erro humano e garantindo consistência.

Entender quando usar cada um é mais importante do que saber o comando. Em ambientes críticos, a escolha certa economiza tempo, evita perda de dados e reduz o risco operacional.
