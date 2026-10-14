---
title: "Backup Oracle direto para Bucket OCI usando RMAN"
date: 2025-11-24T07:00:00-03:00
draft: false
categories: ["Backup & Recovery", "Cloud", "Scripts & Automação"]
tags: ["Cloud", "Oracle", "Script Oracle"]
cover:
  image: "backup_oci.png"
---

Há algum tempo venho padronizando uma arquitetura de backup direto para bucket OCI, integrada ao RMAN, sem passar por filesystem local. É uma abordagem simples, limpa e que resolve vários problemas que normalmente aparecem quando o backup depende de espaço local ou de caminhos montados por NFS.

O ponto chave é que o RMAN consegue enviar o backup direto para o Object Storage utilizando o módulo de integração da própria Oracle. Basicamente, ele trata o bucket como se fosse um media manager. Quem faz essa ponte é a libopc.so, junto com um arquivo .ora que contém os parâmetros do bucket, região, OCID do compartimento e o modo como o RMAN vai se autenticar na OCI.

Quando isso está configurado, você aloca um canal SBT e o RMAN simplesmente joga os backupsets direto no bucket. Não tem etapa intermediária, não tem staging, não tem risco de ficar sem espaço no servidor. O RMAN empacota cada bloco, a libopc pega, converte para chunks e envia via HTTPS para o Object Storage. E sim, o RMAN continua entendendo esses backups para restore, validate, crosscheck, delete… tudo como se fosse um media manager físico.

Antes de qualquer etapa no servidor, **o bucket precisa estar previamente criado no OCI**.  
É nele que o RMAN vai enviar os backupsets diretamente, usando a libopc.  
Sem o bucket criado, o modulo SBT não consegue inicializar o canal, e o RMAN retorna erro de media manager.

No OCI, você deve garantir:

- **Bucket principal** onde ficarão os backups
- **Bucket temporário** para operações internas do módulo
- **Compartment correto** onde os buckets residem.
- **Região idêntica** à usada no parâmetro `OPC_HOST`.

Minha organização básica fica assim:

```
/home/oracle/scripts/backup/          scripts de execução
/home/oracle/scripts/backup/logs/     logs
/u01/app/oracle/immutable/lib/        libopc.so
/u01/app/oracle/immutable/config/     arquivos .ora por banco
```

Um arquivo .ora típico:

```
OPC_HOST=https://objectstorage.sa-saopaulo-1.oraclecloud.com
OPC_BUCKET=BUCKET_BKP_GUSTAVO
OPC_TEMP_BUCKET=BUCKET_PARA_TEMPS
OPC_COMPARTMENT_OCID=ocid1.compartment.oc1..aaaaaaaa123456
OPC_STORAGE_CLASS=Standard
```
> Neste exemplo, estou assumindo que o módulo de backup da Oracle para OCI (*Cloud Backup Module*) já está instalado e com seu wallet configurado, isso pode ter sido feito automaticamente pelo DB System/ExaCS ou manualmente via API Keys.
>
> O arquivo opc_immutable_*.ora mostrado acima contém apenas parâmetros operacionais (bucket, região, compartment).  
> As **credenciais reais** usadas para autenticação no Object Storage (**usuário IAM, fingerprint, API Key, tenancy**, etc.) ficam guardadas de forma segura no **wallet** criado durante a instalação da libopc.so.
>
> Por isso elas **não aparecem** no script RMAN nem precisam ser adicionadas manualmente.

A execução do backup fica assim:

```
#!/bin/bash
VAR_SID=GUSTAVOBANCO
VAR_TIPO=W

rman target / log=/home/oracle/scripts/backup/logs/bkp_${VAR_TIPO}_${VAR_SID}.log &lt;&lt;EOF
CONFIGURE DEFAULT DEVICE TYPE TO SBT_TAPE;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';
CONFIGURE ENCRYPTION FOR DATABASE ON;

RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE SBT_TAPE
    PARMS 'SBT_LIBRARY=/u01/app/oracle/immutable/lib/libopc.so,
           SBT_PARMS=(OPC_PFILE=/u01/app/oracle/immutable/config/opc_immutable_${VAR_TIPO}_${VAR_SID}.ora)';
  BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
  RELEASE CHANNEL c1;
}
EXIT;
EOF
```

O crontab organiza tudo:

```
1 0 * * 1-5 /home/oracle/scripts/backup/exec_bkp.sh D > /home/oracle/scripts/backup/logs/bkp_D.log 2>&1
1 0 * * 6   /home/oracle/scripts/backup/exec_bkp.sh W > /home/oracle/scripts/backup/logs/bkp_W.log 2>&1
1 0 1 * *   /home/oracle/scripts/backup/exec_bkp.sh M > /home/oracle/scripts/backup/logs/bkp_M.log 2>&1
```

A parte que faz mais diferença no dia a dia é que você deixa de depender de I/O local. Não importa se o servidor está com espaço baixo ou se o filesystem está com problema, porque o backup ignora tudo isso.  
Além disso, a compressão do RMAN reduz bem o tráfego, e a criptografia já entra no fluxo de forma nativa, o que resolve requisitos de segurança sem adicionar nada no caminho.

Outra vantagem forte: cada banco pode ter seu próprio bucket, sua própria retenção e sua própria organização. Fica mais limpo, cada ambiente isolado, nada mistura, e o restore fica muito mais previsível.

E sim, restore direto do bucket funciona. O RMAN baixa o backupset novamente via libopc e reconstrói tudo no servidor de destino. O mesmo vale para validate e crosscheck, que continuam funcionando exatamente como no ambiente local.

No fim das contas, é uma solução simples, robusta e fácil de manter.

![](exterminador-683x1024.png)
