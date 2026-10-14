---
title: "Data Guard - O básico para Iniciantes"
date: 2025-10-04T10:55:51-03:00
draft: false
categories: ["Data Guard", "Oracle DBA"]
tags: ["Oracle"]
cover:
  image: "ChatGPT-Image-Oct-6-2025-10_37_57-AM.png"
---

Olá pessoal!

No meu primeiro artigo, comentei sobre ter conseguido meu tão sonhado OCP (2 anos de estudos intensos) e agora, quero comentar sobre um dos assuntos nos quais tenho me dedicado durante esse último ano em que trabalho como DBA Oracle Junior: o tão amado, querido e apaixonante Data Guard.

Confesso que, no início, ao ler o nome "Oracle Data Guard", achei que fosse algo SUPER ULTRA complexo, algo que eu nunca conseguiria entender nem mesmo o básico, rs. Mas aqui estou eu, TCHARAM!!! Com o básico em minhas mãos, e tentarei compartilhar isso com vocês!

**Vamos lá!**

**O que é o Data Guard?**

Em poucas palavras, da forma como eu entendi e que soa menos "aterrorizante", o Oracle Data Guard é um replicador que envia informações do banco de dados principal (chamado de Primário) para o banco de dados Secundário (chamado de Standby), garantindo alta confiabilidade e disponibilidade.

Simples, não é?

---

**Por que devemos utiliza-lo?**

Ótima pergunta! Vou responder com outra pergunta... Se por acaso, no seu servidor de banco de dados, ocorresse algum problema, como um terremoto, corrupção do servidor inteiro, ou até mesmo um temido "sudo rm -rf /* 2>/dev/null", o que você faria com todas as informações do banco? Pois é... com o Data Guard, você não precisa se preocupar, pois todas as informações inseridas no banco principal são replicadas no secundário. Isso parece ser uma ótima razão para usá-lo.

Lembrando que estou tentando compartilhar o pouco que sei com pessoas iniciantes, assim como eu. O Data Guard oferece várias possibilidades, como a criação de diversas bancos de dados standby, mas vamos mantê-lo simples para facilitar a compreensão.

---

**Como conectar no Data Guard?**

Estou partindo do ponto, onde somos iniciantes e vamos manipular um DG que já está instalado e configurado. Em outro post, eu posso até ensinar vocês, uma maneiro bem fácil de instalar o DG ( Data Guard ).

Utilizaremos o DG Broker, pense nele como uma aplicativo igual o aplicativo do RMAN, para você realizar todas as configurações necessárias, mas para o DG.

Para conectar, você vai digitar o seguinte comando em seu SQL*Plus

```sql
dgmgrl /
```

E aqui estamos, conectados em nosso DG.

Perfeito, e agora?

Vamos emitir nosso primeiro comando, assim, poderemos nos orientar e puxar algumas informações referente o nosso DG.

Digite esse comando em seu SQL*Plus

```sql
show configuration;
```

Esse é o resultado esperado.

DG Configuration

Aqui podemos verificar algumas informações importantes. Lembra que eu comentei sobre a replicação entre o banco primário e o secundário?

Pois bem, este comando nos mostrará como a nossa configuração do DG está nesse aspecto.

Na imagem, é possível observar que temos o banco principal (com o nome de DG) e o banco standby (com o nome de dg_stby).

Basicamente, o banco DG está operando e todas as alterações feitas nele são enviadas para o dg_stby. Fácil de entender, não é?

Aqui, estou mostrando a vocês a prova disso. Reparem que tenho o mesmo banco, conforme mostrado na imagem 1 (servidor primário) e imagem 2 (standby), com **db_unique_name** diferente (um dos requisitos do DG, que abordarei mais no futuro). Realizo um insert e um commit.

Notem que no quadrado verde, apenas realizo selects e os valores são atualizados.

DG em prática

Muito legal entender esse conceito do que é o Data Guard, não é mesmo? Eu fico extremamente empolgado sempre que aprendo ou entendo algo novo.

Existem mais alguns comandos, vou dar apenas uma resumida;

Comando que verifica o status de um banco de dados

```sql
show database '<nome do banco';
```
database

Fornece informações completas de todas as propriedades de um banco de dados

```sql
show database verbose '<nome do banco>';
```
verbose

---

**Informações Básicas sobre Data Guard**

Bom, agora que nós entendemos o que é o DG e compreendemos qual é a sua função, vamos abordar algumas informações sobre ele. Quais são os tipos de Data Guard? Como ele transporta informações? Quais são os métodos de proteção do DG?

**Tipos de Data Guard**

O DG possuí 3 tipos, sendo eles:

- **Physical Standby**

Ele é a cópia EXATA do banco primario ( banco principal ). Ele aplica alterações via REDO APPLY, onde o REDO é transmitido para o banco secundário, armazenado no redo log e aplicado no banco secundário(standby).

- **Logical Standby**Ele possui as mesmas informações do banco principal, porém, possúi estrutura fisica diferente, ele é sincronizado via SQL APPLY. o que seria isso? Ele transforma REDO em instruções SQL e executado no banco secundário.
- **Snapshot Standby** Em rápidas palavras, ele é uma cópia do banco principal, PORÉM, ele pode ser alterado, acredito que essa é a forma mais simples e direta para explicar.

---

**Como o Data Guard transporta as informações?**

