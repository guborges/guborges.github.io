---
title: "Trocar PMEM em storage cell Exadata sem derrubar o banco"
date: 2026-07-20T05:56:53-03:00
draft: false
categories: ["ASM & Storage", "Oracle DBA"]
tags: ["ASM", "Exadata", "Oracle", "PMEM"]
summary: "Como substituir um módulo PMEM em predictive failure num storage cell Exadata mantendo os bancos disponíveis via redundância ASM."
cover:
  image: "imagem_2026-07-20_054702119.png"
---

Um storage cell Exadata X9M-2 acusou predictive failure num módulo PMEM. O componente ainda não morreu, mas o alerta é claro: vai falhar, troque logo. O detalhe é que trocar PMEM exige desligar o cell inteiro, e isso com os bancos em produção rodando o tempo todo.

Esse é o tipo de atividade que assusta na primeira vez e vira rotina depois que você entende o que segura a barra: a redundância do ASM. Enquanto os grid disks do cell puderem sair de cena sem quebrar a redundância dos disk groups, o cell pode ser desligado que nenhum banco cai. O procedimento inteiro gira em torno de provar isso antes de mexer em qualquer coisa.

Vou usar nomes genéricos aqui: o cell é `stgcell03`, os disk groups são `DATA` e `RECO`. O módulo com defeito está em `CPU 0 / DIMM 12`.

## O que é PMEM e por que trocar dá trabalho

PMEM é a memória persistente (Intel Optane / DCPMM) que fica no barramento de memória dos storage cells X8M-2 e X9M-2 com storage HC ou EF. Ela funciona como uma camada de cache e de commit accelerator na frente do flash, entregando latência de leitura na casa dos microssegundos via RDMA.

Por estar fisicamente encaixada nos slots de DIMM da placa, não dá pra trocar com a máquina ligada. Diferente de um disco, que você puxa a quente, o módulo PMEM obriga o cell a ser desligado. E um storage cell fora do ar significa que metade do espelho daquele pedaço de dado sai temporariamente do jogo. Daí a importância da checagem de redundância.

Um ponto que a doc oficial deixa claro e que muda o nível de risco: se o PMEM cache está em write-through, o dado já está persistido no grid disk e não há nada a recuperar. Em write-back, o Exadata precisa fazer o flush antes de liberar a troca. O alerta de "pode trocar" só aparece depois que esse flush termina.

## Passo 1: identificar o módulo com defeito

O primeiro comando roda no próprio cell, como root, e mapeia os componentes físicos:

```
cellcli -e "list diskmap"
```

No meio da saída, o módulo problemático aparece assim:

```
PMEM_0_12  8089-a2-2110-000025c7  "CPU: 0; DIMM: 12"  warning - predictive failure  127G  PM_01_stgcell03  /dev/dax0.0
```

`warning - predictive failure` é o estado que interessa. Anote o slot (`CPU 0 / DIMM 12`) e o serial, você vai precisar depois pra confirmar que trocou o módulo certo.

## Passo 2: confirmar que já pode trocar

Aqui está a diferença entre predictive failure e failed. Se o status fosse `critical`/`failed`, você poderia trocar de imediato. Em predictive failure, tem que esperar o Exadata liberar. Consulte o histórico de alertas daquele módulo:

```
cellcli -e "list alerthistory where metricObjectName='PMEM_0_12' detail"
```

A saída traz dois eventos. O que libera a troca é o segundo:

```
alertMessage:  PMEM can be replaced now.
alertAction:   The data on this PMEM module has been successfully flushed.
               Please replace the PMEM module. The cell needs to be shut
               down while the PMEM module is replaced.
```

`PMEM can be replaced now` é o sinal verde da própria máquina. Se essa mensagem não estiver lá, não prossiga. Espere. O flush ainda não terminou e o módulo ainda tem dado a proteger.

## Passo 3: garantir que o ASM aguenta perder o cell

Este é o passo que separa uma manutenção tranquila de um incidente. Antes de desligar qualquer coisa, você precisa provar que tirar os grid disks desse cell não quebra a redundância de nenhum disk group.

Vale conferir também o `failgroup_repair_time` dos disk groups, conectando na instância ASM num nó de banco:

```
SELECT dg.name AS diskgroup, a.name, a.value
FROM   v$asm_diskgroup dg, v$asm_attribute a
WHERE  dg.name IN ('DATA','RECO')
AND    dg.group_number = a.group_number
AND    a.name = 'failgroup_repair_time';
```

Se a janela for durar menos que o repair time (normalmente 24h numa configuração de storage), o ASM não vai começar a dropar disco durante a troca. Uma parada de uma hora está folgada dentro disso.

Agora o comando central, de volta no cell:

```
cellcli -e list griddisk attributes name,asmmodestatus,asmdeactivationoutcome
```

O que você quer ver é `ONLINE` e `Yes` em todas as linhas:

```
DATA_CD_00_stgcell03   ONLINE   Yes
DATA_CD_01_stgcell03   ONLINE   Yes
...
RECO_CD_11_stgcell03   ONLINE   Yes
```

