# SyntonIA — Agente de Recomendação Musical (Grupo 8 — ResIA)

[![Licença: MIT](https://img.shields.io/badge/Licença-MIT-yellow.svg)](LICENSE)
[![Python 3.12](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](requirements.txt)
[![Testes do agente](https://github.com/Lucas-AV/SyntonIA/actions/workflows/agente-tests.yml/badge.svg)](.github/workflows/agente-tests.yml)
[![GitHub Pages](https://github.com/Lucas-AV/SyntonIA/actions/workflows/pages.yml/badge.svg)](.github/workflows/pages.yml)

Projeto da disciplina ResIA (Grupo 8): um **agente conversacional de
recomendação de músicas** que conversa em linguagem natural, entende o
pedido do usuário (gênero, humor, energia, artista de referência...),
personaliza a recomendação com o histórico real do Spotify de quem está
logado e permite salvar o resultado direto numa playlist do Spotify. O
projeto nasceu de uma análise exploratória de um dataset de faixas do
Spotify e evoluiu para um produto completo: backend + frontend do agente,
motor de recomendação por similaridade, integração OAuth com o Spotify,
IA generativa local (Ollama) com alternativa hospedada (Claude), e um
dashboard público com as análises de dados e o pitch do projeto.

> **About (EN):** ResIA (Grupo 8) course project — a conversational music
> recommendation agent. It understands natural-language requests (genre,
> mood, energy, reference artist...), personalizes results using the
> user's real Spotify listening history via OAuth, and can save the
> recommendation as a real Spotify playlist. Built on top of a content-based
> recommendation engine (cosine similarity over Spotify audio features),
> with local LLM inference (Ollama) and an optional hosted backend
> (Claude).

**Site publicado (dashboard + landing):** https://lucas-av.github.io/SyntonIA/
**Quadro do Miro:** [Link do projeto](https://miro.com/app/board/uXjVHttBzWA=/)
**Quadro do JIRA:** Link do Quadro

## Sumário

- [O que o projeto faz](#o-que-o-projeto-faz)
- [Como funciona (arquitetura resumida)](#como-funciona-arquitetura-resumida)
- [Tecnologias](#tecnologias)
- [Integrações externas](#integrações-externas)
- [Como rodar](#como-rodar)
  - [Agente conversacional (produto principal)](#agente-conversacional-produto-principal)
  - [Explorador Spotify (ferramenta de dev)](#explorador-spotify-ferramenta-de-dev)
  - [Análise exploratória e dashboard](#análise-exploratória-e-dashboard)
  - [Análise de mercado (Julia)](#análise-de-mercado-julia)
- [Testes e CI/CD](#testes-e-cicd)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Base de dados](#base-de-dados)
- [Análises disponíveis no dashboard](#análises-disponíveis-no-dashboard)
- [Documentação e apresentação](#documentação-e-apresentação)
- [Equipe](#equipe)
- [Roadmap](#roadmap)
- [Licença](#licença)

## O que o projeto faz

O produto é o **agente conversacional** (pasta [`agente_conversacional/`](agente_conversacional/)):

- **Chat em linguagem natural** — pede recomendação por texto livre
  ("quero um pagode animado", "algo triste pra relaxar") e recebe faixas
  reais com nome, artista e capa.
- **Preview de áudio de 30s** em cada faixa, com fallback automático pro
  YouTube quando o Spotify não fornece prévia.
- **Login com Spotify** (OAuth PKCE) — inclusive **por QR code**, pra
  cenário de demo/kiosk (escaneia com o celular e autoriza sem digitar
  nada no computador da apresentação).
- **Personalização real**: com o usuário logado, o agente monta um
  "perfil de gosto" a partir do histórico de escuta do Spotify (top
  tracks, faixas curtidas, tocadas recentemente) e usa isso pra enviesar
  a recomendação.
- **Painel "Meu Perfil"** com os dados da conta Spotify conectada.
- **Painel "Explorar Spotify"** — busca, lançamentos recentes, minhas
  playlists, quem eu sigo, meus dados de audição e um player com
  controles reais (play/pause/próxima/anterior/volume/shuffle/repeat/fila).
- **Salvar no Spotify** — cria de verdade uma playlist na conta do usuário
  com as faixas recomendadas.
- **"Gerar outra recomendação"**, sem repetir faixas já mostradas na
  sessão; **dark mode**; landing page do projeto.
- Endpoint `GET /recomendar` pra pedir recomendação direto, sem passar
  pelo LLM (útil pra demo/debug).

O repositório também guarda o trabalho que fundamentou o produto: a
análise exploratória do dataset de faixas, a análise de mercado de
streaming (pitch de investimento) e um dashboard público com tudo isso.

## Como funciona (arquitetura resumida)

Um turno de conversa passa por um pipeline em etapas — cada etapa é
determinística sempre que possível, e só usa o modelo de linguagem (LLM)
onde regra fixa não resolve:

```
mensagem do usuário
   -> roteador determinístico (regex; resolve pedidos simples direto)
   -> extração estruturada via LLM (pedidos livres/longos)
   -> validação de schema (gênero/artista contra o dataset real)
   -> busca no motor de recomendação (similaridade por cosseno,
      + perfil de gosto do usuário logado, blend 70/30)
   -> geração da resposta guiada por LLM (ou template determinístico
      de fallback, se o LLM falhar/estiver indisponível)
   -> auditoria mecânica: filtra qualquer faixa citada pelo LLM que
      não veio de fato da busca
```

Especificação técnica completa (ciclo de vida do agente, fluxo OAuth,
contratos entre componentes, casos de uso e edge cases):
[`docs/PIPELINE_AGENTE_PROPOSTA_B.md`](docs/PIPELINE_AGENTE_PROPOSTA_B.md).
Backlog em épicos/tickets: [`docs/BACKLOG_JIRA_PROPOSTA_B.md`](docs/BACKLOG_JIRA_PROPOSTA_B.md).

## Tecnologias

| Camada | Tecnologia | Onde |
|---|---|---|
| Backend do agente | Python 3.12, **FastAPI**, uvicorn | [`agente_conversacional/`](agente_conversacional/) |
| Frontend do agente | HTML/CSS/JS puro (sem framework/bundler, deliberado) | [`agente_conversacional/frontend/`](agente_conversacional/frontend/) |
| Motor de recomendação | pandas + NumPy — índice de similaridade por cosseno sobre 9 features de áudio normalizadas | `agente_conversacional/recomendacao/` |
| IA generativa | **Ollama** (local, `qwen2.5:7b-instruct-q4_K_M`) por padrão; **Claude/Anthropic** como backend hospedado alternativo | `agente_conversacional/llm/` |
| Autenticação/tokens | OAuth 2.0 Authorization Code + PKCE; tokens criptografados (`cryptography.Fernet`) em SQLite | `agente_conversacional/spotify_auth/` |
| Persistência | SQLite local (sessões de tokens); sessões de conversa em memória | `agente_conversacional/` |
| Ferramenta de dev | Flask (backend) + **Vue 3** + Vite (frontend) — explorador da Web API do Spotify | [`spotify_explorer/`](spotify_explorer/) |
| Análise de dados / EDA | pandas, matplotlib, adjustText, JupyterLab | raiz do repo, `scripts/`, `analise_exploratoria.ipynb` |
| Site / dashboard | Jinja2 (gera HTML estático), publicado via GitHub Pages | [`site/`](site/) |
| Análise de mercado | **Julia** (`CSV.jl`, `DataFrames.jl`, `Plots.jl`) — stack isolada de propósito | [`analise_mercado_streaming/`](analise_mercado_streaming/) |
| Testes | pytest (backend do agente: 303 testes; explorer; EDA/site) | `agente_conversacional/`, `spotify_explorer/`, `tests/` |
| CI/CD | GitHub Actions — testes do agente, validação do notebook, deploy do GitHub Pages | [`.github/workflows/`](.github/workflows/) |

Não há banco de dados externo nem Docker/docker-compose — tudo roda
localmente com SQLite e os servidores de desenvolvimento de cada stack.

## Integrações externas

| Integração | Uso | Variáveis de ambiente |
|---|---|---|
| **Spotify Web API** (OAuth PKCE) | Login do usuário, histórico de escuta, perfil de gosto, criar playlist, painel "Explorar Spotify", player | `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, `SPOTIFY_TOKEN_ENCRYPTION_KEY`, `SPOTIFY_TOKEN_DB_PATH` (em [`agente_conversacional/.env.example`](agente_conversacional/.env.example)) |
| **Spotify Web API** (Client Credentials) | Fallback de busca quando o dataset local não cobre o gênero/artista pedido, sem exigir login | mesma app do Spotify acima |
| **Ollama** (LLM local) | Backend padrão de IA generativa — roteamento, extração e geração de texto | `LLM_BACKEND=ollama`, `OLLAMA_BASE_URL`, `OLLAMA_MODEL`, `OLLAMA_NUM_CTX`, `OLLAMA_NUM_PREDICT` |
| **Claude / Anthropic API** | Backend de IA generativa alternativo (hospedado, sem depender de GPU local) | `LLM_BACKEND=claude`, `ANTHROPIC_API_KEY`, `CLAUDE_MODEL` |
| **YouTube Data API v3** | Fallback de preview de áudio quando o Spotify não devolve `preview_url` | `YOUTUBE_API_KEY` |

Registre o app no [Spotify Developer Dashboard](https://developer.spotify.com/dashboard)
pra obter as credenciais do Spotify. As chaves de LLM/YouTube são
opcionais — sem elas, o agente funciona com Ollama local e sem preview
via YouTube.

## Como rodar

### Agente conversacional (produto principal)

Pré-requisito pro backend de LLM local: [Ollama](https://ollama.com) instalado.

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M

cd agente_conversacional
pip install -r requirements.txt
cp .env.example .env      # edite SPOTIFY_*/ANTHROPIC_API_KEY/YOUTUBE_API_KEY conforme necessário
uvicorn app:app --reload  # backend em http://127.0.0.1:8000
```

Frontend (PowerShell):

```powershell
cd agente_conversacional/frontend
.\serve.ps1    # http://127.0.0.1:8080 (porta configurável: -Port)
```

Detalhes de setup, endpoints, CORS, rate limiting e status por épico:
[`agente_conversacional/README.md`](agente_conversacional/README.md).

### Explorador Spotify (ferramenta de dev)

Ferramenta interna usada para explorar/documentar a Web API do Spotify
antes de portar as funcionalidades pro produto — não faz parte do produto
final.

```bash
cd spotify_explorer
cp .env.example .env
pip install -r requirements.txt -r spotify_explorer/requirements.txt   # rodar a partir da raiz do repo
cd frontend && npm install && npm run build && cd ..
python app.py   # http://127.0.0.1:5000
```

Detalhes: [`spotify_explorer/README.md`](spotify_explorer/README.md).

### Análise exploratória e dashboard

```bash
pip install -r requirements.txt
python scripts/group_occurrences.py
python scripts/plot_genre_charts.py
python scripts/plot_genre_mode.py
python scripts/plot_popularity_occurrences.py
python scripts/profile_dataset.py
python scripts/plot_correlations.py
python site/build_site.py                     # gera site/dist/ (abrir site/dist/index.html)
jupyter lab analise_exploratoria.ipynb         # abre o notebook de EDA
```

Notebook pronto pra demonstração guiada célula a célula — veja
[`docs/NOTEBOOK_DEMO.md`](docs/NOTEBOOK_DEMO.md) ou abra direto pelo
[Binder](https://mybinder.org/v2/gh/Lucas-AV/SyntonIA/HEAD?labpath=analise_exploratoria.ipynb),
sem preparar ambiente local.

### Análise de mercado (Julia)

```bash
cd analise_mercado_streaming
julia setup.jl && julia analise_mercado.jl
```

Stack isolada de propósito (Julia, não Python) — relatório completo com
proveniência de cada número em
[`analise_mercado_streaming/RELATORIO.md`](analise_mercado_streaming/RELATORIO.md).
Versão publicada: [Mercado de Streaming](https://lucas-av.github.io/SyntonIA/mercado.html).

## Testes e CI/CD

```bash
pytest                              # raiz: EDA/site
cd agente_conversacional && pytest  # 303 testes — LLM e Spotify sempre mockados
cd spotify_explorer && pytest       # requests mockado
```

Três workflows de GitHub Actions rodam automaticamente:

- [`agente-tests.yml`](.github/workflows/agente-tests.yml) — testes do
  agente em todo push/PR que toque `agente_conversacional/`.
- [`notebook-demo.yml`](.github/workflows/notebook-demo.yml) — valida o
  notebook de demonstração em PRs que toquem notebook/scripts/tests.
- [`pages.yml`](.github/workflows/pages.yml) — build e deploy do
  dashboard estático no GitHub Pages a cada push em `main`.

## Estrutura do repositório

```
agente_conversacional/          # PRODUTO: backend FastAPI + frontend do agente
  app.py                          # entrypoint FastAPI
  chat/                           # pipeline conversacional (roteador, extração, geração, auditoria)
  recomendacao/                   # motor de recomendação (dataset, índice por cosseno, busca, perfil)
  llm/                            # abstração de LLM (backends Ollama e Claude)
  spotify_auth/                   # OAuth PKCE, tokens, histórico, playlist, painel "Explorar Spotify", QR login
  api/, sessions/                 # sessões de conversa e endpoints HTTP
  frontend/                       # HTML/CSS/JS puro (chat, Meu Perfil, Explorar Spotify)
spotify_explorer/                # ferramenta de dev: Flask + Vue/Vite pra explorar a Web API do Spotify
analise_mercado_streaming/       # análise de mercado do pitch, em Julia (stack separada)
data/                             # raw/, processed/ (dataset.csv consolidado), analytics/, hygiene/
scripts/                          # scripts Python de análise/gráficos (EDA)
site/                             # gerador do dashboard estático (build_site.py, templates/, static/)
docs/                             # spec técnica, backlog Jira, apresentação, pesquisa
tests/                            # testes pytest do nível raiz (EDA/site)
analise_exploratoria.ipynb        # notebook principal de EDA
requirements.txt                  # dependências Python do nível raiz (EDA/site)
.github/workflows/                # CI (testes do agente, notebook, deploy do Pages)
```

## Base de dados

Dataset: [Spotify Tracks Dataset](https://www.kaggle.com/datasets/maharshipandya/-spotify-tracks-dataset) (Kaggle).

`data/processed/dataset.csv` contém uma base ampliada de faixas e gêneros
(128.830 registros, 97.534 faixas únicas), gerada a partir das fontes
documentadas em `docs/data_enrichment/`.

- Identificação: `track_id`, `artists`, `album_name`, `track_name`, `track_genre`
- Popularidade: `popularity`
- Características de áudio: `danceability`, `energy`, `loudness`, `speechiness`,
  `acousticness`, `instrumentalness`, `liveness`, `valence`, `tempo`,
  `duration_ms`, `key`, `mode`, `time_signature`, `explicit`

> Limitação conhecida: o dataset não traz histórico de escuta/feedback
> por usuário, só metadados e popularidade agregada da faixa — por isso a
> personalização real do agente usa o histórico do Spotify de quem loga
> (Épico 5), e não colaboração entre usuários do dataset.

## Análises disponíveis no dashboard

Publicadas em https://lucas-av.github.io/SyntonIA/:

- **Popularidade por gênero** — gênero mais popular: *chill* (53,7 de
  popularidade média).
- **Energia × dançabilidade por gênero** — mais energético: *death-metal*
  (0,93); menos energético: *classical* (0,19); mais dançável:
  *chicago-house* (0,77).
- **Escala maior vs. menor por gênero** — maior predominância de escala
  maior: *country* (89%); de escala menor: *deep-house* (54%).
- **Popularidade × volume de catálogo do artista**.
- **Perfil geral do dataset** — contagem de faixas/artistas/álbuns/gêneros,
  duplicidade entre gêneros, distribuição por artista/álbum.
- **Correlação de Pearson** entre popularidade, duração e as 9 features
  de áudio contínuas.
- **Precisão do motor de similaridade por cosseno** e **métodos
  alternativos de recomendação** (colaborativo/híbrido) — páginas que
  documentam a escolha do método atual e o que ficaria pra uma próxima
  iteração.
- **Análise de mercado de streaming** (Julia) e o **notebook de EDA**
  embutidos no próprio dashboard.

## Documentação e apresentação

- [`docs/PIPELINE_AGENTE_PROPOSTA_B.md`](docs/PIPELINE_AGENTE_PROPOSTA_B.md) —
  especificação técnica completa do agente.
- [`docs/BACKLOG_JIRA_PROPOSTA_B.md`](docs/BACKLOG_JIRA_PROPOSTA_B.md) —
  backlog em épicos/tickets.
- [`docs/apresentacao/`](docs/apresentacao/README.md) — pitch de 5 minutos,
  apresentação técnica de 8 minutos (PPTX/PDF), roteiros e guia de ensaio
  do projeto (arquivos já renomeados pra SyntonIA; o conteúdo interno dos
  slides ainda diz MelodIA até alguém rodar o rebuild dos decks).
- [`docs/NOTEBOOK_DEMO.md`](docs/NOTEBOOK_DEMO.md) — roteiro de demonstração
  do notebook de EDA.

## Equipe

<table>
<tr>
<td align="center">
<a href="https://github.com/Lucas-AV"><img src="https://github.com/Lucas-AV.png" width="100" style="border-radius:50%" alt="Lucas Alves Vilela"></a>
<br><b>Lucas Alves Vilela</b>
<br><a href="https://github.com/Lucas-AV"><img src="https://img.shields.io/badge/GitHub-Lucas--AV-181717?logo=github&logoColor=white" alt="GitHub Lucas-AV"></a>
</td>
<td align="center">
<a href="https://github.com/dayarierref"><img src="https://github.com/dayarierref.png" width="100" style="border-radius:50%" alt="Dayane Ferreira"></a>
<br><b>Dayane Ferreira</b>
<br><a href="https://github.com/dayarierref"><img src="https://img.shields.io/badge/GitHub-dayarierref-181717?logo=github&logoColor=white" alt="GitHub dayarierref"></a>
</td>
<td align="center">
<a href="https://github.com/dudsstar16"><img src="https://github.com/dudsstar16.png" width="100" style="border-radius:50%" alt="Eduarda Reis"></a>
<br><b>Eduarda Reis</b>
<br><a href="https://github.com/dudsstar16"><img src="https://img.shields.io/badge/GitHub-dudsstar16-181717?logo=github&logoColor=white" alt="GitHub dudsstar16"></a>
</td>
<td align="center">
<a href="https://github.com/Ruan-Carvalho"><img src="https://github.com/Ruan-Carvalho.png" width="100" style="border-radius:50%" alt="Ruan Sobreira Carvalho"></a>
<br><b>Ruan Sobreira Carvalho</b>
<br><a href="https://github.com/Ruan-Carvalho"><img src="https://img.shields.io/badge/GitHub-Ruan--Carvalho-181717?logo=github&logoColor=white" alt="GitHub Ruan-Carvalho"></a>
</td>
<td align="center">
<a href="https://github.com/femathrl0"><img src="https://github.com/femathrl0.png" width="100" style="border-radius:50%" alt="Felipe Matheus"></a>
<br><b>Felipe Matheus</b>
<br><a href="https://github.com/femathrl0"><img src="https://img.shields.io/badge/GitHub-femathrl0-181717?logo=github&logoColor=white" alt="GitHub femathrl0"></a>
</td>
<td align="center">
<a href="https://github.com/rebecavitoriasalazar-cpu"><img src="https://github.com/rebecavitoriasalazar-cpu.png" width="100" style="border-radius:50%" alt="Rebeca Vitoria Salazar"></a>
<br><b>Rebeca Vitoria Salazar</b>
<br><a href="https://github.com/rebecavitoriasalazar-cpu"><img src="https://img.shields.io/badge/GitHub-rebecavitoriasalazar--cpu-181717?logo=github&logoColor=white" alt="GitHub rebecavitoriasalazar-cpu"></a>
</td>
</tr>
</table>

## Roadmap

- [x] Análise exploratória do dataset de faixas (popularidade, energia,
      dançabilidade, escala) e dashboard publicado no GitHub Pages
- [x] Análise de mercado (Julia) e pitch de investimento
- [x] Arquitetura do agente conversacional definida (Proposta B)
- [x] Épico 0 — Infraestrutura de LLM (Ollama local + backend Claude
      alternativo, health-check)
- [x] Épico 1 — Motor de recomendação por similaridade de cosseno
- [x] Épico 2 — Pipeline conversacional (roteador → extração → busca →
      geração/auditoria)
- [x] Épico 3 — Backend/API de sessões (`/session`, `/chat`, `/chat/historico`)
- [x] Épico 4 — Frontend do chat — 11 de 12 tickets (falta só o fluxo de
      logout, 4.5)
- [x] Épico 5 — Integração Spotify OAuth completa (login PKCE, tokens
      criptografados, histórico, perfil de gosto)
- [x] Épico 8 — Infra de projeto (CORS, erro global, CI) — falta só ligar
      o rate limiter pronto na rota `/chat` (8.4) e publicar em hosting
      real (8.7, opcional)
- [x] Épico 12 — Salvar playlist no Spotify, endpoint sem LLM, dark mode,
      landing page
- [x] Épico 13 — Painel "Explorar Spotify" completo (busca, player,
      lançamentos, playlists, seguindo, meus dados) + login por QR code
- [x] Épico 15 — Validação do pipeline ponta a ponta com Ollama real
      (falta só o checklist manual que depende de login numa conta
      Spotify real)
- [ ] Renomear o repositório GitHub em si pra **SyntonIA** (URL/link, ação
      manual)
- [ ] Rodar `scripts/apresentacao/build_decks.ps1` pra regenerar o PPTX/PDF
      do pitch e da técnica com o texto interno já atualizado pra SyntonIA
      (arquivos e roteiros já renomeados; falta só o rebuild)
- [ ] Modelagem de recomendação colaborativa/híbrida (hoje é baseada em
      conteúdo/similaridade)
- [ ] Avaliação com dataset complementar de interação/avaliação de
      usuários (pesquisa de hábitos musicais roteirizada em
      `docs/pesquisa/`, ainda não publicada/coletada)

## Licença

[MIT](LICENSE) — Copyright (c) 2026 Lucas Alves Vilela
