---
title: "Edition-Based Redefinition: upgrade de aplicação sem downtime no Oracle"
date: 2026-06-16T17:56:24-03:00
draft: false
categories: ["Oracle DBA", "Patching & Upgrades"]
tags: ["dba", "EBR", "Oracle", "Zero Downtime"]
summary: "Como usar EBR no Oracle para trocar PL/SQL, views e estrutura de tabela sem derrubar a aplicacao, com os riscos reais."
cover:
  image: "Gemini_Generated_Image_m4ww7fm4ww7fm4ww.png"
---

Todo mundo que mantém banco em produção já viveu isso: precisa subir uma versão nova de um package, mudar uma view, alterar o formato de uma coluna, e a única forma segura parece ser a janela de madrugada com a aplicação parada. O motivo é simples. Você não consegue substituir um objeto PL/SQL enquanto ele está em uso, e qualquer mudança de estrutura de tabela é vista por todas as sessões na hora.

O Oracle resolve isso desde o 11g Release 2 com o Edition-Based Redefinition, o EBR. A ideia central é deixar de tratar o schema como uma coisa só e passar a ter versões dos objetos editionable convivendo no mesmo banco, ao mesmo tempo. A sessão antiga continua rodando o código velho, a sessão nova já roda o código novo, e em nenhum momento a aplicação cai. O Oracle chama isso de hot rollover.

EBR está disponível em todas as edições do banco sem custo adicional e segue presente no 23ai e no 26ai. Não é um produto à parte, é um conjunto de três recursos que você combina conforme o tipo de mudança.

## O que o EBR realmente faz

A documentação oficial é direta sobre isso: você copia os objetos que formam o componente de banco da aplicação e redefine essa cópia isolada. Quem está usando a aplicação não enxerga nada, continua na versão antiga. Quando você confirma que a versão nova está correta, publica para todos.

O que você usa, e quanto de downtime sobra, depende de três fatores: que tipo de objeto você vai mudar, quão disponível esse objeto precisa ficar durante a mudança, e se as duas versões vão rodar ao mesmo tempo. Isso define quais dos três componentes entram em jogo.

São eles: editions, editioning views e crossedition triggers. Se você só altera objetos editionable (package, procedure, function, view, trigger, type, synonym), a edition sozinha resolve. Se você mexe na estrutura de uma tabela, entra a editioning view. Se outras sessões precisam continuar gravando nessa tabela enquanto você muda a estrutura, entram os crossedition triggers.

## O pré-requisito que ninguém conta no começo

Antes de qualquer coisa: habilitar editions para um schema é irreversível e não é uma operação online.

```text
ALTER USER app ENABLE EDITIONS;
```

Depois disso, esse usuário fica editions-enabled para sempre. Não existe `DISABLE EDITIONS`. Todo objeto editionable que ele já tem ou vier a ter passa a ser um objeto editioned. Se o schema tem objetos não editionable que dependem de objetos editionable, você precisa do `FORCE`, e isso deixa os objetos não editionable inválidos até você recompilar.

```text
ALTER USER app ENABLE EDITIONS FORCE;
```

Para conferir quem já está habilitado:

```text
SELECT username, editions_enabled
FROM   dba_users
WHERE  username = 'APP';
```

A coluna `EDITIONS_ENABLED` vem como Y quando o schema está habilitado. Esse passo de preparação é o único que pode custar uma janela curta, principalmente quando você está editionando uma aplicação que já existe e precisa colocar editioning views na frente das tabelas. A partir daí, os upgrades seguintes são sem downtime. Decida isso com calma, porque não tem volta.

## Componente 1: editions

Edition é um objeto sem schema e sem dono. Todo banco nasce com uma edition chamada `ora$base`. Você cria novas editions a partir dela:

```text
GRANT CREATE ANY EDITION, DROP ANY EDITION TO app;

CREATE EDITION rel2 AS CHILD OF ora$base;
```

Aqui tem um detalhe que pega muita gente. Uma edition só pode ter um filho. As editions formam uma cadeia linear, não uma árvore. Se você tentar criar uma segunda filha da mesma edition, toma erro. Isso significa que EBR não foi feito para dois times tocarem upgrades paralelos e independentes no mesmo banco. É uma linha do tempo: base, rel2, rel3, e assim por diante. Se você omitir o `AS CHILD OF`, a nova edition vira filha da edition folha atual.

