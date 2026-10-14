---
title: "Oracle Data Guard além da replicação: como realmente proteger dados, garantir continuidade e operar sem sustos"
date: 2025-10-20T07:50:00-03:00
draft: false
categories: ["Data Guard", "Oracle DBA"]
tags: ["Oracle"]
cover:
  image: "dg_2.png"
---

Este **não** é um tutorial passo a passo. É um **manual de engenharia** para DBAs: arquitetura de Data Guard, modos de proteção, consistência, latência, sizing, diagnósticos, automação, monitoramento e decisões de design. O foco é **proteger dados de verdade**, não apenas “copiar redo”. Em vários trechos eu conecto os conceitos ao que vivenciei no **PowerLab** do meu mentor, Roberto Sobrinho (arquitetura, modos de proteção, operação com e sem Broker), para dar contexto prático sem transformar isso em receita de bolo.

## 1) Primeiro princípio: Data Guard é sobre **RPO/RTO**, não sobre “réplica”

- **RPO (Recovery Point Objective)**: quanta perda de dados o negócio tolera?
  - **Zero** -> exige **SYNC + AFFIRM** e caminho de rede/IO determinísticos.
  - **>0 (pequena)** -> **ASYNC + NOAFFIRM** com monitoramento de *transport/apply lag*.
- **RTO (Recovery Time Objective)**: quanto tempo para retomar serviço?
  - Governado por **Switchover/Failover + automação + prontidão operacional** (serviços, listeners, SRLs corretos, *flashback on* para reinstate, etc.).

Escolha de modo de proteção e desenho de rede/discos é **derivada de RPO/RTO**. Sem isso, você está só “replicando”.

## 2) Arquitetura, o que realmente roda e onde o gargalo aparece

### 2.1 Caminho do redo

1. **LGWR** produz *redo* no primário.
2. **LNS** pega o fluxo e envia (SYNC/ASYNC) pro standby.
3. **RFS** no standby recebe e escreve em **SRL** (ou ARCH, se for gap).
4. **MRP** aplica do SRL/ARCH para os datafiles do standby.
5. **ARCn** no primário faz *shipping* de archived (pós-switch) para cobrir *gaps*.

### 2.2 Onde o gargalo costuma estar

- **Transporte**: LNS travado esperando ACK (**SYNC/AFFIRM**) -> latência de rede/IO no SRL remoto.
- **Recepção/Gravação**: RFS escrevendo SRL em disco lento ou saturado (I/O do standby).
- **Aplicação**: MRP “engasga” em *single-thread* com cargas de muitos *direct path* ou log switches frequentes (muitos SRLs pequenos).

**Sintomas/Waits para reconhecer**

- No **primário**: log file sync (usuários), LNS wait on SENDREQ, LGWR wait on remote (nomes variam por versão).
- No **standby**: RFS write, MRP wait for redo, control file sequential read (catálogo), db file sequential/scattered read na aplicação.

## 3) Modos de proteção e o que **de fato** muda no commit

| Modo | Transporte | Confirmação | Efeito no `COMMIT` | RPO típico |
| --- | --- | --- | --- | --- |
| **Maximum Protection** | SYNC | AFFIRM | `COMMIT` só confirma após SRL remoto *flush*. Se standby cair -> primário **desliga**. | 0 |
| **Maximum Availability** | SYNC | AFFIRM | Mesmo do acima, **mas** primário segue online se standby indisponível; quando volta, sincroniza. | 0 (sob condição) |
| **Maximum Performance** | ASYNC | NOAFFIRM | `COMMIT` não espera standby. Melhor latência. Pode haver perda mínima. | >0 |

Esse mapeamento de comportamento (incluindo a analogia do “motoboy do redo” do Roberto Sobrinho) é base do PowerLab; aqui expandimos impactos em commit e disponibilidade.

**Decisão prática**

- **RPO zero real** -> *latência de ida+volta < 5–7 ms* estável, discos do standby com **baixa jitter**, e rede dedicada/QoS.
- Links “normais” entre DCs (10–20 ms e/ou variável) -> **Maximum Availability** pode virar **drama operacional** (picos de latência “travando” commits). Nesses casos: **Maximum Performance** + bons alertas é a opção honesta.

## 4) Sizing e layout: SRL, redo, flashback, FRA números que evitam susto

### 4.1 **Standby Redo Logs (SRL)**

