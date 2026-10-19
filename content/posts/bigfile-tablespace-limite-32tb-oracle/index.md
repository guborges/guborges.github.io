---
title: "Bigfile tablespace no teto de 32 TB: o que fazer quando não dá mais para crescer"
date: 2026-10-19T07:10:00-03:00
draft: false
categories: ["Oracle DBA"]
tags: ["Oracle", "Bigfile Tablespace", "Storage", "Partitioning", "DBA"]
summary: "Bigfile com bloco de 8K para em ~32 TB e não aceita segundo datafile. Como monitorar o teto e liberar espaço movendo segmentos online."
cover:
  image: "cover.jpg"
---

Bigfile tablespace resolve muita coisa: um único datafile, `ALTER TABLESPACE ... RESIZE` direto, menos arquivos para o controlfile gerenciar. O problema aparece anos depois, quando aquele único arquivo encosta no limite físico e o time descobre que não existe "adicionar mais um datafile".

Passei por isso recentemente num ambiente Exadata com uma tablespace de dados de aplicação crescendo sem parar. Este post é o resumo do que importa: de onde vem o limite, como enxergar antes de bater nele e quais saídas existem de verdade.

## De onde vem o limite

Um bigfile tablespace tem exatamente um datafile, e esse arquivo pode ter até aproximadamente 4 bilhões (2^32) de blocos. O tamanho máximo, portanto, depende do block size da tablespace:

| Block size | Tamanho máximo do datafile |
|---|---|
| 8K | 32 TB |
| 32K | 128 TB |

Isso está na tabela de limites físicos da Oracle Database Reference. Com bloco de 8K, que é o padrão da maioria dos bancos, o teto é 32 TB e não há parâmetro que mude isso.

E o caminho tradicional de crescer não existe aqui. Tentar adicionar um datafile devolve:

```sql
SQL> ALTER TABLESPACE APP_DATA ADD DATAFILE SIZE 100G;
ALTER TABLESPACE APP_DATA ADD DATAFILE SIZE 100G
*
ERROR at line 1:
ORA-32771: cannot add file to bigfile tablespace
```

Quando o resize passa do máximo de blocos, o erro é o ORA-00740 (datafile size exceeds maximum file size).

## Monitorando a distância até o teto

O `MAXSIZE` do autoextend não é o número que interessa. O que interessa é quanto falta para 2^32 blocos. Esta query mostra isso para todas as bigfile do banco:

```sql
SELECT t.tablespace_name,
       t.block_size,
       ROUND(d.bytes    / POWER(1024,4), 2)               AS atual_tb,
       ROUND(d.maxbytes / POWER(1024,4), 2)               AS maxsize_tb,
       ROUND(POWER(2,32) * t.block_size / POWER(1024,4), 2) AS teto_tb,
       ROUND(d.blocks / POWER(2,32) * 100, 2)             AS pct_do_teto
FROM   dba_tablespaces t
JOIN   dba_data_files  d ON d.tablespace_name = t.tablespace_name
WHERE  t.bigfile = 'YES'
ORDER  BY pct_do_teto DESC;
```

`pct_do_teto` é o número para colocar em alerta. Datafile em 80% do teto já pede plano, porque a saída envolve mover dados e isso leva tempo.

Para saber quanto do arquivo está de fato ocupado (e não só alocado), cruze com `DBA_TABLESPACE_USAGE_METRICS` ou com a soma de `DBA_FREE_SPACE`. Um datafile de 31 TB com 6 TB livres internamente está numa situação bem diferente de um com 200 GB livres.

## Quem está ocupando o espaço

Antes de decidir qualquer coisa, olhe os maiores segmentos da tablespace:

```sql
SELECT owner,
       segment_name,
       partition_name,
       segment_type,
       ROUND(bytes / POWER(1024,3), 1) AS gb
FROM   dba_segments
WHERE  tablespace_name = 'APP_DATA'
ORDER  BY bytes DESC
FETCH FIRST 30 ROWS ONLY;
```

Em ambientes OLTP grandes o padrão costuma ser o mesmo: meia dúzia de tabelas particionadas por data concentram quase tudo, e as partições antigas raramente são escritas.

## As saídas que existem

Não há como "aumentar" o teto da tablespace existente. O block size de uma tablespace é definido na criação. Então toda saída real passa por tirar dados de lá ou parar de colocar dados novos lá.

### 1. Nova tablespace para os dados novos

Crie uma nova bigfile e redirecione as próximas partições para ela. Em tabela particionada, isso é um ajuste de atributo default:

```sql
CREATE BIGFILE TABLESPACE APP_DATA2
  DATAFILE SIZE 1T AUTOEXTEND ON NEXT 64G MAXSIZE UNLIMITED;

ALTER TABLE app_owner.transacoes
  MODIFY DEFAULT ATTRIBUTES TABLESPACE APP_DATA2;
```

Isso não libera nada na tablespace antiga, mas estanca o crescimento. Também é a primeira ação a tomar quando o teto está perto, porque é rápida e não move dados.

### 2. Mover partições antigas online

Com o crescimento contido, o segundo passo é aliviar a tablespace cheia movendo partições históricas:

```sql
ALTER TABLE app_owner.transacoes
  MOVE PARTITION p_2023_01 ONLINE
  TABLESPACE APP_DATA2
  UPDATE INDEXES;
```