No 26ai a sintaxe ganhou `IF NOT EXISTS` explícito, e usar `IF EXISTS` num `CREATE` retorna `ORA-11543`. No 19c esse comportamento de criar se não existe já estava presente de forma implícita na semântica do comando.

Quando você cria uma edition, ela herda na hora todos os objetos editioned da edition pai. A herança é por cópia sob demanda, o copy-on-change. Enquanto você não altera um objeto na edition nova, ele continua sendo o mesmo da pai, sem duplicar nada. No momento em que você altera, o Oracle materializa uma cópia só naquela edition, junto com os dependentes. Quem cria a edition recebe automaticamente o privilégio `USE` com `GRANT OPTION` sobre ela.

Para outros usuários enxergarem a edition, você concede explicitamente:

```text
GRANT USE ON EDITION rel2 TO app;
```

A sessão escolhe em qual edition trabalha:

```text
ALTER SESSION SET EDITION = rel2;
```

E você confirma onde está a qualquer momento:

```text
SELECT SYS_CONTEXT('userenv', 'current_edition_name') FROM dual;
```

Numa sessão recém-conectada que não setou nada, a resposta é a edition default do banco, que começa como `ORA$BASE`. É exatamente esse o pulo do gato do upgrade: você redefine tudo dentro da `rel2` enquanto a produção inteira ainda aponta para a edition antiga.

Vale saber quais tipos são editionable. Com `COMPATIBLE` 12 ou maior, são editionable: `SYNONYM`, `VIEW`, perfil de tradução SQL e todos os tipos PL/SQL (`FUNCTION`, `LIBRARY`, `PACKAGE` e `PACKAGE BODY`, `PROCEDURE`, `TRIGGER`, `TYPE` e `TYPE BODY`). Para ver na sua instância:

```text
SELECT * FROM v$editionable_types;
```

`TABLE` nunca é editionable. Tabela é sempre objeto não editioned, e é aí que entram os outros dois componentes.

## Componente 2: editioning views

Como a tabela é a mesma para todas as editions, você não pode simplesmente mudar a estrutura dela sem afetar quem está na versão antiga. A solução do EBR é colocar uma editioning view na frente da tabela. Ela projeta a tabela de um jeito diferente em cada edition, sem duplicar dado.

```text
CREATE EDITIONING VIEW app.customers_v AS
SELECT cust_id, name, status
FROM   app.customers_t;
```

A editioning view tem regras rígidas de propósito. Ela projeta uma única tabela, sem WHERE, sem join, sem ORDER BY, sem nada além de selecionar colunas. Por causa disso, o Oracle garante que o plano de execução de uma consulta contra a editioning view é idêntico ao da mesma consulta contra a tabela base. Não há penalidade de performance. Toda a noção da view some na compilação do SQL, e o plano resultante mira a tabela direto. Se você tentar incluir um WHERE, vai tomar erros que parecem desconexos, como `ORA-00933`, porque a view violou a restrição.

O padrão de uso é claro: a aplicação referencia só a editioning view, nunca a tabela. As permissões de objeto vão para a view, não para a tabela. Políticas de VPD ficam na view. Auditoria comum e FGA ficam na tabela. Quando você precisa adicionar uma coluna nova, adiciona na tabela base (que é compartilhada) e cada edition expõe sua própria editioning view com o conjunto de colunas que faz sentido para aquela versão.

A editioning view pode ser read-only ou read-write. Se você não for usar crossedition trigger para transformar dado entre formatos, faça read-only. Aí a versão antiga continua lendo, mas não escreve, o que o Oracle chama de semidisponibilidade. Aceitável para tabela de consulta. Se as duas versões precisam ler e gravar, a view é read-write e você vai precisar dos triggers.

Para listar editioning views existentes:

```text
SELECT owner, view_name
FROM   dba_editioning_views;
```

Um detalhe operacional que dói se você não souber: preparar uma aplicação legada para editioning costuma exigir renomear as tabelas base para colocar as views na frente com o nome antigo. Renomear a tabela invalida todo o PL/SQL que dependia dela, com exceção dos triggers. Por isso esse passo de adoção entra na janela inicial.

## Componente 3: crossedition triggers

Esse é o componente que separa quem leu o whitepaper de quem só leu o título.

