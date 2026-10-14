---
title: "Atualizações com JOIN Direto no Oracle 26Ai"
date: 2026-02-23T07:00:00-03:00
draft: false
categories: ["Oracle 26Ai", "SQL & PL/SQL"]
tags: ["26ai", "Oracle"]
cover:
  image: "joins.png"
---

O Oracle AI Database 26Ai apresenta diversas melhorias em relação às versões anteriores. Entre elas está a inclusão do *direct join* em comandos UPDATE e DELETE. Em versões anteriores, como a 19c, atualizar ou excluir registros com base em múltiplas tabelas demandava o uso de MERGE ou subconsultas correlacionadas. Agora, no 26Ai, a linguagem SQL permite declarar tabelas de origem diretamente na cláusula FROM dos comandos UPDATE/DELETE, simplificando a sintaxe e tornando o código mais legível. A documentação oficial de recursos do Oracle destaca que, a partir da versão 26Ai, é possível "juntar a tabela de destino em comandos UPDATE e DELETE a outras tabelas usando a cláusula FROM" e que isso facilita a alteração e exclusão de dados. Para DBAs que ainda trabalham com o 19c, a atualização para 26Ai abre essa e outras possibilidades interessantes, como colunas BOOLEAN nativas e a cláusula FILTER em agregações.

## Comparação rápida 19c vs 26Ai

| Recurso | Como se fazia no 19c | Como fazer no 26Ai |
| --- | --- | --- |
| **Update/Delete com join** | `MERGE` ou sub‑query correlacionada | `UPDATE ... SET ... FROM ... WHERE ...` |
| **Filtro em agregações** | `SUM(CASE WHEN condição THEN x ELSE 0 END)` | `SUM(x) FILTER (WHERE condição)` |
| **Coluna BOOLEAN** | `CHAR(1)` ou `NUMBER(1)` | `BOOLEAN` nativo |

## Demonstração prática: atualizando departamentos com join direto

Este exemplo cria duas tabelas simples: **DEPARTAMENTOS** e **FUNCIONARIOS** e usa a nova sintaxe para atualizar uma coluna da tabela de departamentos com a soma dos salários de seus funcionários.

### 1. Criar tabelas e inserir dados

```
-- tabela alvo
CREATE TABLE departamentos (
  deptno         NUMBER PRIMARY KEY,
  total_salarios NUMBER DEFAULT 0
);

-- tabela fonte
CREATE TABLE funcionarios (
  id      NUMBER PRIMARY KEY,
  deptno  NUMBER,
  salario NUMBER
);

-- dados de exemplo
INSERT INTO departamentos (deptno) VALUES (10);
INSERT INTO departamentos (deptno) VALUES (20);

INSERT INTO funcionarios (id, deptno, salario) VALUES (1, 10, 1000);
INSERT INTO funcionarios (id, deptno, salario) VALUES (2, 10, 2000);
INSERT INTO funcionarios (id, deptno, salario) VALUES (3, 20, 1500);
INSERT INTO funcionarios (id, deptno, salario) VALUES (4, 20, 2500);

COMMIT;
```

### 2. Verificar o estado inicial

```
SELECT deptno, total_salarios FROM departamentos;
```

Resultado esperado:

```
    DEPTNO TOTAL_SALARIOS
---------- --------------
        10              0
        20              0
```

### 3. Atualizar com join direto

No 26Ai o join direto permite relacionar a tabela alvo com a tabela fonte dentro da cláusula FROM. A sintaxe se parece com um SELECT, mas aplica a atualização ao SET:

```
UPDATE departamentos d
   SET d.total_salarios = f.total
  FROM (
        SELECT deptno, SUM(salario) AS total
          FROM funcionarios
         GROUP BY deptno
       ) f
 WHERE d.deptno = f.deptno;
```

Nesse exemplo, a sub‑query f soma os salários por departamento e disponibiliza esses valores para o UPDATE por meio da cláusula FROM. A cláusula WHERE relaciona as linhas da tabela departamentos à sub‑query e garante que cada departamento receba apenas uma soma. Esta sintaxe reduz a verbosidade e evita o uso de MERGE.

Apenas para exemplo, no 19c a querie ficaria da seguinte forma:

```
UPDATE departamentos d
   SET d.total_salarios = (
         SELECT SUM(f.salario)
           FROM funcionarios f
          WHERE f.deptno = d.deptno
       )
 WHERE EXISTS (
         SELECT 1
           FROM funcionarios f
          WHERE f.deptno = d.deptno
       );
```

### 4. Conferir o resultado

```
SELECT deptno, total_salarios FROM departamentos;
```

Resultado esperado:

```
    DEPTNO TOTAL_SALARIOS
---------- --------------
        10           3000
        20           4000
```

Os valores 3000 e 4000 correspondem à soma dos salários em cada departamento. Observe que a cláusula FROM pode referenciar outras tabelas ou sub‑queries e que o critério de junção deve ser chave‑primária ou único na tabela alvo; caso contrário, o banco retornará erro de multiplicidade.

## Como funciona por baixo dos panos

O otimizador do Oracle materializa a sub‑query ou tabela de origem referenciada na cláusula FROM e aplica o join para cada linha da tabela de destino. Assim, o SET manipula apenas as colunas da tabela alvo, enquanto as colunas da(s) tabela(s) de origem servem de referência. A documentação oficial observa que, na versão 26Ai, é possível “juntar a tabela de destino em comandos UPDATE e DELETE a outras tabelas usando a cláusula FROM”, o que torna a manutenção de dados mais simples.

## Migração e compatibilidade

No Oracle 19c, a sintaxe UPDATE ... FROM ou DELETE ... FROM não é suportada. Para realizar atualizações multi‑tabela, era preciso recorrer a MERGE:

```
MERGE INTO departamentos d
USING (
       SELECT deptno, SUM(salario) total
         FROM funcionarios
        GROUP BY deptno
      ) f
ON (d.deptno = f.deptno)
WHEN MATCHED THEN
  UPDATE SET d.total_salarios = f.total;
```

Ou escrever sub‑queries correlacionadas no SET. Com o 26Ai, recomenda‑se migrar esses scripts para a nova forma, que é mais fácil de ler e evita armadilhas de *key preservation*. Lembre‑se de que o upgrade de 19c para 26Ai é suportado diretamente pela Oracle.

## Dicas

- **Chave única** – garanta que a junção entre a tabela de destino e a de origem seja única; caso contrário, você receberá o erro `ORA‑30926` (múltiplos matches).
- **Aliases** – sempre use aliases (`d`, `f`, etc.) para diferenciar colunas de tabelas diferentes e evitar ambiguidades.
- **Permissões** – o usuário deve ter privilégios de leitura nas tabelas de origem e de escrita na tabela de destino.
- **Teste antes de rodar** – execute a consulta de junção (SELECT ... FROM ... WHERE ...) isoladamente para conferir o número de linhas afetadas antes de fazer o `UPDATE`.
- **Compatibilidade** – scripts que usam a nova sintaxe não funcionarão em 19c; planeje a migração com cuidado.

O *direct join* em comandos UPDATE e DELETE é um recurso poderoso que chega com o Oracle 26Ai. Ele torna atualizações multi‑tabela mais claras e reduz a necessidade de MERGE ou de sub‑queries complicadas. Para DBAs que atualizarem para o 26Ai, vale revisitar scripts antigos e simplificá‑los com essa nova sintaxe. Além do join direto, o 26Ai traz outras melhorias, como colunas BOOLEAN nativas e agregações com FILTER, que tornam o SQL ainda mais expressivo.
