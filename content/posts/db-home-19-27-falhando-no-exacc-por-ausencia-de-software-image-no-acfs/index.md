---
title: "DB Home 19.27 falhando no ExaCC por ausência de Software Image no ACFS"
date: 2026-01-12T07:00:00-03:00
draft: false
categories: ["Oracle DBA", "Scripts & Automação"]
cover:
  image: "dbhomeerror.png"
---

Este artigo documenta um incidente real ocorrido em um ambiente **Oracle Exadata Cloud@Customer** versão 19.27, no qual a criação de um novo DB Home falhava silenciosamente em um dos clusters RAC, sem geração de logs e sem erro técnico acionável. O objetivo é registrar exatamente o que ocorreu, quais comandos foram executados, como o problema foi investigado e qual foi a solução prática aplicada para destravar o ambiente.

O ambiente possuía dois clusters RAC independentes, aqui chamados de RAC-A e RAC-B, ambos com quatro nós. A atividade consistia na criação de um novo Oracle DB Home 19.27. Como pré-requisito, a Software Image correspondente à versão 19.27 já havia sido previamente criada na OCI. No RAC-B, a criação do DB Home ocorreu normalmente. No RAC-A, a mesma operação falhava desde o início.

Ao tentar criar o DB Home no RAC-A, as mensagens retornadas eram as seguintes:

```text
ECRA error: {} when checking status of the software image download. Retry the operation.
```
```text
The database host services on node-a01 are unavailable or not in the active state.
Please refer to the following MOS note and restart services:
How to Manage and Check Health of Exadata Cloud Related Services - ExaDB-D, ExaDB-XS & ExaCC (Doc ID 2926648.1)
```
```text
Create DB Home operation failed.
Refer to work request ID xxxxxxxx when opening a Service Request at My Oracle Support.
```

Apesar de a mensagem indicar indisponibilidade dos serviços do host de banco de dados, a análise prática não confirmava essa condição. Os serviços citados encontravam-se ativos, operantes e sem qualquer anomalia aparente no nó, inclusive após reinicialização controlada dos agentes recomendados. Ou seja, o erro apontava para uma causa que não se materializava no ambiente.

As mensagens retornadas pelo processo eram genéricas no sentido de não refletirem o estado real do sistema. Embora indicassem indisponibilidade de serviços, não informavam qual serviço específico estaria com problema, qual etapa do fluxo havia falhado ou qual recurso estava efetivamente ausente. Na prática, a criação do DB Home simplesmente não avançava e nenhum Oracle Home era criado em nenhum dos nós do RAC-A, mesmo com todos os serviços críticos em estado ativo.

A primeira etapa da investigação foi a busca por logs. Foram verificados os caminhos padrão utilizados pelo ExaCC para operações de DB Home, principalmente:

```text
/var/opt/oracle/log/
```

Nenhum log relacionado à criação do DB Home foi encontrado. Não havia diretório de execução, não havia pilot log e não havia qualquer evidência técnica além das mensagens genéricas exibidas na tela.

Em seguida, foi tentada a recuperação da comunicação com os serviços do Exadata por meio da reinicialização dos agentes responsáveis pela orquestração das operações de banco. Os seguintes comandos foram executados no nó do RAC-A:

```text
systemctl status dbcsagent
systemctl stop dbcsagent
systemctl start dbcsagent
systemctl status dbcsagent

systemctl status dbcsadmin
systemctl stop dbcsadmin
systemctl start dbcsadmin
systemctl status dbcsadmin
```

Os serviços subiram corretamente, permaneceram ativos e saudáveis, porém a tentativa de criação do DB Home continuava falhando exatamente da mesma forma. Também foram realizadas tentativas adicionais via OCI, utilizando automação padrão da plataforma. O comportamento foi idêntico: falha silenciosa, sem log e sem qualquer informação adicional.

Nesse ponto, ficou claro que o problema não estava relacionado à versão do banco, à Grid Infrastructure ou à inexistência da Software Image na OCI, já que o RAC-B havia criado o DB Home com sucesso utilizando a mesma versão 19.27 e a mesma image previamente criada. A investigação passou então para uma comparação direta entre os clusters.

No RAC-B, que funcionou corretamente, foi identificado que a software image do DB Home havia sido criada e armazenada no ACFS, no diretório padrão:

```text
/acfs01/dbaas_acfs/images
```