O cenário é o seguinte: você mudou a estrutura. Antes a tabela tinha uma coluna `name`, agora você quebrou em `first_name` e `last_name`. A versão antiga grava em `name`, a nova grava nas duas colunas separadas. Enquanto o upgrade está em curso, as duas versões precisam gravar ao mesmo tempo e os dados precisam ficar coerentes nos dois formatos. Os crossedition triggers fazem essa ponte.

São dois tipos. O forward crossedition trigger pega o dado escrito no formato antigo e transforma para o formato novo. O reverse crossedition trigger faz o caminho contrário, do novo para o antigo, e é o que viabiliza o hot rollover de verdade, com as duas versões em uso simultâneo. Os triggers são editioned e só ficam visíveis na edition onde são atuais. Eles não fazem parte permanente da aplicação. Você cria para a transição e descarta quando todo mundo migrou.

O trigger trata o dado que entra durante a migração. Para os dados que já estavam na tabela, você dispara a transformação em massa aplicando o forward crossedition trigger, normalmente com `DBMS_PARALLEL_EXECUTE` para fatiar a tabela em chunks e processar em paralelo, ou via `DBMS_SQL.PARSE` com o trigger de apply. Isso converte as linhas pré-existentes para o novo formato sem travar a tabela.

Durante essa transformação aparecem as colisões. Uma linha pode já ter sido transformada pelo trigger por causa de um DML normal (o que a doc chama de mudança fortuita, serendipitous change) no exato momento em que o apply em massa tenta transformar de novo. Para distinguir por que o trigger está rodando, existe a função `APPLYING_CROSSEDITION_TRIGGER` do `DBMS_STANDARD`, que retorna TRUE quando é o apply e FALSE quando é mudança fortuita. Para lidar com chave duplicada na transformação você usa os hints `IGNORE_ROW_ON_DUPKEY_INDEX` (ignora e insere o resto) ou `CHANGE_DUPKEY_ERROR_INDEX` (troca o `ORA-00001` pelo `ORA-38911`, que você captura e trata). Isso não é detalhe acadêmico, é o que decide se sua migração de dados aguenta a carga de produção rodando junto.

## O ciclo completo de um upgrade

Juntando as peças, o fluxo de um upgrade online que muda estrutura fica assim. Considere o schema `app` já habilitado para editions e com as tabelas cobertas por editioning views.

Primeiro, cria e isola a edition nova:

```text
CREATE EDITION rel2 AS CHILD OF ora$base;
GRANT USE ON EDITION rel2 TO app;
ALTER SESSION SET EDITION = rel2;
```

Dentro da `rel2`, faz a mudança de estrutura na tabela base (que é compartilhada) e redefine a editioning view para o novo formato, além de subir o PL/SQL novo:

```text
ALTER TABLE app.customers_t ADD (first_name VARCHAR2(60), last_name VARCHAR2(60));

CREATE OR REPLACE EDITIONING VIEW app.customers_v AS
SELECT cust_id, first_name, last_name, status
FROM   app.customers_t;
```

Cria os crossedition triggers para manter os dois formatos coerentes e aplica a transformação nas linhas antigas. Testa a aplicação inteira conectando explicitamente na `rel2`, contra o dado real de produção, com a produção ainda rodando na edition antiga.

Quando estiver confiante, publica para todos trocando a edition default do banco:

```text
ALTER DATABASE DEFAULT EDITION = rel2;
```

Quando você muda a edition default, o Oracle concede o `USE ON EDITION` para PUBLIC automaticamente, então toda sessão nova passa a usar a `rel2`. As sessões antigas continuam na edition anterior até terminarem naturalmente. Esse é o hot rollover.

## Publicar com cuidado: ORA-38805

Tem uma armadilha de timing na hora de aposentar a edition antiga. Processos de background que emitem SQL, como o MMON, rodam num loop e verificam a identidade da edition default a cada iteração. Quando você troca a default, eles emitem um alter session para a nova. Isso costuma acontecer com atraso mínimo. Mas se você tentar dropar a edition antiga antes desse switch acontecer, toma `ORA-38805`. A lição prática é simples: depois de mudar a default edition, dê um respiro antes de remover a antiga, e confirme que não há mais sessões nela.

## Aposentar e remover a edition antiga

