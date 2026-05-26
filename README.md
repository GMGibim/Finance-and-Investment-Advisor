# Quantum Finance Advisor

Sistema multi-agente de consultoria financeira para o mercado brasileiro, construído com o **Google Agent Development Kit (ADK 2.x)** e integrado a dados oficiais da B3 via **MCP (Model Context Protocol)**.

O projeto demonstra um padrão arquitetural de orquestração — um agente coordenador (Lead Advisor) delega tarefas a especialistas internos, consolida as respostas e responde ao cliente final com transparência sobre o que pode e o que não pode entregar.

---

## Arquitetura

```
                                ┌─────────────────────────────────┐
   Cliente  ──►  Prompt  ──►    │  AEST (Lead Advisor)            │  ──►  Resposta final
                                │  Orquestrador                   │
                                └────────────────┬────────────────┘
                                                 │
                                  delega ────────┴──────── delega
                                       │                   │
                              ┌────────▼─────┐    ┌────────▼─────┐
                              │   AP         │    │   AB3        │
                              │  Pesquisador │    │  Dados B3    │
                              └────────┬─────┘    └────────┬─────┘
                                       │                   │
                                       ▼                   ▼
                            Google Search (web)        MCP → B3/CVM/BCB
```

### Os três agentes

| Agente | Papel | Ferramentas |
|---|---|---|
| **AEST** (Lead Advisor) | Atende o cliente, coleta perfil, decide quem chamar, consolida respostas. Mantém o controle da conversa o tempo todo. | `AgentTool(AP)`, `AgentTool(AB3)` |
| **AP** (Agente Pesquisador) | Explica conceitos e produtos do mercado brasileiro (renda fixa, FIIs, tributação, indicadores macro). Usa busca web para informações dinâmicas. | `google_search` |
| **AB3** (Agente de Dados B3) | Traz dados oficiais de ações e FIIs específicos. Filtra ativos por critérios fundamentalistas. | 10 tools MCP de dados de mercado |

### Decisão arquitetural: `AgentTool` em vez de `sub_agents`

O ADK oferece dois paradigmas multi-agente:

- **`sub_agents`** — transferência de controle (handoff). O subagente vira o agente ativo.
- **`AgentTool`** — delegação como ferramenta. O orquestrador permanece no controle.

Este projeto usa **`AgentTool`** por duas razões:

1. **Semântica correta para Lead Advisor:** o AEST precisa consolidar respostas e manter relação com o cliente, não passar a bola.
2. **Compatibilidade técnica:** o Gemini API não permite combinar built-in tools (como `google_search` no AP) com function calling na mesma chamada — restrição que `AgentTool` resolve por isolar a execução de cada subagente.

---

## Stack

