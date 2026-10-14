---
title: "Oracle AI Agent Memory SDK no 26ai: o que muda para o DBA"
date: 2026-05-01T20:14:25-03:00
draft: false
categories: ["IA", "Oracle 26Ai"]
tags: ["26ai", "agent memory", "ai agents", "dba", "governance", "Oracle"]
summary: "Oracle Agent Memory 26.4 transforma o banco em sistema de registro para agentes de IA. Saiba o que muda para o DBA: schema, governança, ciclo de vida e riscos."
cover:
  image: "memoria.png"
---

A Oracle lancou o Oracle AI Agent Memory SDK 26.4 e posicionou o banco como sistema de registro para memoria de agentes de IA. A doc oficial mostra o lado do desenvolvedor. Este post mostra o lado de quem opera o ambiente, faz o GMUD passar e responde por governanca, ciclo de vida e crescimento de tablespace. E mostra dois casos de uso onde memoria persistente faz diferenca real para quem trabalha com banco.

## O cenário

Oracle AI Database 26ai, com o SDK Python `oracleagentmemory==26.4.0` instalado em uma máquina de aplicação. Conexão via `oracledb` (pool ou single connection) para um schema dedicado ao armazenamento de memórias. Modelos de LLM e embedding configurados em OCI Generative AI, vLLM local, OpenAI, Anthropic ou Gemini, conforme a necessidade do agente.

## O problema

LLM esquece tudo entre sessões. Toda interação começa do zero. Para resolver isso, equipes vinham montando stacks fragmentadas: vector store separado para busca semântica, graph database para relações, document store JSON para histórico, e o banco relacional do lado para tudo o que importa. Quatro tecnologias, quatro pipelines, quatro modelos de segurança, quatro formatos de backup.

A Oracle entrou neste espaço dizendo o óbvio: o Oracle AI Database já tem vector, JSON, relacional e graph no mesmo motor. Memória de agente pode viver onde os dados corporativos já vivem. O resultado é o Oracle Agent Memory SDK, com a memória inteira persistida no banco e governada pelos mesmos controles que o resto da empresa já usa.

## Como funciona o SDK

Instalação documentada:

```text
pip install "oracleagentmemory==26.4.0"
```

Inicialização do objeto principal:

```text
from oracleagentmemory.core.oracleagentmemory import OracleAgentMemory
from oracleagentmemory.apis.searchscope import SearchScope
from oracleagentmemory.core.embedders.embedder import Embedder
from oracleagentmemory.core.llms.llm import Llm

embedder = Embedder(model="oci/cohere.embed-multilingual-v3.0")
llm = Llm(model="anthropic/claude-sonnet-4-6")
db_pool = ...  # connection ou pool oracledb
memory = OracleAgentMemory(connection=db_pool, embedder=embedder, llm=llm)
```

Para mostrar o que isso vira na prática, vou usar dois exemplos que fazem sentido para quem trabalha com banco. Nada de exemplo genérico de chatbot.

## Caso 1: agente DBA assistente operacional

Cenário: um agente que responde perguntas operacionais sobre o ambiente. Não substitui o DBA. Substitui a pasta de runbooks e o histórico de incidentes que sempre acaba perdido entre as conversas de Teams.

A memória aqui guarda padrões observados, decisões tomadas em incidentes anteriores, preferências do time. Quando o DBA pergunta "qual foi a última vez que TBS_APP cresceu rápido?", o agente busca no histórico em vez de partir do zero.

```text
# DBA registra contexto durante e depois do incidente
thread = memory.create_thread(user_id="dba_gustavo")

thread.add_messages([
    {
        "role": "user",
        "content": (
            "TBS_APP cresceu 8 GB hoje entre 02h e 04h. "
            "Job de carga mensal rodou nesse intervalo."
        ),
    },
    {
        "role": "assistant",
        "content": (
            "Crescimento alinhado com a janela do job de carga. "
            "Sem acao imediata necessaria. Recomendo monitorar proxima execucao."
        ),
    },
])

thread.add_memory(
    "TBS_APP cresce de 5 a 10 GB no dia da carga mensal, sempre entre 02h e 04h. "
    "Comportamento esperado, nao e incidente."
)

thread.add_memory(
    "Padrao do ambiente: validar AWR antes de qualquer ALTER SYSTEM em horario comercial."
)
```