Você só pode dropar editions folha. Como a cadeia é linear, isso normalmente significa remover a ponta da cadeia, então o planejamento da hierarquia importa. Antes de dropar, faça a desmontagem dos triggers. Os forward crossedition triggers são dropados automaticamente quando a edition é dropada. Os reverse devem ser desabilitados primeiro, como parte do teardown, antes de remover a edition. Confira o que sobrou consultando as views com sufixo `_AE`, que mostram objetos de todas as editions, contra as views normais, que mostram só a edition corrente:

```text
SELECT object_name, edition_name, status
FROM   dba_objects_ae
WHERE  owner = 'APP'
ORDER  BY object_name, edition_name;
```

## EBR em CDB, RAC e Data Guard

Em multitenant, o escopo de uma edition, editioning view ou crossedition trigger é a PDB onde o recurso foi criado. Em banco non-CDB, o escopo é o banco inteiro. Isso é importante quando você tem o mesmo aplicativo em várias PDBs: cada uma tem sua própria linha de editions.

A documentação trata EBR como parte da Maximum Availability Architecture, combinável com os outros recursos de HA. Vale o aviso honesto: a doc posiciona EBR especificamente para o upgrade da camada lógica da aplicação, código e estrutura. Para upgrade do próprio binário do banco ou migração com janela mínima, o caminho que a doc indica é Oracle GoldenGate e Data Guard com switchover, que são problemas diferentes. EBR resolve o como troco meu código e meu schema sem derrubar a app, não o como faço upgrade de versão do RDBMS.

## Onde isso quebra

Os pontos abaixo estão todos documentados e merecem entrar no seu runbook antes de você prometer zero downtime para alguém.

- Habilitar editions é irreversível. Não existe desfazer. Trate como decisão de arquitetura, não como teste.
- Uma edition tem no máximo um filho. Esqueça o sonho de dois times com editions paralelas independentes. A cadeia é linear.
- O `FORCE` no `ENABLE EDITIONS` deixa objetos não editionable inválidos. Tenha o plano de recompilação pronto.
- Renomear tabelas base para adotar editioning views invalida o PL/SQL dependente, exceto triggers. Esse é o custo da adoção inicial.
- Materialized view e virtual column não enxergam objetos editioned por padrão. Se dependem de função editioned, precisam declarar uma evaluation edition com a cláusula `EVALUATE USING`, senão você toma exceção ao consultar ou atualizar.
- `ORA-38805` ao dropar a edition default anterior cedo demais. Espere o switch dos processos de background.
- Colisão de chave na transformação de dados gera `ORA-38911` quando você usa `CHANGE_DUPKEY_ERROR_INDEX`. Isso é esperado e precisa de tratamento, não é bug.

## Validação técnica

Confirmado na documentação oficial do Oracle (Database Development Guide 19c capítulo 28, SQL Language Reference de `CREATE EDITION` e `ALTER DATABASE`, EBR Technical Deep Dive 4.0 e EBR FAQ):

A sintaxe de `CREATE EDITION`, `AS CHILD OF`, `ALTER SESSION SET EDITION`, `ALTER DATABASE DEFAULT EDITION`, `GRANT USE ON EDITION` e `ALTER USER ENABLE EDITIONS`. A restrição de um único filho por edition. Que habilitar editions é irreversível e não é live. Os tipos editionable sob `COMPATIBLE` maior ou igual a 12. Que `TABLE` nunca é editionable. Que a editioning view gera o mesmo plano da tabela base. O comportamento de forward e reverse crossedition triggers e o auto-drop do forward. O escopo de PDB em CDB. E os erros `ORA-38805` e `ORA-38911` nos cenários descritos.

Depende do seu ambiente e deve ser testado antes de produção: o tempo real do switch de processos de background antes de dropar a edition antiga, o dimensionamento do `DBMS_PARALLEL_EXECUTE` para a transformação em massa de acordo com o volume da tabela, e a janela necessária para o passo único de adoção (habilitar editions e colocar editioning views na frente de tabelas legadas).

## Conclusão

EBR não é mágica e não é grátis em esforço. O custo está concentrado no passo de adoção e na disciplina de design, principalmente em sempre referenciar a editioning view e nunca a tabela. Em compensação, depois que a aplicação está editionada, trocar PL/SQL, view e até estrutura de tabela vira uma operação online de verdade, sem janela e com rollback natural enquanto as duas versões coexistem. Para ambiente crítico que não tem o luxo de parar, é a ferramenta nativa certa para o problema.
