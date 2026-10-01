# Workshop: Arquitetura AI em Ambiente Enterprise

> Documento de referência vivo. Parte 1 — Conceitos base e anatomia de um agente.
> Próxima parte: Arquitetura multi-agente.

**Última atualização:** 2026-10-01

---

## Índice

1. [Conceitos base (modelo canónico)](#1-conceitos-base-modelo-canónico)
2. [Anatomia de um agente — estrutura do repositório](#2-anatomia-de-um-agente--estrutura-do-repositório)
3. [config.yaml — identidade e configuração](#3-configyaml--identidade-e-configuração)
4. [agent.py — o ciclo de orquestração](#4-agentpy--o-ciclo-de-orquestração)
5. [Tools — ferramentas](#5-tools--ferramentas)
6. [Contexto — RAG, agentic search e MCP](#6-contexto--rag-agentic-search-e-mcp)
7. [Memória](#7-memória)
8. [Observabilidade e evals](#8-observabilidade-e-evals)
9. [Documentação e empacotamento](#9-documentação-e-empacotamento)
10. [Próximos temas](#10-próximos-temas)

---

## 1. Conceitos base (modelo canónico)

| Conceito | Definição | Nota enterprise |
|---|---|---|
| **Agente** | Sistema onde um modelo de linguagem usa ferramentas num ciclo para atingir um objetivo. Decide os passos, invoca ferramentas, observa resultados e repete até terminar. | Tem autonomia sobre o caminho, não só sobre a resposta. |
| **Workflow (fluxo agentic)** | Os passos estão pré-definidos pelo programador; o modelo preenche partes do fluxo. | Mais previsível e auditável — é o que mais se usa em enterprise. |
| **Agente autónomo** | O próprio modelo dirige o fluxo em tempo de execução. | Mais flexível, mais difícil de governar. |
| **Agente orquestrador** | Agente que recebe o pedido, decompõe-o, delega em sub-agentes especializados e junta os resultados (padrão *orchestrator-workers*). | Útil quando as subtarefas não são conhecidas à partida. |
| **MCP server** | Servidor que implementa o *Model Context Protocol*, protocolo aberto publicado pela Anthropic. Interface normalizada entre o modelo e ferramentas, dados e sistemas. | Constrói-se uma vez, reutiliza-se em todos os agentes. |
| **Markdown (.md)** | Formato de texto simples com marcação leve (`#` títulos, `**negrito**`, `-` listas). Trocadilho com *markup*. | Legível em bruto e formatado; versiona muito bem em Git. |

---

## 2. Anatomia de um agente — estrutura do repositório

Objetivo: um repositório versionável e *forkable*, onde personalidade, capacidades e lógica estão separadas.

```
meu-agente/
├── README.md
├── requirements.txt          # ou pyproject.toml (+ lock)
├── .env.example
├── Dockerfile
├── CHANGELOG.md
├── LICENSE
├── CONTRIBUTING.md
├── SECURITY.md
├── .github/workflows/        # CI: testes, evals, scan de segredos
├── agent/
│   ├── config.yaml
│   ├── agent.py
│   ├── prompts/
│   │   └── system.md
│   ├── tools/
│   │   ├── __init__.py       # registo das ferramentas
│   │   ├── search.py
│   │   └── database.py
│   ├── context/              # RAG, fontes de dados, servidores MCP
│   └── memory/               # estado persistente (se existir)
└── evals/
    ├── golden_dataset.jsonl
    ├── run_evals.py
    └── rubrics/
```

| Ficheiro / Pasta | Descrição |
|---|---|
| `README.md` | Visão geral do agente e instruções de setup |
| `requirements.txt` | Dependências do projeto |
| `.env.example` | Modelo de variáveis de ambiente, sem segredos |
| `agent/config.yaml` | Identidade e configuração: modelo, prompt, tools, ciclo |
| `agent/agent.py` | Orquestração — o ciclo de decisão do agente |
| `agent/tools/` | Uma ferramenta por ficheiro, mais o registo delas |
| `agent/context/` | Configuração de RAG, fontes de dados e servidores MCP |
| `agent/memory/` | Lógica de estado persistente, se existir |
| `agent/prompts/` | System prompts separados do código |
| `evals/` | Casos de teste que validam o comportamento |

---

## 3. config.yaml — identidade e configuração

Tudo o que se quer poder mudar **sem tocar no código**. Declarativo, legível, **sem segredos**.

| Bloco | Conteúdo |
|---|---|
| **Modelo** | Nome do modelo e parâmetros (temperatura, máximo de tokens…) |
| **System prompt** | Diretamente ou, preferencialmente, referência a um ficheiro em `prompts/` |
| **Ferramentas** | Lista de tools ativas, pelo nome — liga/desliga capacidades |
| **Contexto** | Fontes de RAG e servidores MCP consumidos |
| **Comportamento do ciclo** | Máximo de iterações, timeouts, estratégia em caso de falha |
| **Metadados** | Nome, versão, descrição do agente |

### Parâmetro: temperatura

Controla o grau de aleatoriedade com que o modelo escolhe cada palavra. **Baixa** → escolhe quase sempre a palavra mais provável (consistente, previsível). **Alta** → dá hipóteses a palavras menos prováveis (variado, criativo, mais propenso a erros).

| Temperatura | Comportamento | Uso típico em enterprise |
|---|---|---|
| 0 – 0,3 | Determinístico, consistente | Agentes com tools, extração de dados, classificação, código, compliance |
| 0,4 – 0,7 | Equilibrado | Redação de emails, resumos, apoio ao cliente |
| 0,8 – 1,0 | Criativo, variado | Brainstorming, marketing, ideação |

Notas:
- A escala depende do fornecedor (Anthropic: 0–1; outros podem ir até 2).
- Temperatura 0 não garante respostas 100% idênticas — a reprodutibilidade garante-se com evals.
- Em alguns modelos com raciocínio alargado, a temperatura é fixa ou ignorada.
- **Regra enterprise:** temperatura baixa por defeito; subir só em agentes de trabalho criativo.

Exemplo ilustrativo:

```yaml
name: agente-apoio-cliente
version: 1.2.0
description: Responde a pedidos de clientes com base no CRM e na base de conhecimento.

model:
  name: claude-opus-5-5
  temperature: 0.2
  max_tokens: 4096

prompt: prompts/system.md

tools:
  - procurar_cliente
  - obter_resumo_conta
  - criar_ticket          # escrita → requer aprovação humana

context:
  mcp_servers:
    - name: dynamics
      url: ${DYNAMICS_MCP_URL}
  rag:
    index: kb-suporte

loop:
  max_iterations: 15
  timeout_seconds: 120
  on_tool_error: return_to_model
```

---

## 4. agent.py — o ciclo de orquestração

Se o config é a identidade, o `agent.py` é o comportamento. Deve ser **genérico e magro**: a personalidade vive no config e nos prompts, a capacidade nas tools.

| Bloco | O que faz | Ponto-chave |
|---|---|---|
| **1. Arranque** | Lê o config, carrega o prompt, lê credenciais do ambiente, cria o cliente do modelo | Nada fixo no código |
| **2. Registo das ferramentas** | Monta a lista de tools ativas (nome, descrição, esquema), incluindo as de MCP | É o "menu" que o modelo vê |
| **3. Ciclo** | Chama o modelo → executa a tool pedida → devolve o resultado → repete até à resposta final | O modelo decide o caminho, o código executa |
| **4. Guarda-costas** | Limite de iterações, timeouts, erros devolvidos ao modelo, aprovação humana | Evita loops e ações indevidas |
| **5. Observabilidade** | Regista chamadas, tools usadas, tokens e tempos | Auditoria e diagnóstico |

Pseudo-código do ciclo:

```python
messages = [user_request]
for i in range(config.loop.max_iterations):
    response = model.call(system=prompt, messages=messages, tools=tools)
    if response.is_final_answer:
        return response
    for tool_call in response.tool_calls:
        if tool_call.is_sensitive:
            require_human_approval(tool_call)
        try:
            result = tools.execute(tool_call)
        except Exception as e:
            result = f"Erro: {e}"      # devolvido ao modelo, não rebenta o agente
        messages.append(result)
        tracer.log(tool_call, result)
raise MaxIterationsReached
```

---

## 5. Tools — ferramentas

Uma ferramenta tem duas metades: a **declaração** (o que o modelo vê) e a **implementação** (o código que corre).

| Elemento | O que é | Ponto-chave |
|---|---|---|
| **Nome** | Identificador curto e explícito | Ex.: `procurar_cliente` |
| **Descrição** | O que faz, quando usar, quando não usar, o que devolve | A peça mais importante — é por ela que o modelo decide |
| **Esquema de parâmetros** | JSON Schema com tipos, obrigatórios e descrição de cada campo | Evita chamadas mal formadas |
| **Implementação** | A função que executa o trabalho | Separada da declaração, testável à parte |
| **Regra 1** | Ferramentas por tarefa, não por endpoint | Menos tools, mais úteis |
| **Regra 2** | Resultados enxutos | Filtrar, resumir, paginar |
| **Regra 3** | Erros úteis | Mensagem que ajuda o modelo a corrigir (ex.: "cliente não encontrado, verifica o NIF") |
| **Regra 4** | Separar leitura de escrita | Escrita passa por aprovação humana |

Exemplo de declaração:

```json
{
  "name": "procurar_cliente",
  "description": "Procura um cliente no CRM pelo NIF ou nome. Usar quando o utilizador menciona um cliente específico. Não usar para listar todos os clientes. Devolve id, nome, segmento e estado da conta.",
  "input_schema": {
    "type": "object",
    "properties": {
      "nif":  { "type": "string", "description": "NIF do cliente, 9 dígitos" },
      "nome": { "type": "string", "description": "Nome ou parte do nome" }
    }
  }
}
```

---

## 6. Contexto — RAG, agentic search e MCP

Pergunta que esta camada responde: *como é que o agente sabe coisas que não estão no modelo?*

| Mecanismo | Quando usar | Como funciona | Ponto crítico |
|---|---|---|---|
| **Contexto estático** | Regras, glossário, políticas curtas | Vai sempre no prompt | Tem de ser pequeno |
| **RAG** | Grandes volumes de documentação | Indexação (chunks → embeddings → base vetorial) e consulta (pergunta → pedaços mais parecidos → prompt) | Chunking, pesquisa híbrida, permissões |
| **Agentic search** | Alternativa ao RAG com modelos de contexto grande | O agente pesquisa iterativamente com uma tool de busca | Menos infraestrutura, mais chamadas |
| **MCP** | Sistemas vivos (Dynamics, Jira, BD) | Servidor expõe *tools*, *resources* e *prompts* | Construído uma vez, reutilizado por todos os agentes |

**Pontos críticos do RAG em enterprise:**
- **Chunking** — cortar mal um documento destrói o significado.
- **Pesquisa híbrida** — semântica + palavras-chave; essencial para códigos, referências e nomes próprios.
- **Permissões** — se o utilizador não pode ver um documento, o agente também não o pode recuperar para ele. As permissões viajam com os dados.

**MCP expõe três tipos de primitivas:**
- **Tools** — ações que o modelo pode invocar.
- **Resources** — dados que podem ser lidos (ficheiros, registos).
- **Prompts** — modelos de instruções reutilizáveis.

---

## 7. Memória

| Tipo | O que guarda | Duração | Técnica / cuidado |
|---|---|---|---|
| **Curto prazo** | Histórico da conversa e resultados das tools | Durante a tarefa | Truncar, resumir, limpar resultados de tools |
| **Trabalho (notas)** | Estado da tarefa: feito, em falta, decisões | Tarefas longas, retomas | Ficheiro ou estrutura que o agente atualiza |
| **Longo prazo — episódica** | O que aconteceu, casos passados | Entre sessões | Recuperada quando relevante |
| **Longo prazo — semântica** | Factos e preferências | Entre sessões | Rever e expirar |
| **Longo prazo — procedimental** | Como fazer as coisas | Entre sessões | Muitas vezes vira prompts ou skills |
| **Cuidados enterprise** | RGPD, isolamento, higiene | Transversal | Começar sem memória de longo prazo |

**Cuidados obrigatórios em enterprise:**
- **Privacidade / RGPD** — o que se guarda, durante quanto tempo, direito ao esquecimento.
- **Isolamento** — a memória de um utilizador ou cliente nunca se mistura com a de outro.
- **Higiene** — memórias desatualizadas contaminam decisões; é preciso rever, corrigir e expirar.

> Regra prática: começar **sem** memória de longo prazo e só a introduzir quando houver um caso claro.

---

## 8. Observabilidade e evals

Observabilidade responde a *"o que aconteceu?"*; evals respondem a *"está correto?"*.

| Elemento | O que é | Ponto-chave |
|---|---|---|
| **Tracing** | Cada execução é um *trace*, cada passo um *span* (entrada, saída, tempo, tokens) | Reconstruir o raciocínio do agente |
| **Métricas** | Custo, latência, fiabilidade, qualidade | Qualidade é medida pelos evals |
| **Stack** | OpenTelemetry + Langfuse / LangSmith / Arize, ou Datadog / Azure Monitor | Aproveitar o que a empresa já usa |
| **Eval por código** | Verificações determinísticas: formato, tool certa, valores | Rápido e barato, usar sempre que possível |
| **Eval por modelo** | LLM-as-judge com grelha de critérios | Calibrar contra avaliação humana |
| **Eval humana** | Especialistas reveem amostras | A referência que valida as outras |
| **Golden dataset** | 20–30 casos reais, incluindo difíceis e falhas passadas | Correr em cada mudança, idealmente no CI |
| **Trajetória** | Avaliar o caminho, não só o resultado | Apanha acertos por acaso e passos indevidos |

---

## 9. Documentação e empacotamento

O que transforma o agente num ativo reutilizável e *forkable*.

| Elemento | Conteúdo | Ponto-chave |
|---|---|---|
| **README.md** | O quê, requisitos, instalação, configuração, evals | A funcionar em 15 minutos |
| **Dependências + lock** | Versões exatas | Evita quebras silenciosas |
| **Dockerfile** | Ambiente contentorizado | Corre igual em todo o lado |
| **CI/CD** | Testes, evals, verificação de segredos | Em cada pull request |
| **Versionamento + CHANGELOG** | SemVer | Mudança de prompt é mudança de versão |
| **LICENSE, CONTRIBUTING, SECURITY** | Governação do repositório | Define uso, contribuição e reporte |
| **Template repo** | Cópia limpa para novos projetos | Personalizar só config, prompts e tools |

**O README deve responder, por esta ordem:**
1. O que é o agente e que problema resolve.
2. O que precisa para correr (chaves de API, servidores MCP).
3. Como instalar e arrancar em poucos comandos.
4. Como configurar (`config.yaml`, `.env.example`).
5. Como correr os evals.

---

## 10. Próximos temas

- **Arquitetura multi-agente** — quando vale a pena ter vários agentes em vez de um; padrões (orchestrator-workers, pipeline, hand-off, avaliador-otimizador).
- Governança e segurança em enterprise.
- Integração com sistemas existentes (Dynamics, ESB, APIs).
