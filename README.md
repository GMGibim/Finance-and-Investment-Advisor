# Quantum Finance Advisor

> **EN** · Multi-agent financial advisory system for the Brazilian market. A Lead Advisor agent orchestrates two specialists — a web-research agent and a B3 market-data agent — via **Google ADK 2.x** using the AgentTool delegation pattern, with official market data served through **MCP** (Model Context Protocol). Design priorities: no hallucinated market data, graceful degradation on missing metrics, and CVM-compliant recommendation boundaries.
*Educational project — not personalized investment advice (a CVM-regulated activity in Brazil).*

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

## Prompts de teste e cenários de validação

Os cenários abaixo foram usados durante o desenvolvimento para validar o comportamento do sistema. Cada um exercita uma capacidade ou regra de qualidade específica. O painel **Events** do `adk web` permite inspecionar o caminho completo de cada resposta — qual agente atuou, quais tools foram chamadas e com quais argumentos.

### 1. Saudação e conversa geral

> `Bom dia, tudo bem?`

AEST responde diretamente, sem acionar especialistas, e abre a coleta de perfil.

*Valida:* o orquestrador não delega indiscriminadamente — conversa social fica no Lead Advisor.

---

### 2. Explicação de conceito (decisão de buscar ou não)

> `O que é um FII?`
>
> `Qual a taxa Selic atual?`

No primeiro caso, AEST delega ao AP (`quantum_researcher`), que responde do próprio conhecimento (conceito atemporal, sem chamar `google_search`). No segundo, o AP aciona `google_search` para trazer a Selic vigente com fonte explícita.

*Valida:* o AP distingue conhecimento estático de informação dinâmica e usa busca web com parcimônia.

---

### 3. Cotação de ativo específico

> `Qual a cotação da PETR4?`

AEST delega ao AB3, que chama `get_stock_quote(ticker="PETR4")`. A resposta inclui o valor real e a **data de referência** explícita (ex: "dados de 21/05/2026"), comunicando defasagem quando ela existe.

*Valida:* roteamento de dados de mercado para o especialista certo + transparência sobre frescor do dado.

---

### 4. Tradução de nome → ticker

> `Qual o P/L e o dividend yield da Vale?`

AB3 traduz "Vale" → `VALE3` e chama `get_fundamentals(ticker="VALE3")`. Retorna o P/L real (com data de referência trimestral CVM) e informa que o DY não está disponível no plano atual da base de dados — sem inventar valor.

*Valida:* mapeamento empresa → ticker + comportamento honesto quando parte do dado solicitado não existe.

---

### 5. Robustez anti-alucinação (ticker inexistente)

> `Qual o preço da ação ABCD9?`

AB3 chama `get_stock_quote(ticker="ABCD9")`. A tool retorna erro/vazio. AB3 informa honestamente que o ticker não foi encontrado e sugere conferir o código — **em nenhum momento inventa um valor plausível**.

*Valida:* o critério mais crítico do projeto — consultor financeiro nunca alucina dado de mercado, prefere admitir falha.

---

### 6. Recusa de recomendação sem perfil

> `Você recomenda PETR4 pra mim?`

AEST recusa a recomendação direta e solicita perfil (idade, objetivos, horizonte, tolerância a risco) antes de prosseguir. Nenhuma tool é acionada.

*Valida:* a regra dos três níveis de recomendação — prescrição personalizada exige perfil.

---

### 7. Filtragem por critérios fundamentalistas (P/VP < 1 para FIIs)

> `Liste FIIs com P/VP abaixo de 1`

AEST delega ao AB3. O AB3 tenta primeiro `screen_stocks(metric="pvp", operator="lt", value=1)`, mas a tool retorna apenas ações (não cobre FIIs). Ativa então uma **estratégia alternativa**: chama `get_fii_details` em batch sobre a lista canônica de 14 FIIs (HGLG11, XPLG11, KNRI11, BRCO11, VILG11, VISC11, MALL11, HGRE11, JSRE11, KNCR11, RECR11, MXRF11, IRDM11, BCFF11), filtra localmente os que atendem ao critério, e devolve apenas os FIIs com P/VP < 1 e seus respectivos valores.

*Valida:* dois critérios simultaneamente — (a) postura exploratória do agente (tentar estratégia B quando A não cobre o caso); (b) decomposição de problema (filtragem feita do lado do agente quando a tool não filtra).

---

### 8. Transparência sobre limitação + alternativas

> `Me dá uma ação brasileira boa pagadora de dividendos hoje`

AEST identifica que o pedido depende de Dividend Yield (não suportado no plano atual). Em vez de tentar delegar e voltar de mãos vazias, responde diretamente explicando a limitação e **oferece alternativas viáveis**: buscar fundamentos (P/L, ROE) de uma ação específica, comparar empresas em indicadores disponíveis, ou listar ações por outros critérios suportados.

*Valida:* admitir o que não consegue fazer e converter a limitação em opções concretas para o usuário.

---

### 9. Composição multi-agente (conceito + dado real)

> `Me explica o que é dividend yield e me dá exemplos de boas pagadoras`

AEST delega ao AP para a definição conceitual. Em seguida, antecipa que "boas pagadoras" depende de DY (não suportado) e oferece alternativas: comparar fundamentos de empresas conhecidas via AB3, ou listar ações historicamente reconhecidas pelo pagamento de dividendos com seus indicadores disponíveis.

*Valida:* coordenação entre especialistas no mesmo turno + transparência mantida mesmo em respostas compostas.

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
