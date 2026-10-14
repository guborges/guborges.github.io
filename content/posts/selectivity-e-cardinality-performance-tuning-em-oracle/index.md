---
title: "Selectivity e Cardinality: Performance Tuning em Oracle"
date: 2025-10-07T17:58:04-03:00
draft: false
categories: ["Oracle DBA", "Performance & Tuning"]
tags: ["Oracle"]
cover:
  image: "ChatGPT-Image-Oct-7-2025-05_49_31-PM.png"
---

Antes de pensar em reescrever queries, criar índices ou ajustar parâmetros, é fundamental compreender como o otimizador do Oracle toma decisões. Grande parte dos problemas de performance não está no código em si, mas na forma como o banco interpreta os dados e escolhe o plano de execução.

Dois conceitos essenciais nesse processo são **Selectivity** e **Cardinality**. Eles servem como base para a estimativa de custos, a escolha dos caminhos de acesso e a eficiência geral de qualquer consulta. Sem dominá-los, qualquer tentativa de tuning será apenas tentativa e erro.

### SELECTIVITY: Medindo o quanto sua consulta é precisa

SELECTIVITY: o quanto sua consulta é específica

**Selectivity** mede a fração de linhas filtradas por uma condição. Em termos simples, ela mostra **quanto do total de dados realmente é relevante para sua query**.

- O valor sempre fica entre **0 e 1**
- Valores **próximos de 0** indicam filtros **altamente seletivos** (*bom sinal*)
- Valores **próximos de 1** indicam filtros **pouco seletivos** (*pouco eficientes*)

**Exemplo prático:**  
Considere uma tabela `CLIENTES` com 1.000.000 de registros.

1. Uma consulta genérica:

```text
SELECT * FROM CLIENTES;
```

- Todas as linhas são retornadas -> Selectivity = 1.0 (ou 100%)
- Isso significa que a operação não filtrou nada.

2. Uma consulta mais específica:

```text
SELECT * FROM CLIENTES WHERE PAIS = 'BRASIL';
```

- Digamos que apenas 50.000 registros satisfaçam essa condição
- Selectivity = 50.000 / 1.000.000 = **0.05 (5%)**
- Alta seletividade, pois apenas 5% das linhas foram relevantes

Quanto mais seletivo o filtro, maior a chance de o otimizador **usar um índice com eficiência**. Por isso, colunas com muitos valores distintos (alta cardinalidade de dados) são excelentes candidatas para índices.

### CARDINALITY: a quantidade real de dados retornados

Se a selectivity indica a proporção filtrada, a **cardinality** mostra o número absoluto de linhas que passam por determinada etapa da execução.

A relação entre elas é direta:

```text
cardinality = selectivity × número total de linhas
```

**Exemplo prático:**  
Imagine uma tabela `PEDIDOS` com 200.000 registros.

1. Você executa a seguinte query:

```text
SELECT * FROM PEDIDOS WHERE STATUS = 'ENTREGUE';
```

- Se 20.000 registros têm esse status:
  - Selectivity = 20.000 / 200.000 = **0.10 (10%)**
  - Cardinality = 200.000 × 0.10 = **20.000**

2. Agora um filtro mais restritivo:

```text
SELECT * FROM PEDIDOS WHERE STATUS = 'ENTREGUE' AND VALOR > 5000;
```

- Digamos que apenas 1.200 registros atendam às duas condições:
  - Selectivity = 1.200 / 200.000 = **0.006 (0,6%)**
  - Cardinality = 200.000 × 0.006 = **1.200**

Perceba como a cardinalidade diminui à medida que os filtros se tornam mais seletivos. Isso influencia diretamente a forma como o otimizador escolhe operações como *index range scan*, *full table scan* ou *nested loops*.

### Quando a cardinalidade é mal estimada

Um dos maiores problemas de performance surge quando o otimizador **estima a cardinalidade de forma incorreta**. Isso geralmente ocorre por falta de estatísticas atualizadas ou por filtros não representativos.

Por exemplo: se o Oracle espera que sua consulta retorne 100 linhas e ela traz 10.000, pode acabar escolhendo um plano de execução ineficiente, com *joins* custosos ou leitura desnecessária de blocos.

Manter as estatísticas atualizadas e revisar filtros complexos é essencial para evitar esses cenários.

### Por que isso importa tanto para DBAs

Selectivity e cardinality são a base de todas as decisões do otimizador.  
Se você entende como elas funcionam, passa a ter **controle real** sobre o plano de execução e consegue prever o impacto de cada filtro ou índice.

Ao otimizar uma consulta:

- Verifique a **selectivity** das colunas usadas em filtros
- Analise a **cardinality** esperada em cada etapa do plano
- Atualize as estatísticas regularmente para que as estimativas sejam precisas

Em resumo,

**Selectivity**: proporção de linhas filtradas. Quanto menor, melhor.

**Cardinality**: número absoluto de linhas retornadas. É a base da estimativa de custo.

Dominar esses dois conceitos é o primeiro passo para otimizações eficazes e planos de execução previsíveis.

![](cropped-logo2.png)
