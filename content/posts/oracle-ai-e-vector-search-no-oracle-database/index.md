---
title: "Oracle AI e Vector Search no Oracle Database"
date: 2026-03-02T07:00:00-03:00
draft: false
categories: ["Oracle 26Ai", "Oracle DBA"]
tags: ["26ai", "Oracle", "Script Oracle"]
cover:
  image: "vectorsearch.png"
---

Tive o prazer de semana passada participar do evento em São Paulo, ACE Connect Brazil, e assistir uma palestra do ACE Director Mario Barduchi sobre Oracle AI e Vector Search.

Eu vinha deixando esse tema meio de lado. Achava interessante, mas não tinha parado para entender tecnicamente o que realmente mudou no banco.

Depois da palestra, fui olhar com mais calma e comecei a ver com outros olhos. Não como buzzword de IA, mas como recurso novo de banco de dados mesmo.

O ponto que mais me chamou atenção foi a possibilidade de trabalhar com modelos de IA dentro do próprio Oracle. Importar modelos no formato ONNX, executar inferência no banco e gerar embeddings sem depender obrigatoriamente de serviço externo muda bastante a arquitetura.

Quando você percebe que isso já está integrado ao 23ai e 26ai, deixa de ser tendência distante e passa a ser algo concreto para laboratório.

## O que é Vector Search

Vector Search é busca por similaridade semântica.

Um texto é convertido em vetor dentro de um espaço multidimensional.  
A consulta também é convertida em vetor.  
O banco calcula a distância entre esses vetores.

Quanto menor a distância, maior a similaridade de significado.

Não estamos falando de LIKE ou igualdade de string.  
Estamos falando de proximidade semântica.

## Tipo de dado VECTOR

O Oracle passou a suportar o tipo de dado VECTOR de forma nativa.

E como o Mario comentou na palestra, não é para sair criando coluna VECTOR sem critério. Embedding ocupa espaço. Cada vetor tem dimensionalidade definida, geralmente centenas ou milhares de posições. Isso cresce rápido.

Se você começar a gerar embedding para tudo, sem estratégia, o impacto em storage vai aparecer. Estamos falando de estruturas numéricas de alta dimensão armazenadas linha a linha. Em bases grandes isso pode representar gigabytes adicionais com facilidade.

VECTOR resolve um tipo específico de problema: similaridade semântica.  
Ele não substitui modelagem relacional, nem índice tradicional, nem tuning clássico.

Exemplo simples:

```text
CREATE TABLE artigos (
    id         NUMBER PRIMARY KEY,
    titulo     VARCHAR2(200),
    conteudo   CLOB,
    embedding  VECTOR(768, FLOAT32)
);
```

Você continua tendo estrutura relacional normal, com PK, CLOB e constraints.

O vetor entra como complemento para permitir busca semântica, não como substituição de tudo.

Isso simplifica bastante quando comparado a arquiteturas que usam banco separado só para vetores.

## Importação de modelos no banco

Uma parte relevante é o suporte a modelos ONNX.

Você pode importar um modelo de embedding para dentro do próprio Oracle e executar inferência diretamente no banco.

O fluxo deixa de depender obrigatoriamente de API externa.

Texto entra  
Modelo roda  
Embedding é gerado  
Vetor é armazenado

Tudo dentro do ambiente do banco.

Em cenário corporativo isso impacta latência, governança, segurança e compliance, principalmente quando não é aceitável enviar dado sensível para serviço externo.

### Exemplo prático de importação de modelo ONNX

Supondo que o arquivo do modelo esteja no servidor do banco, por exemplo:

`/u01/models/embedding_model.onnx`

Primeiro criamos um diretório lógico no Oracle apontando para esse caminho:

```text
CREATE OR REPLACE DIRECTORY ONNX_DIR AS '/u01/models';
GRANT READ, WRITE ON DIRECTORY ONNX_DIR TO gustavo;
```

Agora importamos o modelo para dentro do banco:

