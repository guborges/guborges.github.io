---
title: "Backup via Standby – ORA-17629"
date: 2025-12-08T07:05:00-03:00
draft: false
categories: ["Backup & Recovery", "Data Guard", "Oracle DBA"]
tags: ["Oracle"]
cover:
  image: "ORA-17629.png"
---

Durante um backup RMAN executado **pelo standby**, o processo falhou com:

```text

Oracle error from target database: 
ORA-17629: Cannot connect to the remote database server
ORA-17627: ORA-12154: TNS:could not resolve the connect identifier specified
RMAN-06822: backup not consistent because switching of archived log failed at the primary

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of backup command at 11/08/2025 20:14:19
RMAN-06822: backup not consistent because switching of archived log failed at the primary
RMAN Client Diagnostic Trace file : /u02/app/oracle/diag/clients/user_oracle/RMAN_1095788293_110/trace/ora_rman_324554_0.trc
```

O ponto central é:  
**neste tipo específico de operação**, o standby precisa **acessar o primary** para fazer o `archive log current`.  
E, nesse momento, a conexão via:

```text
rman target /
```

não foi suficiente mesmo eu já realizando outros backups de outros bancos dessa mesma forma.

Neste caso, o RMAN exigiu um connect identifier válido para alcançar o primary.

### **Solução funcional**

Conectar o RMAN usando usuário/senha + TNS:

```text
rman target sys/senha
```

Após isso, o standby conseguiu acessar o primary e o backup completou normalmente.

### **Resumo**

- Backup via standby
- ORA-17629 ocorreu ao tentar comunicar com o primary
- Para esse caso específico, OS Authentication não resolveu
- Conexão via sys@PRIMARY_TNS permitiu o acesso necessário ao primary
- Backup concluído sem erros