Com `ONLINE` a partição aceita DML durante o move. Sem `UPDATE INDEXES`, a partição correspondente dos índices locais e os índices globais ficam `UNUSABLE`, o que em produção normalmente vira incidente.

Para tabela não particionada, o `ALTER TABLE ... MOVE ONLINE` existe desde o 12.2:

```sql
ALTER TABLE app_owner.config_hist
  MOVE ONLINE TABLESPACE APP_DATA2
  UPDATE INDEXES;
```

O espaço liberado volta como free space dentro do datafile antigo e pode ser reutilizado pelos segmentos que continuam lá. O arquivo em si não encolhe sozinho.

### 3. Compressão no mesmo move

O `MOVE PARTITION` aceita cláusula de compressão, então dá para mover e comprimir na mesma operação. Em Exadata, partições históricas são candidatas naturais a Hybrid Columnar Compression:

```sql
ALTER TABLE app_owner.transacoes
  MOVE PARTITION p_2022_01 ONLINE
  TABLESPACE APP_ARCH
  COMPRESS FOR ARCHIVE LOW
  UPDATE INDEXES;
```

Confira licenciamento antes: compressão avançada para OLTP depende de Advanced Compression, e HCC é recurso de storage Exadata (ou outros storages Oracle suportados).

### 4. Block size de 32K na tablespace nova

Se a tabela realmente precisa morar numa única tablespace acima de 32 TB, a alternativa é criar a nova tablespace com bloco de 32K, que leva o teto a 128 TB. Isso exige configurar `DB_32K_CACHE_SIZE` na instância, porque blocos não padrão usam um buffer cache separado. É uma decisão de arquitetura, não um quick fix: memória dedicada, comportamento diferente de I/O e mais um cache para dimensionar.

## Riscos para levar em conta

Estes pontos não são incidentes que reproduzi para o post, são consequências diretas de como as operações acima funcionam e que valem planejamento:

- Move online precisa de espaço na tablespace de destino para o segmento inteiro antes de liberar o de origem.
- Todo move gera redo proporcional ao volume movido. Com Data Guard, isso vira transporte e apply no standby, então acompanhe o lag.
- `UPDATE INDEXES` em índices globais grandes deixa o move mais lento. A própria documentação de particionamento comenta que atualizar índices compensa mais quando a partição é pequena em relação à tabela.
- Um detalhe reportado em fórum: `AUTOEXTEND ... MAXSIZE 32T` em bigfile de 8K falha com ORA-00740 em 12.2 e 19c, porque o máximo real fica um pouco abaixo de 32 TB. `MAXSIZE UNLIMITED` funciona.

## Validação técnica

Confirmado na documentação oficial:

- Limite de ~2^32 blocos por datafile de bigfile, 32 TB com 8K e 128 TB com 32K (Oracle Database Reference, Physical Database Limits).
- `ORA-32771` ao adicionar um segundo arquivo em bigfile (Database Error Messages).
- `ORA-00740` quando o tamanho pedido excede o máximo de blocos (Database Error Messages).
- `RESIZE` em `ALTER TABLESPACE` válido apenas para bigfile (SQL Language Reference).
- `ALTER TABLE ... MOVE ONLINE` com DML durante o move a partir do 12.2 (Administrator's Guide, Changes in This Release).
- Comportamento de índices `UNUSABLE` no `MOVE PARTITION` sem `UPDATE INDEXES`, e `MOVE PARTITION ... COMPRESS` (VLDB and Partitioning Guide).

Prática de mercado:

- Redirecionar partições novas antes de mover as antigas.
- Alertar por percentual do teto de blocos, e não pelo `MAXSIZE`.

Varia por ambiente:

- Tempo de move e volume de redo dependem do tamanho dos segmentos, da carga e do storage.
- Disponibilidade de HCC e Advanced Compression depende de plataforma e licença.

## Conclusão

Bigfile continua sendo uma boa escolha, mas o teto é fixo e a única forma de lidar com ele é tirar dados de dentro. Coloque o `pct_do_teto` no monitoramento hoje, redirecione as partições novas quando ele passar de 70 a 80%, e deixe o move das partições históricas para janelas planejadas. Esperar o ORA-00740 aparecer transforma uma tarefa de rotina em um incidente.

## Referências

- [Oracle Database Reference: Physical Database Limits](https://docs.oracle.com/en/database/oracle/oracle-database/18/refrn/physical-database-limits.html)
- [ORA-32771](https://docs.oracle.com/error-help/db/ora-32771/)
- [ORA-00740](https://docs.oracle.com/en/error-help/db/oci-00740/)
- [Administrator's Guide: Changes in This Release (MOVE ONLINE)](https://docs.oracle.com/en/database/oracle/oracle-database/18/admin/release-changes.html)
- [VLDB and Partitioning Guide: Moving Partitions](https://docs.oracle.com/database/121/VLDBG/GUID-3DB0B6A6-FD33-4355-8BAC-7C5B4BE4C4E0.htm)
- [VLDB and Partitioning Guide: Partitioning and Table Compression](https://docs.oracle.com/database/122/VLDBG/partition-table-compression.htm)
- [Toad World Forum: ORA-00740 com MAXSIZE 32T](https://forums.toadworld.com/t/bigfile-tablespace-datafile-resize-ora-00740/49517)