Em outra sessão, semanas depois, o DBA pergunta sobre crescimento da mesma tablespace. O agente busca pelo escopo do usuário e devolve as memórias relevantes:

```text
results = memory.search(
    query="crescimento tablespace TBS_APP",
    scope=SearchScope(user_id="dba_gustavo")
)

for result in results:
    print(f"- [{result.record.record_type}] {result.content}")
```

A diferença prática: sem memória, o agente responde com receita genérica de "verifique dba_segments e considere extensão". Com memória, devolve "esse crescimento é o padrão do job mensal, validado por você em janeiro, não é alarme". O contexto operacional acumula.

O risco é o oposto: memória poluída faz o agente sustentar uma decisão errada. Se em janeiro o DBA registrou um workaround tático como "verdade permanente", o agente vai repetir esse erro indefinidamente. Vale planejar revisão periódica das memórias críticas antes de pôr esse padrão em produção.

## Caso 2: agente de tuning com memória de queries problemáticas

Cenário diferente. Agente focado em performance, com memória de planos de execução que já causaram problema, queries que exigiram intervenção, janelas de manutenção do ambiente.

Quando aparece uma query nova com perfil parecido (mesmas tabelas, padrão de join, volume estimado), o agente já sabe que tem histórico relevante.

```text
thread = memory.create_thread(user_id="tuning_team")

# Registro de uma query problematica vista em producao
thread.add_messages([
    {
        "role": "user",
        "content": (
            "Query do batch noturno em ORDERS join ORDER_ITEMS "
            "fez full scan e estourou PGA. SQL_ID 8h2k9..."
        ),
    },
    {
        "role": "assistant",
        "content": (
            "Diagnostico: estatisticas de ORDER_ITEMS estavam desatualizadas. "
            "Apos DBMS_STATS.GATHER_TABLE_STATS, plano voltou para nested loop com index range scan."
        ),
    },
])

thread.add_memory(
    "ORDERS join ORDER_ITEMS e sensivel a estatisticas. "
    "Historico: gather de estatisticas apos carga mensal evita full scan."
)

thread.add_memory(
    "Janela de manutencao do ambiente: domingo 22h as 02h. "
    "Operacoes pesadas (rebuild de indice, gather full) so nesta janela."
)
```

Semanas depois, em outro incidente:

```text
results = memory.search(
    query="full scan ORDER_ITEMS plano de execucao",
    scope=SearchScope(user_id="tuning_team")
)
```

O agente devolve o contexto: "essa combinação já apareceu, causa foi estatística, fix foi gather, janela permitida é domingo à noite". Em vez de o time refazer toda a investigação, parte do diagnóstico anterior.

O ganho não está em automatizar o tuning. Está em não perder o conhecimento que já foi construído. Que é exatamente o que acontece hoje com pasta de runbooks abandonada e ticket fechado sem registro de causa raiz.

## O que muda para o DBA

Aqui está o que a doc oficial não enfatiza e o que o post de developer pula. Toda essa estrutura roda em cima de objetos do banco. O SDK abstrai, mas o DBA precisa saber.

### 1. Schema dedicado é regra, não opção

Crie um schema separado para a memória dos agentes. Não misture com schema de aplicação. Quota controlada, tablespace dedicada, sem QUOTA UNLIMITED ON USERS. Exemplo conservador:

```text
CREATE TABLESPACE TBS_AGENT_MEMORY
  DATAFILE '+DATA' SIZE 10G AUTOEXTEND ON NEXT 1G MAXSIZE 100G
  EXTENT MANAGEMENT LOCAL AUTOALLOCATE
  SEGMENT SPACE MANAGEMENT AUTO;

CREATE USER agent_memory_owner IDENTIFIED BY "<senha_forte>"
  DEFAULT TABLESPACE TBS_AGENT_MEMORY
  TEMPORARY TABLESPACE TEMP
  QUOTA 50G ON TBS_AGENT_MEMORY;

GRANT CREATE SESSION, CREATE TABLE, CREATE SEQUENCE,
      CREATE INDEX, CREATE PROCEDURE TO agent_memory_owner;
```