Dentro desse diretório, existia o path correspondente à image criada automaticamente pelo processo:

```text
/acfs01/dbaas_acfs/images/ocid1.databasesoftwareimage.oc1.sa-region-1.xxxxxxxxxxxxxxxxxxxxx
```

Ao validar o mesmo caminho no RAC-A, foi constatado que essa software image simplesmente não existia. Ou seja, apesar de a image existir na OCI, ela não havia sido materializada no ACFS do cluster problemático, e o fluxo automatizado não gerava qualquer log ou erro claro indicando essa falha.

Com essa evidência, foi adotada uma abordagem manual. A software image existente no RAC-B foi copiada diretamente para o mesmo caminho no RAC-A, respeitando exatamente a estrutura esperada pelo DBaaS. O comando executado foi:

```text
scp -r /acfs01/dbaas_acfs/images/ocid1.databasesoftwareimage.oc1.sa-region-1.xxxxxxxxxxxxxxxxxxxxx \
oracle@node-a01:/acfs01/dbaas_acfs/images/
```

Após a cópia da image para o RAC-A, foi executado o precheck completo da criação do DB Home utilizando o dbaascli, forçando explicitamente o uso da software image por meio do parâmetro imageFilePath. O comando utilizado foi:

```text
dbaascli dbHome create \
  --version 19.27.0.0.0 \
  --oracleHomeName dbhome_v10_DB_IMAGE_1927_20251211 \
  --oracleHome /u02/app/oracle/product/19.0.0.0/dbhome_11 \
  --executePrereqs \
  --imageFilePath /acfs01/dbaas_acfs/images/ocid1.databasesoftwareimage.oc1.sa-region-1.xxxxxxxxxxxxxxxxxxxxx/ocid1.databasesoftwareimage.oc1.sa-region-1.xxxxxxxxxxxxxxxxxxxxx.zip \
  --waitForCompletion true
```

A execução foi realizada com o DBAAS CLI versão 25.1.2.0.0. O precheck validou com sucesso todas as etapas, incluindo validação da image, checagem de versão do Oracle Home, compatibilidade com a versão do Grid Infrastructure, verificação de espaço em disco, checagem de usuários e grupos, validação dos nós do cluster, CVU prechecks e unzip da image localmente. A etapa de download da image foi corretamente ignorada, pois a image já estava presente no ACFS.

Com o precheck aprovado, foi executada a criação efetiva do DB Home no RAC-A utilizando o mesmo imageFilePath. O comando foi:

```text
dbaascli dbHome create \
  --version 19.27.0.0.0 \
  --oracleHomeName dbhome_v10_DB_IMAGE_1927_20251211 \
  --oracleHome /u02/app/oracle/product/19.0.0.0/dbhome_11 \
  --imageFilePath /acfs01/dbaas_acfs/images/ocid1.databasesoftwareimage.oc1.sa-region-1.xxxxxxxxxxxxxxxxxxxxx/ocid1.databasesoftwareimage.oc1.sa-region-1.xxxxxxxxxxxxxxxxxxxxx.zip \
  --waitForCompletion true
```

Durante a execução, o dbaascli percorreu todas as etapas esperadas do processo, incluindo validação da image, checagem de versões, verificação de espaço em disco, unzip local, clonagem do Oracle Home, cópia do software para os nós remotos, attach do home no inventory, execução dos scripts de root, atualização do inventory global e geração dos metadados do DB Home. Todas as etapas foram concluídas com sucesso.

Ao final da execução, o novo Oracle Home foi criado corretamente no caminho:

```text
/u02/app/oracle/product/19.0.0.0/dbhome_11
```

O DB Home ficou funcional e pronto para aplicação dos patches customizados planejados.

Este caso mostra que, mesmo com a Software Image previamente criada na OCI, a criação do DB Home pode falhar silenciosamente caso essa image não seja corretamente materializada no ACFS do cluster. A ausência de logs torna o troubleshooting especialmente difícil e pode induzir a diagnósticos incorretos sobre indisponibilidade de serviços. Em ambientes com múltiplos clusters, a comparação direta com um cluster funcional pode revelar rapidamente inconsistências invisíveis à automação. Quando a software image não existe no ACFS, a criação do DB Home simplesmente não ocorre, e forçar manualmente o uso da image via dbaascli pode ser a forma mais rápida e eficaz de destravar o ambiente sem depender imediatamente da abertura de um Service Request.