`asmdeactivationoutcome=Yes` significa que o ASM consegue tirar aquele grid disk de operação sem perder redundância. Se qualquer linha voltar `No`, pare tudo. Desligar o cell nessa condição faz o ASM desmontar o disk group afetado, e os bancos caem de forma abrupta. `No` quer dizer que o disco parceiro em outro cell já está fora, ou que a redundância daquele grupo já está comprometida. Nesse caso, o problema é outro e precisa ser resolvido antes.

## Passo 4: inativar os grid disks e desligar

Com todas as linhas em `Yes`, inative os grid disks. Isso coloca os discos OFFLINE no ASM de forma controlada:

```
cellcli -e alter griddisk all inactive
``````
GridDisk DATA_CD_00_stgcell03 successfully altered
GridDisk DATA_CD_01_stgcell03 successfully altered
...
```

Confirme que a inativação segurou e que o `asmdeactivationoutcome` continua `Yes`:

```
cellcli -e list griddisk attributes name,asmmodestatus,asmdeactivationoutcome
cellcli -e list griddisk
```

Todas as linhas devem estar `OFFLINE`/`Yes` no primeiro comando e `inactive` no segundo. Só depois de confirmar isso o cell pode ser desligado:

```
shutdown -h now
```

Um storage cell leva alguns minutos pra desligar por completo. Não assuma que a máquina caiu depois de dez segundos. Se tiver acesso ao ILOM, dá pra confirmar o estado real:

```
show /System   ->   power_state = Off
```

Com o cell desligado e confirmado como off, o módulo pode ser trocado fisicamente.

## Passo 5: religar e verificar a peça nova

Depois da troca, o cell normalmente religa sozinho. Se não subir, o power on sai pelo ILOM (`start /System`). Um detalhe que vale saber: depois da troca, o Management Server remapeia o novo módulo, e isso dispara um reboot automático adicional. No meu caso o cell fez dois ciclos de boot antes de estabilizar. Não é problema, é esperado, mas se você não souber disso vai achar que algo travou.

Com o cell no ar de novo, confirme que a peça nova está saudável:

```
cellcli -e "list physicaldisk where diskType=PMEM"
``````
PMEM_0_12   8089-a2-2315-00001831   normal
```

Repare no serial: é diferente do que você anotou no passo 1, e o status agora é `normal`. Módulo novo, reconhecido, sem falha.

## Passo 6: reativar e acompanhar o resync

Reative os grid disks:

```
cellcli -e alter griddisk all active
```

A reativação começa em alguns minutos e cada disco entra em `SYNCING` antes de voltar pra `ONLINE`. Esse resync usa Fast Mirror Resync: ele restaura só os extents que teriam sido escritos enquanto o disco esteve fora, não dispara rebalance do ASM. Por isso é rápido comparado a uma reconstrução completa.

Pra acompanhar sem ficar rolando a tela, um loop simples:

```
while :; do clear; cellcli -e list griddisk attributes name,asmmodestatus; date; sleep 10; done
```

Os discos vão de `SYNCING` pra `ONLINE`. No meu run, o RECO terminou primeiro e o DATA levou perto de uma hora e meia pra fechar tudo. Não feche a janela de manutenção enquanto cada disco não estiver `ONLINE`. E antes de tirar qualquer outro cell de operação, o sync deste tem que estar completo, senão você perde redundância dos dois lados ao mesmo tempo.

Pra fechar, limpe os alertas e valide a configuração do cell:

```
cellcli -e drop alerthistory all
cellcli -e alter cell validate configuration
```

## Validação técnica

O que foi confirmado na documentação oficial da Oracle:

A sequência `list diskmap` → `list alerthistory ... detail` → checar `asmdeactivationoutcome` → `alter griddisk all inactive` → `shutdown` → `alter griddisk all active` é o procedimento documentado (Oracle Docs, MOS 1188080.1). A mensagem `PMEM can be replaced now` no fluxo de predictive failure está no Service Manual do X9-2. O comportamento de write-through (sem recuperação) versus write-back (resilvering) está na doc de PMEM. E o aviso de que derrubar o cell com `asmdeactivationoutcome=No` desmonta o disk group também é oficial.

O que é comportamento observado no meu ambiente e pode variar: o número de reboots automáticos após a troca (vi dois), o tempo de resync (depende do volume de escrita durante a parada e do tamanho dos disk groups), e os tempos exatos. Nomes de disk group, grid disk e cell aqui são genéricos.

O que depende do seu ambiente: se o PMEM está em write-back ou write-through muda se há flush a esperar; o `failgroup_repair_time` pode estar configurado diferente; e a topologia de redundância (normal ou high) muda a folga que você tem pra tirar cells de operação.

## Conclusão

A troca de PMEM em si é um trabalho de poucos comandos. O que faz a diferença é o passo 3: confirmar que `asmdeactivationoutcome` está `Yes` em tudo antes de desligar. Esse é o comando que decide se a manutenção é um não-evento ou um incidente com banco caindo em produção.

Se você respeita a ordem, confirma a liberação da peça, valida a redundância, inativa, desliga, troca, religa, reativa, espera o sync fechar, o banco nem sente. Foi exatamente o que aconteceu aqui: cell fora por mais de uma hora, zero impacto nos bancos.
