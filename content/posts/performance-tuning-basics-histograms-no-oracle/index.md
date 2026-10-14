---
title: "Performance Tuning Basics : Histograms no Oracle"
date: 2025-12-01T07:00:00-03:00
draft: false
categories: ["Oracle DBA", "Performance & Tuning"]
tags: ["Oracle"]
cover:
  image: "histograms.png"
---

Quando falamos de ajuste de performance no Oracle, uma das coisas mais simples e ao mesmo tempo mais mal compreendidas é o uso de histograms. E para resumir de forma direta: histogram é um tipo especial de estatística de coluna que ajuda o otimizador a entender melhor a distribuição real dos dados. Quando os dados são muito desiguais dentro de uma coluna, o histogram pode ser exatamente o que evita um plano ruim.

Antes de entrar no contexto do Oracle, vale explicar rapidamente o que é um histogram no sentido mais simples possível.

**O que é um histogram em linguagem simples**

Um histogram nada mais é do que uma representação visual da distribuição dos dados. É como um gráfico de barras. Você divide o intervalo dos valores em várias faixas e conta quantos valores caem em cada faixa. Cada faixa vira uma barra, com altura proporcional à quantidade de valores.

Isso ajuda qualquer pessoa a entender rapidamente se os dados são concentrados em um ponto, se estão espalhados, se há picos ou se existem valores raros. No mundo de banco, isso é extremamente útil.

**Como o Oracle usa histograms**

Sem histogram, o otimizador assume que os dados são distribuídos de forma uniforme. O problema é que na maioria dos casos isso não é verdade.

Exemplo clássico:

Suponha que temos a tabela ENGINE com 10000 registros e a coluna SALE_COUNTRY. Motores vendidos para Índia, EUA e Reino Unido. Só que:

Índia: 9000  
EUA: 900  
UK: 100

Agora você roda uma query filtrando por UK.

Sem histogram, o Oracle calcula a cardinalidade assim:

10000 registros / 3 países = 3333 linhas estimadas

Só que UK tem só 100 linhas. Ou seja: a estimativa fica completamente fora da realidade. E com uma estimativa alta assim, o Oracle pode escolher um full scan desnecessário.

Com histogram, o otimizador sabe que UK é um valor raro. Isso muda a cardinalidade estimada e, com isso, o plano. O Oracle provavelmente usaria um index range scan, se existir índice.

**Tipos de histograms no Oracle**

O Oracle usa basicamente dois tipos, além de variações mais recentes:

1. Height-balanced  
   Usado quando há mais distinct values do que buckets. Ele tenta dividir a coluna em faixas com quantidades semelhantes de linhas em cada bucket.
2. Frequency  
   Mais preciso e o mais comum. Cada bucket representa um valor específico da coluna com sua frequência real. Oracle cria esse tipo quando o número de distinct values é menor ou igual ao número de buckets.

No 12c surgiram também o hybrid histogram e o top-frequency histogram, que combinam ou priorizam valores mais populares.

**Como verificar se a tabela usa histogram**

Basta consultar:

DBA_TAB_COL_STATISTICS  
ou  
USER_TAB_COL_STATISTICS

Ali você vê tipo de histogram, número de buckets e se há histogram ativo na coluna.

**Parâmetros importantes: METHOD_OPT**

Esse é o parâmetro que controla quando e como o Oracle cria histograms durante coleta de estatísticas.

O padrão desde o 10g é:

FOR ALL COLUMNS SIZE AUTO

Possíveis valores:

AUTO: Oracle decide baseado em distribuição e workload  
REPEAT: só recria histograms já existentes  
SKEWONLY: decide baseado apenas na distribuição  
1 a 254: número de buckets  
1 desativa o histogram

Para ver seu valor atual:

select dbms_stats.get_prefs('METHOD_OPT') from dual;

Para alterar para REPEAT:

```text
exec dbms_stats.set_global_prefs('METHOD_OPT', 'FOR ALL COLUMNS SIZE REPEAT');
```

Para desativar:

```text
exec dbms_stats.set_global_prefs('METHOD_OPT', 'FOR ALL COLUMNS SIZE 1');
```

**Criar e remover histograms**

O Oracle cria automaticamente quando você usa DBMS_STATS, desde que haja uso real da coluna em queries. Ele registra o uso dessas colunas em SYS.COL_USAGE$.

Para remover um histogram específico:

```text
dbms_stats.delete_column_stats('SCOTT', 'EMP', 'EMP_ID', col_stat_type=>'HISTOGRAM');
```

Para evitar que o Oracle recrie esse histogram:

```text
BEGIN<br>dbms_stats.set_table_prefs('SCOTT', 'EMP', 'METHOD_OPT'=>'FOR ALL COLUMNS SIZE AUTO, FOR COLUMNS SIZE 1 EMP_ID');<br>END;<br>/
```

**Quando criar histogram**

Crie quando:

A coluna aparece em filtros de WHERE  
A distribuição dos dados é altamente skewed

Não é só para colunas com índice. Funciona para qualquer coluna filtrada.

Não crie quando:

A distribuição é uniforme  
A coluna não é usada em WHERE  
Criar em todas as colunas só atrapalha estatísticas e parsing  
Nunca crie em chave primária  
Não crie para colunas únicas usadas apenas com igualdade

Frequency histogram é muito mais preciso que height-balanced  
Histogram ocupa espaço no dicionário e aumenta custo de manutenção  
O Oracle só cria histogram para colunas usadas em predicados  
COL_USAGE$ só é populada quando ocorre hard parse

Se você trabalha com performance no Oracle, entender histograms não é opcional. É uma das formas mais eficientes de fazer o otimizador enxergar o cenário real e gerar planos muito mais inteligentes.

![](exterminador-683x1024.png)