- **Quantidade**: por *thread*, **(N grupos online + 1)**. Se RAC com 4 threads, dimensione por thread.
- **Tamanho**: igual ao *redo online* do primário (ou maior) para evitar *fragmentação de switch* no standby.
- **Discos**: SRL é caminho crítico do commit (em SYNC/AFFIRM). Garanta **IOPS/latência** comparáveis ao redo do primário.

### 4.2 **Log switch e geração de redo**

- Muitos *switches* pequenos = mais cabeçalhos, mais MRP *stop/start* e mais *archive shipping*. Prefira **logs maiores** vs. frequentes *switches* se o perfil permitir.

### 4.3 **Flashback Database**

- **Obrigatório** se você quer **reinstate** rápido após failover/switchover. Sem flashback, o ex-primário dificilmente volta “no amor” ao DG.
- Calcule **retention target** para cobrir **janela de incidente + diagnóstico + ação** (muitas empresas subdimensionam e só descobrem quando precisam).

### 4.4 **FRA (Fast Recovery Area)**

- Some: archived (prim+stby, se for local), flashback logs, backups, e margem para *burst*.
- Se usar **Active Data Guard** com queries pesadas, reserve I/O para datafiles e **separe** SRL/redo em LUNs de baixa latência.

## 5) Rede e latência: parâmetros que realmente movem a agulha

- **NET_TIMEOUT / REOPEN (LOG_ARCHIVE_DEST_n)**: delimite quanto tempo o primário “espera” o standby antes de degradar ou reabrir o canal.
- **Compressão** de redo (**ASYNC**) ajuda em links estreitos; avalie CPU vs. largura de banda real.
- **Encryption (SQLNET)** é padrão em muitas empresas; meça impacto.
- **Multipath**/rede redundante: duas rotas físicas ou VLANs separadas reduzem *flaps*.

**Teste de latência “de verdade”**

- Faça **ping de payload e jitter** (não só ICMP vazio). Meça *p95/p99*. Data Guard sofre com **jitter**, não apenas com a média.

## 6) Active Data Guard, consistência e integridade

- **Read Only with Apply**: consultas em tempo real; otimiza relatórios sem bater no primário.
- **Block Media Recovery** no standby reduz janelas de impacto por corrupção setorial.
- **Lost write detection**: DG ajuda a **detectar e mitigar** *lost writes* (checksum + divergência aplicada/exibida).
- **Consistência de reports**: documente *read consistency* e *lag tolerado* com as áreas usuárias (BI/analytics).

> **Regra de ouro**: **não transforme o standby em outro prod**. Limite workloads; proteja **I/O do SRL/MRP**.

## 7) Broker vs. Manual - pragmatismo

- **Sem Broker**: controle fino, scripts previsíveis, ótimo para *debug* e ambientes peculiares. (PowerLab cobre bem as rotinas.)
- **Com Broker (DGMGRL)**: reduz erro humano, oferece **FSFO (Fast-Start Failover)** com *Observer*, *profiles* e validações automáticas. Excelente para operações padronizadas e equipes com *runbooks*.

**Minha recomendação**

- **Use Broker** em produção, **mantenha scripts manuais** para *break glass* (troubleshooting e contingência).

## 8) FSFO e Far Sync: quando você precisa de *verdadeiro* RPO zero

- **FSFO (Fast-Start Failover)**: *Observer* externo decide o failover quando critérios são atendidos (lag, disponibilidade, health). Reduz RTO humano.
- **Far Sync**: ponto “buffer” próximo ao primário para manter **SYNC + AFFIRM** de baixa latência até o Far Sync e **ASYNC** deste até o standby remoto.
  - Útil quando o DC remoto tem latência “alta demais” para commits síncronos, mas o DC local permite *round-trip* curto até o Far Sync.

## 9) RAC, PDB/CDB e serviços: detalhes que definem a experiência

- **RAC**: SRL **por thread** e *role-based services* (p.e., -role PRIMARY/STANDBY) controlados por SRVCTL/Broker.
- **CDB/PDB**: switchover/failover operam no **CDB**; confirme open mode das PDBs e o *startup trigger* apropriado.
- **Services**: use -role e *failover methods* (BASIC/TCPS, ao seu padrão). Mapeie serviços de leitura **apenas** no standby.

## 10) Automação e monitoramento (o que e como medir)

### 10.1 Métricas vitais