Vou tentar explicar da forma mais simples possível, que é o intuito deste artigo: quando um usuário realiza um commit no banco principal, o Oracle registra essas informações em um arquivo redo. O Data Guard, em seguida, pega essas informações e as envia para o banco standby, onde a mágica acontece.

---

**Métodos de proteção do Data Guard**

O Data Guard possuí 3 métodos de proteção de dados, sendo eles, de forma resumida:

**Máxima Proteção**

Garante o famoso zero de perda de dados, caso ocorra alguma falha na rede ou no banco de dados secundário. Se ocorrer alguma falha que resulte na perda de sincronia com o banco de dados secundário, o banco de dados primário é desligado.

**Máxima Disponibilidade**

Garante o famoso zero data loss, sem comprometer a disponibilidade do banco de dados primário. O banco de dados primário não é desligado enquanto houver pelo menos um standby redo log sincronizado; se ocorrer a perda de sincronismo, o banco de dados primário continua operacional até que tudo retorne à sua normalidade.

**Máxima Performance**

O modo padrão oferece o mais alto nível de proteção possível sem afetar o desempenho do banco de dados. Toda transação que é confirmada é prontamente registrada no redo log local.

---

**Querys**

Pessoal, para concluir este artigo básico sobre o Data Guard (ou melhor, um guia direto ao ponto com o que você precisa saber para não ficar completamente perdido, como costumo ficar rs), vou compartilhar algumas consultas para vocês utilizarem no dia a dia.

Para começar, vamos verificar as configurações do nosso Data Guard, que basicamente englobam tudo que tentei explicar neste artigo.

```sql
SELECT DATABASE_ROLE, DB_UNIQUE_NAME INSTANCE, OPEN_MODE, PROTECTION_MODE, PROTECTION_LEVEL, SWITCHOVER_STATUS FROM V$DATABASE;
```
Lembra que eu comentei sobre como o Data Guard transporta as informações? Pois então, em meus laboratórios, já enfrentei algumas falhas em que o Redo Apply estava desabilitado, o que significa que o serviço de transporte estava desativado. Onde eu verifico isso? Aqui, verificando se o processo MRPO está habilitado no banco standby.

--- Verificando processos rodando

--- Usado para verificar o processo MRPO

```sql
SELECT PROCESS, STATUS, THREAD#, SEQUENCE#, BLOCK#, BLOCKS FROM V$MANAGED_STANDBY;
```
MRPO

E se por acaso, tiver um LAG de transação nas nossas configurações, como verificamos?

Bom, essa consulta permite que você acompanhe o intervalo de tempo entre a geração de registros (redos) no banco de dados primário e a aplicação desses registros no banco de dados secundário. Quanto menor o valor de atraso, mais frequentemente os "redos" estão sendo aplicados e mais próxima está a replicação em tempo real.

```sql
SELECT name, value, datum_time, time_computed FROM V$DATAGUARD_STATS WHERE name like 'apply lag';
```
Cansei! Não quero mais ter o redo apply sendo executado, como eu paro e inicio?

-- start - Iniciar o redo apply ( no banco standby, ok?)

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```

-- stop - Parar o redo apply ( no banco standby, ok?)

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
```

E agora, a cereja do bolo, para mim, o comando mais legal, aquele que eu mais utilizo sempre (realmente todos os dias, pois faço isso constantemente rs), é ler as mensagens de erro quando algo dá errado no meu banco de dados.

- Verificar as mensagens de falha no DATAGUARD

```sql
SELECT MESSAGE FROM V$DATAGUARD_STATUS;
```
---

Bom, encerrando mais um artigo para iniciantes como eu, tentando compartilhar o pouco que sei com vocês. Eu entendo perfeitamente como é se sentir completamente perdido ao lidar com uma análise, especialmente com alguma tecnologia à qual não tenho a menor ideia de como conectar. Isso pode ser assustador, mas também é incrivelmente gratificante.

Eu garanto a vocês que o 1% do conhecimento que prometo compartilhar aqui se multiplica por 200%, porque tenho medo de passar informações incorretas (o que pode acontecer). Eu me esforço ao máximo para pesquisar em várias fontes e revisar várias vezes. Para mim, isso é extremamente satisfatório, e eu aprendo muito no processo. Recomendo a todos começarem a fazer o mesmo; prometo que não se arrependerão, e seu aprendizado se acelerará 100 vezes mais!

Aqui está uma dica: tente ensinar a alguém o que está estudando. No processo, você descobrirá várias lacunas em seu próprio conhecimento e aprenderá muito mais. Essa foi uma das técnicas que utilizei para sair de ser um completo iniciante em Oracle e alcançar meu tão sonhado título de OCP 19.

No próximo artigo, que estou planejando fazer, pretendo abordar o ASM. Recentemente, participei de uma crise relacionada ao ASM no trabalho, e foi uma experiência incrível. Participar, entrar em uma conferência, observar um especialista trabalhando e analisando a situação, entendendo o como, o onde, o quando e o porquê, foi uma verdadeira aula que não encontramos em nenhum curso por aí! Vou tentar criar este artigo, cobrindo os conceitos básicos do ASM e compartilhando minha experiência com a análise impressionante realizada pelo especialista. A forma como ele resolveu a situação me deixou realmente impressionado, e será incrível compartilhar isso.

Encerramos por aqui, pessoal. Até a próxima! Perdoa qualquer erro, na próxima vou estar melhor!

![](image-683x1024.png)