- **LLM:** Gemini 2.5 Flash (Google AI Studio)
- **Framework de agentes:** [Google Agent Development Kit (ADK) 2.x](https://google.github.io/adk-docs/)
- **Dados de mercado:** [Bolsai](https://usebolsai.com) via servidor MCP local (`uvx bolsai-mcp`)
- **Linguagem:** Python 3.10+
- **Runtime:** `adk web` (interface de desenvolvimento embutida)

---

## Estrutura do projeto

```
quantum-finance-advisor/
├── .env                                  # GOOGLE_API_KEY + BOLSAI_API_KEY
├── .gitignore
├── README.md
└── quantum_advisor/
    ├── __init__.py
    ├── agent.py                          # AEST (root_agent)
    └── sub_agents/
        ├── __init__.py
        ├── researcher/
        │   ├── __init__.py
        │   └── agent.py                  # AP (researcher_agent)
        └── b3_data/
            ├── __init__.py
            └── agent.py                  # AB3 (b3_data_agent)
```

---

## Setup

### Pré-requisitos

- Python 3.10+
- Conta no [Google AI Studio](https://aistudio.google.com/apikey) para a chave da Gemini API
- Conta no [Bolsai](https://usebolsai.com) para a chave do MCP de dados da B3

### Instalação

```bash
git clone <repo-url>
cd quantum-finance-advisor

python -m venv .venv
# Linux/Mac:
source .venv/bin/activate
# Windows:
.venv\Scripts\Activate.ps1

pip install --upgrade pip
pip install "google-adk[mcp]" uv
```

### Configuração

Crie um arquivo `.env` na raiz com:

```dotenv
GOOGLE_GENAI_USE_VERTEXAI=FALSE
GOOGLE_API_KEY=sua_chave_gemini
BOLSAI_API_KEY=sua_chave_bolsai
```

### Execução

```bash
adk web
```

Acesse `http://127.0.0.1:8000` e selecione `quantum_advisor` no dropdown.

---

## Princípios de design

### 1. Engenharia no `instruction`, não no código

O comportamento crítico do sistema (anti-alucinação, roteamento entre especialistas, transparência sobre limitações) está expresso em linguagem natural nos `instruction` de cada agente. O código Python ao redor é mínimo. Esse é um padrão central do ADK e do desenvolvimento de agentes em geral.

### 2. Confiabilidade > completude

O sistema prefere **admitir limitação** a **improvisar dado**. Quando uma ferramenta não tem o dado solicitado (ex: Dividend Yield no plano gratuito), o agente informa explicitamente e oferece alternativas viáveis em vez de retornar valor inventado ou estimativa.

### 3. Três níveis de "recomendação"

O AEST distingue:

- **Nível 1 — Explicar critérios:** sempre permitido. Ex: "o que olhar num bom FII?"
- **Nível 2 — Filtrar e exemplificar:** permitido quando solicitado. Ex: "liste FIIs com P/VP < 1". Apresentação como "ativos que atendem aos critérios", não como prescrição.
- **Nível 3 — Recomendação prescritiva personalizada:** recusado. Atividade regulada pela CVM, requer consultor certificado.

Essa distinção evita tanto a falha por excesso de cautela (recusar dar exemplos quando o usuário queria) quanto a falha por excesso de assertividade (recomendar sem perfil).

### 4. Postura exploratória dos especialistas

O AB3 tem instrução explícita de **testar a ferramenta** antes de declarar limitação, e de **tentar estratégias alternativas** quando a primeira falha. Ex: se `screen_stocks` não cobre FIIs, partir para batch de `get_fii_details` em lista canônica.

### 5. Transparência genérica ao usuário

O cliente nunca vê nomes de fornecedores, APIs ou agentes internos. Limitações são comunicadas como "nossa base de dados atual" e "esta versão inicial", mantendo a abstração do produto.

---

## Capacidades e limitações

### O que o sistema faz bem

- **Conceitos e educação financeira** (via AP + Google Search): produtos de renda fixa e variável, tributação, indicadores macro atuais.
- **Dados de ações específicas** (via AB3 + MCP): cotação, P/L, P/VP, ROE, EV/EBITDA, margens, comparação multi-ticker, demonstrações financeiras (CVM).
- **Dados de FIIs específicos:** preço, valor patrimonial (NAV), P/VP.
- **Filtragem de ações** por métricas fundamentalistas (P/L, ROE, margens, etc.).
- **Filtragem de FIIs por P/VP** (via batch de chamadas individuais sobre lista canônica).
- **Indicadores macro:** Selic, IPCA, CDI, USD/BRL atualizados.
- **Coleta de perfil** antes de recomendação personalizada.
- **Tradução de nome de empresa para ticker** (ex: "Petrobras" → PETR4).
- **Disclaimers automáticos** em respostas que envolvem ativos específicos.

### Limitações conhecidas (plano gratuito da base de dados)

- Dividend Yield (DY) de ações e FIIs não disponível.
- Histórico de dividendos / proventos não disponível.
- Métricas operacionais de FIIs (vacância, taxa de ocupação) não estruturadas.
- Classificação automática de FIIs em tijolo/papel/híbrido não fornecida pela API — o sistema usa uma lista canônica curada como ponto de partida.
- Cotações de "preço atual" têm defasagem de 1+ dia útil (free tier típico de dados B3).

Essas limitações são comunicadas ao usuário com sugestões de alternativas viáveis.

---

## Critérios de qualidade validados

| Critério | Status |
|---|---|
| Não inventar cotações ou indicadores | ✅ |
| Roteamento correto entre especialistas | ✅ |
| Coleta de perfil antes de recomendação personalizada | ✅ |
| Transparência sobre data de referência dos dados | ✅ |
| Tradução automática nome de empresa → ticker | ✅ |
| Recusa de tickers inexistentes (sem fallback alucinatório) | ✅ |
| Filtragem objetiva por critérios disponíveis | ✅ |
| Múltiplas chamadas de ferramenta no mesmo turno | ✅ |
| Comunicação transparente de limitações + alternativas | ✅ |
| Disclaimer regulatório em recomendações | ✅ |

---

## Próximos passos (roadmap)

- **Persistência de perfil:** armazenar o perfil do cliente entre sessões (atualmente, ele é coletado a cada nova conversa).
- **Testes automatizados:** suite com `pytest` cobrindo cenários críticos (anti-alucinação, roteamento, casos de limitação).
- **Tratamento explícito de rate limits:** retry com backoff e mensagens claras quando o Bolsai ou o Gemini API atingem cota.
- **Migração para Vertex AI:** trocar Gemini API por Vertex para uso em produção (apenas mudança de variáveis de ambiente, código não muda).
- **Plano Pro do Bolsai:** destravar Dividend Yield, histórico de dividendos e classificação de FIIs.
- **Fonte secundária com cross-check:** validação cruzada de cotações entre Bolsai e fonte alternativa.

---

## Aviso legal

Este sistema é uma orientação educacional e **não substitui** a análise de um consultor financeiro certificado. Não constitui recomendação de investimento personalizada, atividade regulada pela CVM e que exige profissional habilitado.

---

## Referências

- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [Google ADK Samples — Financial Advisor](https://github.com/google/adk-samples/tree/main/python/agents/financial-advisor)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Bolsai — MCP server para B3](https://usebolsai.com)
- [Developer's guide to multi-agent patterns in ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
