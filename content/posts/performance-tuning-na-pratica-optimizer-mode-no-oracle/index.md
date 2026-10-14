---
title: "Performance Tuning na prática - OPTIMIZER_MODE no Oracle"
date: 2025-12-23T17:31:51-03:00
draft: false
categories: ["Oracle DBA", "Performance & Tuning"]
tags: ["Oracle"]
cover:
  image: "optmizer.png"
---

Quando uma query é enviada ao Oracle, não é o SQL em si que define como os dados serão buscados. Quem decide isso é o Cost Based Optimizer. Ele avalia estatísticas, volume de dados, seletividade, índices disponíveis e custo estimado de cada operação para montar o plano de execução.

O parâmetro **OPTIMIZER_MODE** influencia diretamente esse processo. Ele define qual é o objetivo principal do otimizador ao gerar os planos. Dependendo do tipo de sistema, uma configuração pode ser adequada ou totalmente contraproducente.

Não existe modo certo ou errado de forma absoluta. Existe o modo correto para cada tipo de carga.

## O que o OPTIMIZER_MODE realmente controla

De forma simples, esse parâmetro diz ao Oracle se ele deve priorizar:

- velocidade para entregar os primeiros registros
- eficiência para processar toda a consulta
- ou um equilíbrio baseado em quantidade de linhas esperadas

Essa escolha impacta diretamente se o Oracle tende a usar índices, full table scans, paralelismo e até join methods diferentes.

## Modos mais utilizados na prática

Hoje, na prática, três modos fazem sentido em ambientes Oracle modernos.

### FIRST_ROWS

Nesse modo, o foco do otimizador é reduzir o tempo até que os primeiros registros apareçam para o usuário.

O Oracle tende a escolher caminhos que retornem dados rapidamente, mesmo que o custo total da query seja maior. A consequência mais comum é a preferência por acessos via índice.

Esse comportamento costuma funcionar bem em sistemas transacionais, telas de consulta e aplicações onde o usuário não consome grandes volumes de dados, mas espera resposta imediata.

### FIRST_ROWS(n)

Aqui o comportamento é parecido com o FIRST_ROWS, mas com uma informação extra importante.

Você informa explicitamente quantas linhas pretende consumir. Isso ajuda o otimizador a ser mais assertivo na escolha do plano, pois ele passa a trabalhar com um alvo concreto.

Esse modo é muito comum em consultas com paginação, relatórios limitados ou APIs que retornam apenas um subconjunto de registros.

### ALL_ROWS

No ALL_ROWS, o objetivo muda completamente.

O Oracle passa a buscar o plano mais eficiente para processar a consulta inteira, minimizando consumo total de recursos. A prioridade não é mostrar dados rapidamente, mas terminar o processamento da forma mais barata possível.

Por isso, nesse modo, é comum ver full table scans, inclusive paralelos, sendo escolhidos mesmo quando existem índices.

Esse é o modo padrão do Oracle e faz bastante sentido para cargas analíticas, relatórios pesados e ambientes de Data Warehouse.

## Como identificar o modo ativo no banco

Para saber qual é o comportamento padrão configurado no banco, basta consultar o parâmetro:

```text
SHOW PARAMETER optimizer_mode
```

Ou, se preferir via dicionário:

```text
SELECT value
FROM v$parameter
WHERE name = 'optimizer_mode';
```

## Ajustando o comportamento apenas para uma query

Nem sempre é desejável alterar o comportamento do banco inteiro. Em muitos casos, o ajuste deve ser pontual.

O Oracle permite influenciar o otimizador diretamente na query usando hints.

Para priorizar eficiência total da execução:

```text
SELECT /*+ ALL_ROWS */ col1, col2
FROM tabela_exemplo;
```

Para priorizar resposta rápida inicial:

```text
SELECT /*+ FIRST_ROWS */ col1, col2
FROM tabela_exemplo;
```

Quando o número de linhas é conhecido e relevante:

```text
SELECT /*+ FIRST_ROWS(50) */ col1, col2
FROM tabela_exemplo;
```

Esse tipo de ajuste é muito mais seguro do que mudar parâmetros globais, principalmente em ambientes compartilhados.

## Alterando o modo em nível de sessão

Se o objetivo for testar ou aplicar o comportamento apenas para a conexão atual, é possível alterar o modo da sessão:

```text
ALTER SESSION SET optimizer_mode = ALL_ROWS;
```

Isso vale apenas enquanto a sessão estiver ativa e não afeta outros usuários.

## Alterando o modo padrão do banco

Quando existe uma decisão arquitetural clara, o modo padrão do banco pode ser alterado:

```text
ALTER SYSTEM SET optimizer_mode = FIRST_ROWS SCOPE=SPFILE;
```

Esse tipo de mudança exige cuidado, testes e restart da instância. Em ambientes OLTP, por exemplo, pode fazer sentido. Em ambientes analíticos, normalmente não.

## Aplicando um modo específico para um usuário

Em alguns cenários, apenas um usuário ou aplicação precisa de um comportamento diferente. Isso pode ser resolvido com um trigger de logon.

Exemplo:

```text
CREATE OR REPLACE TRIGGER trg_set_optimizer
AFTER LOGON ON DATABASE
BEGIN
  IF SYS_CONTEXT('USERENV','SESSION_USER') = 'APP_RELATORIO' THEN
    EXECUTE IMMEDIATE 'ALTER SESSION SET optimizer_mode = ALL_ROWS';
  END IF;
END;
/
```

Assim, o ajuste é automático, transparente para a aplicação e restrito apenas ao usuário desejado.

Nenhum modo de otimização compensa estatísticas ruins.

O Cost Based Optimizer depende completamente de estatísticas atualizadas. Se elas estiverem incorretas ou desatualizadas, o plano gerado pode ser ruim independentemente do OPTIMIZER_MODE configurado.

Antes de discutir modo de otimização, sempre valide o estado das estatísticas dos objetos envolvidos.
