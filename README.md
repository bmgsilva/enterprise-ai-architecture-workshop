# Workshop: Arquitetura AI em Ambiente Enterprise

Documento de referência vivo sobre arquitetura de sistemas de IA em contexto enterprise: conceitos, anatomia de agentes, padrões multi-agente, governança e integração.

## Conteúdo

| # | Parte | Estado |
|---|---|---|
| 1 | [Conceitos base e anatomia de um agente](docs/01-conceitos-e-anatomia-agente.md) | ✅ Concluída |
| 2 | [Arquitetura multi-agente](docs/02-arquitetura-multi-agente.md) | ✅ Concluída |
| 3 | [Governança e segurança em enterprise](docs/03-governanca-e-seguranca.md) | 🚧 Em curso |
| 4 | Integração com sistemas existentes (Dynamics, ESB, APIs) | 📝 Planeada |

## Parte 1 — resumo

- **Conceitos base:** agente, workflow vs. agente autónomo, agente orquestrador, MCP server.
- **Anatomia de um agente:** estrutura do repositório, `config.yaml`, `agent.py`, tools, contexto (RAG, agentic search, MCP), memória, observabilidade e evals, documentação e empacotamento.

## Parte 2 — resumo

- **Quando dividir:** contexto, paralelismo, especialização, fronteiras de segurança; na dúvida, um agente só.
- **Padrões:** orquestrador-trabalhadores, pipeline, hand-off, avaliador-otimizador, rede.
- **Comunicação:** mensagens, estado partilhado, artefactos por referência; MCP (vertical) vs A2A (horizontal).
- **Frameworks e plataformas:** Claude Agent SDK, LangGraph, Microsoft Agent Framework, Microsoft Foundry, Claude Managed Agents.

## Parte 3 — resumo (em curso)

- **Modelo de ameaças:** tríade letal e OWASP Top 10 para aplicações agentic.
- **Controlos técnicos:** sete camadas de defesa; o modelo propõe, o código decide; níveis de risco das ações.
- **Governança:** inventário de agentes, papéis, ciclo de vida, comité proporcional ao risco.
- **Regulação europeia:** próximo bloco.

Os documentos incluem diagramas em Mermaid, desenhados automaticamente pelo GitHub.

## Versões

O mesmo conteúdo é mantido no Notion, em espelho deste repositório.
