---
title: "RMAN - O básico para Iniciantes"
date: 2025-10-04T11:37:50-03:00
draft: false
categories: ["Backup & Recovery", "Oracle DBA"]
tags: ["Oracle"]
cover:
  image: "ChatGPT-Image-Oct-6-2025-10_46_31-AM.png"
---

**Olá pessoal!**

Neste meu artigo, gostaria de falar um pouco sobre o RMAN, algo que tenho dedicado bastante tempo aos estudos (juntamente com o DataGuard). Como mencionei, estou aprendendo e quero começar a escrever esses artigos, tanto para registrar meu conhecimento quanto para fixá-lo em minha mente e também compartilhar.

---

**O que é o RMAN?**

O RMAN, resumidamente, é um aplicativo fornecido pela Oracle que auxilia na realização de backups e recuperações do banco de dados. Podemos efetuar o backup de diversas maneiras, seja um backup completo, parcial ou em um ponto específico no tempo.

---

**Por que devemos utiliza-lo?**

Essa é uma pergunta bastante simples! Se, por acaso, um usuário, ou até mesmo você, realizar um "UPDATE" sem uma cláusula "WHERE", como você poderá recuperar os dados? EXATAMENTE! Com o backup. E se você perder um arquivo de dados (datafile)? Seu sistema operacional (SO) corromper? E se? E se? E se? Para evitar todas essas preocupações, realizamos backups com o RMAN, garantindo assim que possamos dormir em paz e não perder cabelos (embora tenham me alertado sobre essa segunda parte tarde demais).

---

**Como conectar no RMAN?**

Como estou iniciando minha jornada de escrita, também estou aprendendo. Vou compartilhar informações básicas sobre todas as configurações que podemos realizar, certo?

Para fazer a conexão, digitaremos a seguinte opção no SQL*Plus:

```sql
rman target /
```

P.S.: Podemos adicionar mais informações a essa conexão, mas vamos deixar isso para os usuários mais avançados.

Pronto, estamos conectados ao nosso aplicativo de backups.

Perfeito, mas e agora?

Ótima pergunta, mestre. Dentro do aplicativo, podemos executar algumas consultas sobre nossos backups, realizar os próprios backups e até mesmo executar seleções (selects). Vamos fazer alguns testes?

---

**Buscando informações do RMAN**

Vou mostrar a vocês algumas possibilidades que podemos realizar com o RMAN.

Vamos começar pelo mais importante, que é fazer o nosso BACKUP. Com este comando (o mais simples possível), realizaremos um backup completo do nosso banco de dados.

```sql
Backup database;
```

Essa é a saída que devemos esperar do nosso comando.

Backup database;

Com esse comando, podemos fazer a listagem de todos os backups catalogados em nossa base.

```sql
LIST BACKUP;
```

Essa é a saída que devemos esperar do nosso comando. Aqui, um ponto importante: temos a listagem de TODOS os backups realizados na nossa base de dados. Nessa tela, conseguimos verificar informações importantes, como:

- **Backup está comprimido?**
- **Data do Backup**
- **Tipo do backup**
- **Data do Backup (BCK)**
- **Local onde está armazenado o backup**

List Backup

Um zoom nas informações que mencionei acima: cada bloco desses representa uma "piece" do backup. Isso costumava me confundir muito no começo, quando eu não entendia absolutamente nada do que esse comando mostrava, rs.

todos os backups

```sql
LIST BACKUP SUMMARY;
```

Esse comando traz exatamente as mesmas informações do anterior, porém de forma resumida, sem tantos detalhes, como tamanho, localização, etc.

backup summary

```sql
LIST BACKUP OF DATAFILE 1;
```

Surgiu a dúvida: "Será que eu fiz backup do arquivo tal?" Aqui está o comando certo para você; com ele, encontraremos informações específicas do datafile 1 (por exemplo).

lista de datafile especifico

```sql
LIST ARCHIVELOG ALL; 
```

E aqui, é usado no RMAN (Recovery Manager) para listar todos os arquivos de log de backup (archive logs) disponíveis no banco de dados.

archives

---

**Catalogando "peças" do backup.**

Outra parte muito importante e que costumava me confundir MUITO é o catálogo de backup. "Nossa, mas o que é isso?" De forma resumida (da maneira como fiz para que eu conseguisse entender): Nós solicitamos ao Oracle o seguinte:

> **"Oracle, por gentileza (educação, né?), vá até esse caminho aqui e traga para mim todas as 'peças' de backup que você desconhece, pois eu posso ter movido as 'peças' via SO, ou estou recuperando meu banco em outra base, existem diversos motivos."**

Perfeito, e como fazemos isso?

```sql
CATALOG START WITH 'diretorio' NOPROMPT;
```

Simples, não é mesmo? Vamos apenas fornecer o diretório para o ORACLE e pedir a ele para localizar todas as "peças" que desejamos. Como mencionei, podem haver N motivos para realizar essa operação, sendo a movimentação dos backups apenas um deles.

Em um exemplo bem simples, o comando ficaria assim. Aqui, realizei o backup de um datafile em um caminho específico e, em seguida, o movi usando o sistema operacional com o comando "mv" para um novo diretório. Após isso, informei ao RMAN o novo caminho deste datafile.

---

**SQL salva vidas**

Pessoal, para finalizar com chave de ouro, gostaria de compartilhar uma consulta SQL que utilizo em minhas rotinas como DBA. Essa consulta verifica todos os backups realizados na base, se foram concluídos com sucesso e seu status atual (rodando ou finalizado). É MUITO útil para uma primeira análise.

```sql
col STATUS format a10
col hrs format 999.99

select SESSION_KEY, INPUT_TYPE, STATUS,
to_char(START_TIME,'mm/dd/yy hh24:mi') start_time,
to_char(END_TIME,'mm/dd/yy hh24:mi') end_time,
elapsed_seconds/3600 hrs from V$RMAN_BACKUP_JOB_DETAILS
order by session_key; 
```

A saída desta consulta é basicamente a seguinte:

Reparem que todas as informações necessárias para uma primeira análise estão presentes. Acreditem, esta consulta salva vidas e cabelos!

---

Bom pessoal, é isso que gostaria de compartilhar com vocês, essas pesquisas bastante básicas sobre o RMAN. Ainda estou aprendendo a escrever textos tão longos, rs, mas prometo trazer informações mais detalhadas sobre o assunto que vou abordar no futuro, e também (tentar) apresentar textos em inglês :).

Trouxe algumas informações tanto sobre a conexão quanto sobre como realizar análises básicas no RMAN. Essa era uma área em que eu tinha MUITA dificuldade no começo, mas agora tenho uma compreensão mais sólida de como me conectar e verificar os backups que temos na base.