**Nota prática validada em lab:** `CREATE INDEX` não é privilégio de sistema válido em Oracle 26ai (causa `ORA-00990`). Use os roles `CONNECT, RESOURCE` mais `UNLIMITED TABLESPACE` que cobrem tudo que o SDK precisa, ou liste só os privilégios válidos: `CREATE SESSION, CREATE TABLE, CREATE SEQUENCE, CREATE PROCEDURE`.

A primeira execução do SDK contra esse schema vai materializar as estruturas necessárias. Faça em ambiente de homologação primeiro, mapeie todos os objetos criados e revise antes de levar para produção.

### 2. Crescimento descontrolado é o risco número um

Memória de agente acumula se ninguém configurar política de retenção. Cada interação vira mensagem, vira embedding, vira eventualmente memória extraída pelo LLM. Multiplique por número de usuários, sessões e agentes ativos no ambiente.

Monitore desde o dia um:

```text
SELECT segment_name, segment_type,
       ROUND(bytes/1024/1024/1024, 2) AS gb
FROM   dba_segments
WHERE  owner = 'AGENT_MEMORY_OWNER'
ORDER  BY bytes DESC;
```

E acompanhe a curva de crescimento por semana, não só o tamanho atual.

### 3. Governança via VPD para multi-tenant

Se o mesmo schema atende múltiplos clientes, aplicações ou tenants, VPD é a opção a considerar. O exemplo abaixo aplica DBMS_RLS no padrão genérico de Oracle, não é configuração específica do SDK Agent Memory:

```text
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'AGENT_MEMORY_OWNER',
    object_name     => 'AGENT_MEMORY',
    policy_name     => 'TENANT_ISOLATION',
    function_schema => 'AGENT_MEMORY_OWNER',
    policy_function => 'TENANT_ISOLATION_POLICY',
    statement_types => 'SELECT, INSERT, UPDATE, DELETE',
    update_check    => TRUE
  );
END;
/
```

A função de policy usa `SYS_CONTEXT` para retornar predicado que filtra por tenant da sessão. Sem isolamento via VPD ou via filtro explícito na aplicação, um agente que consulte memórias sem escopo pode acessar registros de outros usuários do mesmo schema.

### 4. Ciclo de vida via DBMS_SCHEDULER

Memória sem expiração é dívida técnica. Configure jobs de purge desde o início:

Os nomes exatos das tabelas materializadas pelo SDK precisam ser confirmados no schema antes de qualquer DELETE. Em lab Oracle 26ai 23.26.1.0.0 a tabela de mensagens é `MESSAGE`, mas a estrutura pode mudar entre patches. Mapeie os nomes via `SELECT table_name FROM user_tables` antes de criar qualquer job de purge, e valide a existência das colunas antes de filtrar por elas.

Detalhe importante: política de retenção precisa de aprovação do dono da aplicação, não do DBA sozinho. Apagar memória pode quebrar comportamento esperado do agente. Documente, aprove, rode.

### 5. Backup e RPO

Memória de agente é dado de produção. Entra no plano de backup como qualquer outro schema crítico. RMAN normalmente, retention policy alinhada com o resto. Se a aplicação consumindo o agente é tier 1, o schema de memória também é. Se a memória inclui informação que o usuário ditou para o agente, pense em LGPD antes de definir período de retenção do backup.

## Modelos compatíveis na versão 26.4

A doc oficial publica a lista exata de LLMs e modelos de embedding suportados em abril de 2026. A escolha errada quebra a inicialização do SDK com erro de modelo não suportado. A lista evolui rápido entre patches, então sempre confirme contra a doc da versão exata instalada.

### LLMs suportados