```text

BEGIN
  DBMS_VECTOR.LOAD_ONNX_MODEL(
    'ONNX_DIR',                       -- DIRECTORY
    'embedding_model.onnx',           -- FILE_NAME
    'EMBEDDING_MODEL',                -- MODEL_NAME
    JSON('{"function":"embedding"}'), -- METADATA (obrigatório)
    NULL                              -- EXTERNAL_DATA_FILE_NAME
  );
END;
/
```

Após a importação, é possível validar se o modelo foi registrado:

```text
SELECT model_name,
       mining_function,
       algorithm,
       model_size
FROM user_mining_models;
```

Com o modelo carregado, ele passa a ser um objeto gerenciado dentro do banco.

A geração de embedding pode ser feita diretamente via SQL, por exemplo:

```text
SELECT VECTOR_EMBEDDING(
         EMBEDDING_MODEL
         USING 'Texto para gerar embedding' AS INPUT
       ) AS emb
FROM dual;
```

O resultado dessa chamada pode ser armazenado diretamente na coluna do tipo `VECTOR`.

Pontos importantes:

- O modelo precisa estar em formato ONNX suportado pelo Oracle
- A dimensão do embedding precisa ser compatível com a coluna VECTOR
- O modelo passa a consumir CPU e memória durante inferência
- Versionamento de modelo deve ser tratado como parte da governança do ambiente

## Consulta por similaridade

Uma consulta típica pode ser assim:

```text
SELECT id,
       titulo,
       VECTOR_DISTANCE(
         embedding,
         VECTOR_EMBEDDING(EMBEDDING_MODEL USING 'texto da consulta' AS INPUT),
         COSINE
       ) AS distancia
FROM artigos
ORDER BY distancia
FETCH FIRST 5 ROWS ONLY;
```

Você passa o vetor da pergunta e recebe os registros semanticamente mais próximos.

Esse tipo de consulta é base para busca inteligente, recuperação de documentação, base de conhecimento interna e soluções com RAG.

## Indexação vetorial

Quando o volume de embeddings cresce, índice vetorial passa a ser essencial.

O Oracle implementa técnicas de nearest neighbor aproximado para acelerar a busca.

Você ajusta parâmetros de precisão e desempenho conforme necessidade do ambiente.

Isso precisa ser tratado como qualquer outro índice crítico em produção, considerando plano de execução, estatística e consumo de recurso.

## Exemplo prático simples

Para sair da teoria, imagine uma base de conhecimento interna com artigos técnicos.

```text
CREATE TABLE artigos (
    id         NUMBER PRIMARY KEY,
    titulo     VARCHAR2(200),
    conteudo   CLOB,
    embedding  VECTOR(384, FLOAT32)
);
```

Inserindo um registro:

```text
INSERT INTO artigos (id, titulo, conteudo, embedding)
VALUES (
    1,
    'Erro ORA-01102 em ambiente RAC',
    'Erro ORA-01102 ocorre quando o banco tenta montar em modo exclusivo...',
    VECTOR_EMBEDDING(
        EMBEDDING_MODEL
        USING 'Erro ORA-01102 ocorre quando o banco tenta montar em modo exclusivo...' AS INPUT
    )
);
```

Após gerar o embedding com o modelo importado:

```text
UPDATE artigos
SET embedding = VECTOR_EMBEDDING(
                  EMBEDDING_MODEL
                  USING conteudo AS INPUT
                )
WHERE id = 1;
```

Criando índice vetorial:

```text
CREATE VECTOR INDEX idx_artigos_embedding
ON artigos (embedding)
ORGANIZATION NEIGHBOR PARTITIONS
DISTANCE COSINE;
```

Consultando por similaridade:

```text
SELECT id,
       titulo,
       VECTOR_DISTANCE(
         embedding,
         VECTOR_EMBEDDING(EMBEDDING_MODEL USING 'sua pergunta aqui' AS INPUT),
         COSINE
       ) AS distancia
FROM artigos
ORDER BY distancia
FETCH FIRST 3 ROWS ONLY;
```

Mesmo que a pergunta não seja idêntica ao texto armazenado, o banco retorna o conteúdo mais próximo semanticamente.
