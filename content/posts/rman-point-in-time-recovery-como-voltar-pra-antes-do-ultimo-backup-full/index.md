---
title: "RMAN Point-in-Time Recovery: como voltar pra antes do último backup full"
date: 2026-04-20T06:46:14-03:00
draft: false
categories: ["Backup & Recovery", "Oracle DBA"]
tags: ["Oracle", "RMAN"]
cover:
  image: "devoltaparafuturo.png"
---

Seu último full backup é de sábado de manhã. Alguém detectou um problema na sexta-feira à tarde e você precisa voltar o banco pra antes disso. Dá pra fazer? Sim. O RMAN usa o full **anterior**, não o de sábado. E não, o RMAN não reverte transações. Ele faz roll forward a partir de um backup mais antigo.

## O cenário

Ambiente Oracle 19c standalone, modo ARCHIVELOG, retention policy configurada. A rotina é:

- Full backup (level 0) todo sábado de manhã
- Incrementais diários (level 1) durante a semana
- Archivelogs copiados para o FRA a cada commit e backup de archivelog de hora em hora

Na segunda-feira, descobrem que uma atualização em massa na sexta-feira às 14h corrompeu dados de negócio. O cliente pede pra voltar o banco pra sexta-feira às 13h30, antes da atualização.

O full mais recente é o de sábado de manhã, ou seja, **depois** do target time. Esse backup não serve.

## O problema

Duas confusões aparecem nesse momento e é nelas que o DBA tropeça.

**Confusão 1: "O último full é o ponto de partida do restore."**

Não é. Com `SET UNTIL` anterior ao full mais recente, o RMAN vai ignorar o full de sábado e procurar o backup mais recente cujo checkpoint SCN seja anterior ao target. Isso é documentado:

> By default, RESTORE chooses the most recent backup set or file copy, that is, the file copy or backup set that needs the least media recovery.

Quando você amarra o `SET UNTIL`, o "mais recente" passa a ser o mais recente **dentro da janela válida para aquele target**. No nosso caso, o full da semana anterior.

**Confusão 2: "O RMAN reverte as transações até chegar no target."**

Não reverte. O RMAN não conhece transações. Ele opera em nível de bloco e de datafile. O que ele faz é:

1. Restaura datafiles de um backup **anterior** ao target
2. Aplica redo (archivelogs e online logs) **para frente**, sequência a sequência, até o SCN do target
3. Para no target e abre com `RESETLOGS`

Reverter transação é coisa de rollback no nível da sessão (via UNDO). Flashback Database também opera em nível físico, mas usando flashback logs, não redo. DBPITR é roll forward de um estado anterior até um ponto alvo. A direção é sempre para frente no tempo, a partir de um ponto no passado que é anterior ao target.

## Diagnóstico antes de começar

Antes de apertar qualquer botão, três checks são obrigatórios.

**Check 1: o backup full anterior ainda existe?**

```text
-- Lista os fulls disponíveis no controlfile/catalog
RMAN> LIST BACKUP OF DATABASE SUMMARY;
```

Se a retention policy é `REDUNDANCY 1` ou foi rodado `DELETE OBSOLETE` depois do full de sábado, o backup da semana anterior pode estar marcado como obsoleto ou já ter sido deletado fisicamente. Se foi deletado, acabou. Sem flashback database ativo, não há volta.

Em retention por `RECOVERY WINDOW OF N DAYS`, o Oracle mantém o backup necessário para qualquer PITR dentro da janela. Essa é a configuração segura para esse cenário.

```text
RMAN> SHOW RETENTION POLICY;
```

**Check 2: os archivelogs entre o full antigo e o target existem?**

```text
-- Lista archivelogs pelo range de SCN ou tempo
RMAN> LIST ARCHIVELOG FROM TIME "to_date('SEX_02_DEZ_00:00:00','DY_DD_MON_HH24:MI:SS')" UNTIL TIME "to_date('SEX_09_DEZ_14:00:00','DY_DD_MON_HH24:MI:SS')";
```

Qualquer gap aqui mata a recuperação. O redo tem que estar contínuo desde o checkpoint SCN do backup restaurado até o SCN do target. Um gap no meio e você para no último archivelog válido antes do gap, não no target desejado.