- **Transport Lag** e **Apply Lag** (V$DATAGUARD_STATS).
- **MRP status/progress** (V$MANAGED_STANDBY).
- **SRL usage** (duração de reaproveitamento e waits) e **log switch interval**.
- **Redo generation rate** (MB/s) vs. **rede disponível** (real, não teórica).
- **Flashback space pressure** (para garantir reinstate).
- **Event histogram** para latência de aplicação.

### 10.2 Alarmes inteligentes

- *Lag crescentes* com correlação de **rede/IO**.
- **Mudança de modo** (SYNC -> ASYNC) e **destinations inativas**.
- **Quase estouro da FRA/Flashback**.
- **FSFO state** (Observer down).

### 10.3 Rotinas de saúde (diárias/semanais)

- Verificar `VALID_FOR` dos DESTs (para não ter destino “mudo” após role transition).
- Ensaiar **Switchover** trimestral com janela acordada.
- Exercitar **Reinstate** (flashback test) em ambiente de teste. (PowerLab fecha com recomposição do ambiente.)

## 11) Troubleshooting dirigido a causa (não a sintoma)

1. **Lag alto**
   - **Transporte**: veja `LNS wait` e latência de rede. Teste I/O de SRL (fio randômico pequeno).
   - **Aplicação**: `MRP` saturado por DDLs pesados/LOBs/direct path -> avalie *parallel recovery*, *precreate datafiles*, *nodelay*.
2. **Switchover preso**
   - Sessões não finalizam no primário; datafiles offline; SRLs faltantes em threads RAC; *flashback off* no futuro primário.
3. **FSFO não executa**
   - *Observer* indisponível, perfis conservadores, *thresholds* de lag exagerados, *ping* ICMP passando mas TCP com bloqueio intermediário.
4. **Corrupção/“perda de write”**
   - Use **ADG** pra detectar divergência; faça **Block Media Recovery** no standby; avalie **DB_LOST_WRITE_PROTECT** (nível e custo).

## 12) Segurança e compliance

- **Criptografia em trânsito** (TCPS/Native Network Encryption) + **em repouso** (TDE) valide impacto na taxa de redo.
- **Segregação de credenciais** para Data Guard Broker, *Observer* e automações.
- **Auditoria**: eventos de role transition e mudanças de `LOG_ARCHIVE_DEST_n`.

## 13) Checklists prontos (de campo)

### 13.1 Antes de habilitar SYNC + AFFIRM

- RTT p95 <= 7 ms (testado com payload), jitter baixo.
- SRL em storage de **baixa latência** (mesmo nível do redo).
- NET_TIMEOUT coerente com SLA (nem curto a ponto de “flapar”, nem longo a ponto de travar commit).
- Teste de *switch* de rota/Link.

### 13.2 Antes do Switchover

- apply lag = 0, transport lag = 0.
- **SRL ok por thread** no futuro primário.
- **Flashback on** em ambos.
- Serviços prontos com -role.

### 13.3 Após Failover

- **Open resetlogs** do novo primário validado.
- **Reinstate** do ex-primário via flashback (se possível).
- Revalide **destinations** e `VALID_FOR`.
- Documente *incidente -> ação -> lições*.

## 14) Guia de decisão (em 60 segundos)

- **RPO = 0** e **latência estável**? -> **Maximum Availability** (ou Protection, se cultura aceitar shutdown na perda do standby) + **FSFO** + SRL premium.
- **RPO > 0** ou **latência variável**? -> **Maximum Performance** com compressão/otimização, monitoramento de lag e **procedimentos de failover ágeis**.
- Link remoto com RTT alto mas quer RPO 0? -> **Far Sync** perto do primário.
- Time pequeno? -> **Broker** para orquestração + **scripts de emergência**.

## 15) Conclusão (e provocação)

Se você trata Data Guard como **“replicar archivelog”**, vai acertar a maior parte dos dias e falhar *exatamente* no dia em que o negócio mais precisar. Data Guard **é engenharia de continuidade**: começa em **RPO/RTO**, passa por **rede/IO**, depende de **sizing correto**, exige **observabilidade inteligente** e termina em **disciplina operacional** (switchover ensaiado, failover consciente e reinstate previsível).

> **Proteção de dados não é um *feature toggle*; é um projeto.**  
> E o seu projeto só termina quando você consegue rodar o *Chaos Day* do PowerLab no seu ambiente e ainda dormir tranquilo.

![](image-683x1024.png)