| Provider | Modelo |
| --- | --- |
| Anthropic | `anthropic/claude-opus-4-7` |
| Anthropic | `anthropic/claude-opus-4-6` |
| Anthropic | `anthropic/claude-sonnet-4-6` |
| Anthropic | `anthropic/claude-haiku-4-5` |
| OpenAI | `openai/gpt-4.1` |
| OpenAI | `openai/gpt-4.1-mini` |
| OpenAI | `openai/gpt-4o` |
| OpenAI | `openai/gpt-5` |
| OpenAI | `openai/gpt-5-mini` |
| OpenAI | `openai/gpt-5.1` |
| OpenAI | `openai/gpt-5.2` |
| OpenAI | `openai/gpt-5.4` |
| OpenAI | `openai/gpt-5.4-mini` |
| Gemini | `gemini/gemini-3.1-flash-lite-preview` |
| Gemini | `gemini/gemini-3-flash-preview` |
| Gemini | `gemini/gemini-3.1-pro-preview` |
| OCI Gemini | `oci/google.gemini-2.5-flash` |
| OCI Gemini | `oci/google.gemini-2.5-flash-lite` |
| OCI Gemini | `oci/google.gemini-2.5-pro` |
| OCI xAI | `oci/xai.grok-3` |
| OCI xAI | `oci/xai.grok-3-fast` |
| OCI xAI | `oci/xai.grok-3-mini` |
| OCI xAI | `oci/xai.grok-3-mini-fast` |
| OCI xAI | `oci/xai.grok-4` |
| OCI xAI | `oci/xai.grok-4-1-fast-non-reasoning` |
| OCI xAI | `oci/xai.grok-4-1-fast-reasoning` |
| OCI xAI | `oci/xai.grok-4-fast-non-reasoning` |
| OCI xAI | `oci/xai.grok-4-fast-reasoning` |
| OCI xAI | `oci/xai.grok-4.20-non-reasoning` |
| OCI xAI | `oci/xai.grok-4.20-reasoning` |
| OCI xAI | `oci/xai.grok-code-fast-1` |
| OCI Cohere | `oci/cohere.command-latest` |
| OCI Cohere | `oci/cohere.command-a-03-2025` |
| OCI Cohere | `oci/cohere.command-a-reasoning-08-2025` |
| OCI Cohere | `oci/cohere.command-a-vision-07-2025` |
| OCI Cohere | `oci/cohere.command-a-translate-08-2025` |
| OCI Cohere | `oci/cohere.command-plus-latest` |
| OCI Cohere | `oci/cohere.cohere.command-r-08-2024` |
| OCI Cohere | `oci/cohere.command-r-plus-08-2024` |
| vLLM | `openai/google/gemma-4-26B-A4B-it` |
| vLLM | `openai/openai/gpt-oss-120b` |

### Embeddings suportados

| Provider | Modelo |
| --- | --- |
| OCI Cohere | `oci/cohere.embed-english-v3.0` |
| OCI Cohere | `oci/cohere.embed-english-light-v3.0` |
| OCI Cohere | `oci/cohere.embed-multilingual-v3.0` |
| OCI Cohere | `oci/cohere.embed-v4.0` |
| OpenAI | `openai/text-embedding-3-large` |
| OpenAI | `openai/text-embedding-3-small` |
| Gemini | `gemini/gemini-embedding-001` |
| Gemini | `gemini/gemini-embedding-2-preview` |
| vLLM | `hosted_vllm/Snowflake/snowflake-arctic-embed-l-v2.0` |

Recomendação prática para ambiente em português: o modelo `oci/cohere.embed-multilingual-v3.0` tem suporte declarado a múltiplos idiomas, incluindo português, e é a opção a avaliar quando o conteúdo das memórias não é só em inglês. Validar a qualidade no contexto específico antes de adotar como padrão.

## Validação

O caminho mais prático para a primeira execução é deixar o SDK criar as tabelas. Passe `schema_policy='create_if_necessary'` na inicialização do `OracleAgentMemory` e ele materializa tudo no schema do usuário conectado:

Em lab Oracle AI Database 26ai (23.26.1.0.0) com schema `agent_memory_owner`, o SDK 26.4 cria 9 objetos automaticamente: `THREAD`, `MESSAGE`, `MEMORY`, `ACTOR_PROFILE`, `RECORD_CHUNKS`, `ORACLEAGENTMEMORY_SCHEMA_META`, mais 3 índices internos do HNSW (`VECTOR$RECORD_CHUNKS_EMBEDDING_VEC_*`) já materializados sobre a coluna vetorial. A coluna `EMBEDDING` sai como `VECTOR(8200)`, dimensionada conforme o embedder configurado.

Confirme que o SDK conectou e criou estruturas:

```text
SELECT table_name FROM dba_tables WHERE owner = 'AGENT_MEMORY_OWNER';
SELECT index_name, index_type FROM dba_indexes WHERE owner = 'AGENT_MEMORY_OWNER';
```

Confirme que o vetor está chegando:

```text
SELECT column_name, data_type
FROM   dba_tab_columns
WHERE  owner = 'AGENT_MEMORY_OWNER'
AND    data_type LIKE 'VECTOR%';
```

Teste a busca pelo SDK com escopo de usuário e verifique no banco se o registro foi gerado:

Para validar que o registro foi criado, consulte diretamente as tabelas que o SDK materializa. Em lab Oracle 26ai a tabela de threads é `THREAD`, mas confirme via `SELECT table_name FROM user_tables` antes de assumir nomes em qualquer ambiente, já que a estrutura pode mudar entre patches.

## Boas práticas

- Schema dedicado e tablespace dedicada, com quota explícita.
- VPD habilitada desde o dia um se houver mais de um tenant ou aplicação consumindo a mesma instalação.
- Política de retenção definida antes do go live, com aprovação do dono da aplicação.
- Monitoramento de crescimento por dia, não só alerta de espaço.
- Backup integrado ao plano corporativo, com retenção alinhada à LGPD.
- Job de purge auditado, com log de quantas linhas foram removidas em cada execução.
- Rotina de revisão de memória crítica em agentes operacionais, para evitar que decisão tática vire dogma.
- Testes de carga no ambiente de homologação antes de qualquer agente entrar em produção.

## Validação técnica

### Confirmado na documentação oficial

- Pacote PyPI `oracleagentmemory==26.4.0` é o canal oficial de instalação (Oracle Agent Memory 26.4 Get Started).
- Classes `OracleAgentMemory`, `Embedder`, `Llm` e `SearchScope` existem nos módulos descritos no Get Started.
- Métodos `create_thread`, `add_messages`, `add_memory` e `search` são parte da API documentada.
- Lista completa de LLMs e embeddings compatíveis em abril de 2026 está publicada no Get Started 26.4 e foi reproduzida na seção "Modelos compatíveis".
- Pré-requisito de Python 3.10 ou superior é declarado na doc oficial.
- Coluna `EMBEDDING VECTOR(8200)` e índice HNSW são materializados pelo SDK quando se usa `schema_policy='create_if_necessary'` (validado em lab Oracle AI Database 26ai 23.26.1.0.0).
- Embedder é dependência obrigatória do fluxo de `add_messages`, não apenas de `search` explícito (validado em lab).

### Pode variar por ambiente

- Estrutura exata de tabelas materializadas pela SDK pode mudar entre patches da versão 26.x.
- Tipo de índice vetorial padrão depende da configuração da instância e da versão do Oracle AI Database.
- Compatibilidade de modelo evolui rapidamente. A lista válida hoje pode estar maior amanhã. Sempre validar contra a doc da versão instalada.
- Política de retenção e backup depende de regulamentação local e do contrato com cliente.

Oracle Agent Memory transforma o banco em sistema de registro para memória de agentes de IA. Tecnicamente é um SDK Python com tabelas, vetores e políticas. Operacionalmente, é mais um schema crítico que o DBA passa a operar, com regras próprias de crescimento, governança e retenção. Os dois casos mostrados, agente operacional e agente de tuning, ilustram que a memória só vale o que vale o conhecimento que ela acumula. Quanto antes essa estrutura entra no runbook, menor o risco de descobrir o problema em produção, com agente vazando memória de usuário no prompt do próximo, ou pior, sustentando uma decisão errada porque ninguém revisou.