**Check 3: PREVIEW do restore.**

Antes de tocar no banco, previsualize o que o RMAN vai fazer:

```text
RMAN> RUN {
  SET UNTIL TIME "to_date('09-DEC-2024 13:30:00','DD-MON-YYYY HH24:MI:SS')";
    RESTORE DATABASE PREVIEW;
}
```

O `PREVIEW` mostra quais backup sets e archivelogs o RMAN pretende usar, sem restaurar nada. Se a lista mostrar o full de sábado (posterior ao target), algo está errado. Provavelmente o `SET UNTIL` não pegou ou a ordem dos comandos está errada. Se a lista mostrar o full antigo mais os archivelogs/incrementais necessários, você está no caminho certo.

## Solução

Fluxo completo do DBPITR para o cenário sexta às 13h30.

### Passo 1: shutdown e startup em mount

```text
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
```

O DBPITR exige o banco montado. Os datafiles vão ser sobrescritos e o banco não pode estar aberto.

### Passo 2: ajustar NLS_DATE_FORMAT na sessão

```text
$ export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
$ rman target /
```

Isso evita ambiguidade no `SET UNTIL TIME`. Sem isso, o RMAN pode interpretar a string com um formato diferente do esperado e o target SCN fica errado.

### Passo 3: verificar incarnation

Se houve algum RESETLOGS recente (por outro PITR anterior, por exemplo), o target pode estar em uma incarnation parent:

```text
RMAN> LIST INCARNATION;
```

Se o target time está **antes** de uma `RESETLOGS` existente, é preciso fazer reset para a incarnation correta:

```text
RMAN> RESET DATABASE TO INCARNATION <numero>;
```

No nosso cenário simples, sem RESETLOGS prévio, esse passo não é necessário. Mas não assuma, verifique.

### Passo 4: executar o restore e recover

```text
RMAN> RUN {
  SET UNTIL TIME "to_date('09-DEC-2024 13:30:00','DD-MON-YYYY HH24:MI:SS')";
  RESTORE DATABASE;
  RECOVER DATABASE;
}
```

O `SET UNTIL` vem **antes** do RESTORE. Essa ordem importa. Se você colocar o `SET UNTIL` depois do RESTORE, o RMAN pode restaurar o full mais recente (sábado) porque não sabe ainda qual é o target.

O que acontece por baixo:

1. O `SET UNTIL` fixa o target SCN da sessão RMAN
2. O RESTORE procura no catálogo/controlfile o backup full mais recente com checkpoint SCN menor que o target SCN e restaura os datafiles desse backup
3. Se houver incremental level 1 entre o full restaurado e o target, o RMAN aplica o incremental primeiro, bloco a bloco, muito mais rápido que aplicar redo
4. Depois, aplica archivelogs para a frente, sequência por sequência, até atingir o target SCN
5. Para

### Passo 5: abrir com RESETLOGS

```text
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

Ou em SQL*Plus. Após incomplete recovery, o RESETLOGS é obrigatório. Isso cria uma nova incarnation do banco. O stream de redo recomeça do sequence 1 na nova incarnation.

## Validação

Três validações rápidas pra confirmar que o PITR funcionou.

**Validar o tempo/SCN de abertura:**

```text
SQL> SELECT RESETLOGS_TIME, RESETLOGS_CHANGE# FROM V$DATABASE;
```

O RESETLOGS_TIME deve bater com o momento em que você abriu o banco. O RESETLOGS_CHANGE# é o primeiro SCN da nova incarnation.

**Validar a nova incarnation:**

```text
RMAN> LIST INCARNATION OF DATABASE;
```

A CURRENT deve ser a que acabou de ser criada. A anterior aparece como PARENT.

**Validar os dados de negócio:**

Antes do `RESETLOGS`, se você quiser confirmar que o PITR chegou no ponto certo:

```text
SQL> ALTER DATABASE OPEN READ ONLY;
```

Abre em read-only, permite query nas tabelas de negócio pra confirmar que a atualização corrompida não está lá. Se estiver ok, fecha, faz o `OPEN RESETLOGS`. Se não estiver, você precisa fazer novo PITR com target mais cedo. E pra isso terá que recuperar o backup novamente, porque os datafiles já foram modificados pelo recover.

## Troubleshooting

### Problema: RMAN-06054, media recovery requesting unknown log

**Sintoma:** durante o `RECOVER DATABASE`, RMAN pede um archivelog que não está no catalog nem no disco.

**Causa:** gap na sequência de archivelogs. O redo entre o backup restaurado e o target não está completo.

**Solução:** catalogar archivelogs que estejam em disco mas fora do catálogo:

```text
RMAN> CATALOG START WITH '/caminho/dos/archivelogs/';
```

Se o archivelog foi perdido de verdade (backup de archivelog corrompido, tape ilegível), o máximo que você consegue é recuperar até o último archivelog válido antes do gap. Nesse caso, ajuste o `SET UNTIL` para um SCN imediatamente anterior ao gap.

### Problema: ORA-01152, file not restored from a sufficiently old backup

**Sintoma:** ao tentar OPEN RESETLOGS, aparece ORA-01152 mencionando um datafile.

**Causa:** os datafiles não foram todos restaurados de um backup anterior ao target. Isso acontece quando o `SET UNTIL` não foi aplicado corretamente no RESTORE, ou quando um datafile não tem backup antigo disponível.

**Solução:** verifique que o `SET UNTIL` precede o RESTORE no bloco RUN. Refaça o restore, se necessário forçando com `RESTORE DATABASE FROM TAG '<tag_do_backup_antigo>'`.

### Problema: retention já deletou o full anterior

**Sintoma:** `LIST BACKUP OF DATABASE` não mostra nenhum full anterior ao target. O `RESTORE ... PREVIEW` falha com mensagem de "no backup found".

**Causa:** retention policy agressiva (REDUNDANCY 1) mais `DELETE OBSOLETE` deletaram o backup necessário.

**Solução:** não tem solução via RMAN se o backup foi fisicamente removido. Alternativas de recuperação:

- Flashback Database, se estiver ativo e o flashback window cobre o target
- Flashback Query no nível de tabela, se o undo_retention permitir
- RMAN Table Point-in-Time Recovery (RECOVER TABLE), se o problema é só em algumas tabelas e existe backup + archivelogs suficientes
- Recuperar o backup antigo de um storage secundário (tape archive, snapshot de storage, cópia offsite)

Depois do incidente, revisar a retention policy. `RECOVERY WINDOW OF N DAYS` com N alinhado ao RTO/RPO do negócio é o padrão seguro.

### Problema: incarnation errada

**Sintoma:** `RMAN-20208` ou mensagem dizendo que o target time é anterior ao RESETLOGS atual.

**Causa:** o target está em uma incarnation antiga (parent ou orphan).

**Solução:**

```text
RMAN> LIST INCARNATION;
RMAN> RESET DATABASE TO INCARNATION &lt;numero_da_parent>;
```

Depois rodar o restore/recover normalmente.

## Erros comuns

| Erro | Causa provável | Ação |
| --- | --- | --- |
| RMAN-06054 | Archivelog ausente entre backup e target | Catalogar archivelogs faltantes ou ajustar target para antes do gap |
| ORA-01152 | Datafile restaurado de backup posterior ao target | Verificar ordem do SET UNTIL e refazer restore |
| RMAN-20208 | Target em incarnation antiga | RESET DATABASE TO INCARNATION antes do restore |
| ORA-01194 | RESETLOGS antes de completar o recover | Completar o RECOVER DATABASE antes de abrir |
| RMAN-06025 | Archivelog necessário marcado como expired | CROSSCHECK ARCHIVELOG ALL; depois recatalogar se o arquivo existe em disco |

## Boas práticas

- Retention policy baseada em `RECOVERY WINDOW` alinhado ao RPO do negócio, não em `REDUNDANCY`
- `CONFIGURE CONTROLFILE AUTOBACKUP ON` sempre. Sem isso, recuperação de metadata do banco vira pesadelo
- Backup de archivelog frequente, com múltiplos destinos (pelo menos dois LOG_ARCHIVE_DEST_N)
- `RESTORE DATABASE PREVIEW` antes de qualquer PITR real, sempre
- Testar o procedimento de PITR em ambiente de homologação periodicamente. Um runbook que nunca foi executado é ficção
- Documentar o tempo médio de restore pra datasets reais, não estimativas teóricas
- Quando possível, manter Flashback Database ativo como primeira linha de defesa para incomplete recovery, é muito mais rápido que DBPITR

## Riscos e avisos

> **ATENÇÃO:** o `OPEN RESETLOGS` cria uma nova incarnation e invalida a possibilidade de recuperar além desse ponto na incarnation anterior sem configuração especial. Antes de executar, confirme que o banco está no estado desejado usando `OPEN READ ONLY` se possível.

> **ATENÇÃO:** o DBPITR afeta o banco inteiro. Todas as transações entre o target e o momento atual são perdidas. Se o problema está isolado em uma tabela, considere Flashback Query, Flashback Table ou RMAN Table PITR antes de recorrer ao DBPITR.

> **ATENÇÃO:** se o retention policy está configurado como `REDUNDANCY` em vez de `RECOVERY WINDOW`, o backup necessário pode não existir mais. Verifique antes de prometer o PITR ao negócio.

## Validação técnica

**Confirmado na documentação oficial:**

- DBPITR restaura datafiles de um backup anterior ao target time e aplica redo/incrementais forward até o ponto alvo (Oracle Database Backup and Recovery User's Guide 19c, capítulo 18).
- `RESTORE` escolhe por padrão o backup mais recente que minimize o media recovery necessário; com `SET UNTIL`, passa a ser o mais recente dentro da janela válida (Oracle Database Backup and Recovery Reference 19c, RESTORE).
- Media recovery aplica redo no sentido forward. Nas palavras da doc: "Media recovery selectively applies the changes recorded in the online and archived redo logs to the restored datafile to roll it forward" (Oracle Recovery Principles).
- `OPEN RESETLOGS` é obrigatório após incomplete recovery e inicia nova incarnation (Oracle Database Backup and Recovery User's Guide 19c).
- `NOREDO` é aplicável apenas a bancos em NOARCHIVELOG ou quando redo não pode ser aplicado (Oracle Database Backup and Recovery User's Guide 19c, cap. 20).

**Prática de mercado:**

- Retention policy por `RECOVERY WINDOW OF N DAYS` é consenso para ambientes com compromisso de RPO fixo. `REDUNDANCY` sobrevive melhor em ambientes com esquema de backup muito simples, mas complica PITR de janela ampla.
- Múltiplos destinos de archivelog (pelo menos dois) é padrão em ambientes de produção com criticidade alta.
- `RESTORE DATABASE PREVIEW` antes de qualquer PITR real é prática de mercado. A doc menciona o comando, mas não o enforça.
- Flashback Database como primeira linha de defesa para incomplete recovery é amplamente adotado onde o overhead de flashback log é aceitável.

**Pode variar por ambiente:**

- O tempo de DBPITR depende do volume de redo a aplicar e da velocidade do storage. Um banco de 2TB com 24 horas de redo pra aplicar pode levar horas.
- Uso de incrementais level 1 entre o full e o target acelera muito o recover. O RMAN prefere o incremental ao redo. Se sua rotina não faz incrementais diárias, todo o gap é aplicado via archivelog.
- Em RAC, todos os nós precisam estar desligados durante o DBPITR e o procedimento é iniciado por um único nó.
- Em ambientes com Data Guard, o PITR no primary cria divergência de incarnation e exige tratamento específico no standby (ver MOS Doc 1902323.1 para cenários de PITR em ambiente Data Guard).
- Em CDBs, o PITR pode ser feito no CDB inteiro ou apenas em PDBs específicos, com implicações diferentes sobre undo e auxiliary destination (cap. 18 da doc 19c).

RMAN PITR pra antes do último full funciona exatamente como pra qualquer outro target: o `SET UNTIL` manda o RMAN escolher o backup mais recente anterior ao alvo, e o recover aplica redo forward até o ponto desejado. O RMAN não reverte transações. Ele restaura um estado antigo e reconstrói até o target. A armadilha não está no comando, está no pré-requisito: o backup anterior tem que existir e os archivelogs têm que ser contínuos até o target. Retention policy e rotina de archivelog decidem se esse PITR é possível ou se vai virar postmortem.
