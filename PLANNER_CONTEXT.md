# Contexto Completo — Meu Planner (Esther Saraiva)

> Documento gerado em 19/05/2026 para uso em Cursor, Claude, ou qualquer outro AI.
> Descreve a arquitetura, features, dados e comportamentos do dashboard pessoal.

---

## O que é

Dashboard pessoal de gerenciamento de tarefas de uma **Product Designer no Nubank**, construído como um **único arquivo HTML** (~5700 linhas) com CSS e JavaScript embutidos. Não tem backend, não tem dependências externas (exceto Google Fonts). Toda a persistência é feita via `localStorage` com a chave `esther-v3`.

**Arquivo principal:** `meu-dashboard.html`

---

## Stack técnica

| Item | Detalhe |
|---|---|
| HTML/CSS/JS | Vanilla, tudo em um único arquivo |
| Persistência | `localStorage` — chave `esther-v3` (objeto JSON com tasks, projects, meetings, notes, initiatives, teams) |
| Fonte | Google Fonts — Inter |
| Build | Nenhum — abrir direto no browser |
| Sem frameworks | Sem React, Vue, Angular, jQuery |

---

## Estrutura do estado global (S)

```javascript
var S = {
  tasks: [],        // array de task objects
  projects: [],     // times/projetos (IMDS, IPD, POC, DC)
  meetings: [],     // alinhamentos registrados manualmente
  notes: [],        // notas e decisões
  initiatives: [],  // iniciativas do Gantt semestral
  teams: []         // times do usuário
};
```

`S` é carregado do localStorage no boot e salvo de volta em toda modificação via `save()`.

---

## Arrays de dados seedados no HTML

Além do `S` do localStorage, o arquivo contém arrays estáticos que são injetados no boot:

| Array | Descrição |
|---|---|
| `JIRA_TASKS` | Issues Jira seedadas (prefixo `jira-`) — nunca mudam automaticamente |
| `JIRA_SYNC_TASKS_DATA` | Issues sincronizadas pela skill sync-jira (prefixo `jsync-`) |
| `TRANSCRIPT_TASKS_DATA` | Tasks extraídas de transcrições de reuniões (prefixo `tr-`) |
| `CALENDAR_EVENTS_DATA` | Eventos do Google Calendar da semana atual |
| `WEEKLY_MEETINGS` | Lista de reuniões recorrentes para marcar como "analisada" |
| `SLACK_THREADS_DATA` | Threads do Slack sincronizadas (prefixo `slack-`) |

Os arrays de tasks são injetados em `S.tasks` no boot com deduplicação por `id`. Deletados pelo usuário ficam em `S.deletedIds[]` e são ignorados.

---

## Seções do dashboard (sidebar)

| Rota | ID da section | Função de render |
|---|---|---|
| My tasks | `sec-overview` | `render()` |
| My teams | `sec-projects` | `render()` |
| Week | `sec-week` | `renderWeek()` |
| Roadmap | `sec-semester` | `renderSemester()` |
| Meetings | `sec-calendar` | `renderCalendar()` |
| Slack | `sec-slack` | `renderSlack()` |

Navegação via `nav(id, el)` — ativa a section correta e chama o render adequado.

---

## Seção: My Tasks (overview)

Kanban com 3 tabs:

### Tabs
| Tab | `activeTab` | Colunas |
|---|---|---|
| Analisar | `'analisar'` | Transcrições (`transcript`), JIRA (`jira`) |
| No Radar | `'noradar'` | Tarefas (`__noradar` → blocked / backlog_item / idea) |
| Ativas | `'ativas'` | A Fazer (`todo`), Em Andamento (`doing`), Em Validação (`validation`), Entregue (`done`) |

### Task object
```javascript
{
  id: 'string',           // prefixos: jira-, jsync-, tr-, slack-task-, etc.
  title: 'string',
  type: 'task' | 'alignment' | 'presentation',
  priority: 'high' | 'medium' | 'low',
  status: 'todo' | 'doing' | 'validation' | 'done' | 'blocked' | 'backlog_item' | 'idea' | 'jira' | 'transcript',
  project: 'IMDS' | 'IPD' | 'POC' | 'DC' | '',
  initiative: 'string',
  due: 'YYYY-MM-DD' | '',
  notes: 'string',
  with: 'string',         // para alinhamentos
  subtype: 'string',      // para alinhamentos (sync, review, kickoff, etc.)
  audience: 'string',     // para apresentações
  sortOrder: number,      // para ordenação no kanban
  progress: 0-100,        // para done
  createdAt: 'ISO string',
  // Campos específicos de transcrição:
  fromMeeting: 'string',
  meetingDate: 'YYYY-MM-DD',
  // Campos específicos de Jira:
  jiraKey: 'PROJ-XXXX',
  jiraStatus: 'string'
}
```

### Drag & drop
- HTML5 Drag API com `draggable="true"` nos cards
- `ovDragStart`, `ovDragEnd`, `ovCardDragOver`, `ovCardDrop`, `ovDragOver`, `ovDragLeave`, `ovDrop`
- Suporta reordenação dentro da coluna (via `sortOrder`) e mover entre colunas

---

## Seção: Roadmap (Gantt semestral)

Gantt visual de iniciativas. Cada iniciativa tem:
- `name`, `start`, `end` (YYYY-MM-DD), `color`
- `ganttBars[]` — barras adicionais (para sub-períodos ou épicos do Jira)

Épicos do Jira são sincronizados via skill `sync-epicos` e injetados nas `ganttBars` via bloco `<!-- SYNC-EPICOS:DATA -->`.

---

## Seção: Meetings (Calendar)

Renderiza `CALENDAR_EVENTS_DATA` em layout de grade semanal. Cada evento:
```javascript
{
  id: 'cal-MMDD-N',
  date: 'YYYY-MM-DD',
  startTime: 'HH:MM',
  endTime: 'HH:MM',
  title: 'string',
  attendees: ['Nome1', 'Nome2'],
  docUrl: 'https://docs.google.com/...' | null,
  color: 'hex'
}
```

Atualizado automaticamente via task agendada (`sync-calendar-hourly`) que roda todo hora.

---

## Seção: Slack

Exibe threads do Slack onde Esther foi mencionada, classificadas como:
- `pending` — não respondidas (destaque vermelho)
- `replied` — já respondidas

Thread object:
```javascript
{
  id: 'slack-CHANNELID-TIMESTAMP',
  channel: 'nome-do-canal',
  sender: 'Nome',
  excerpt: 'Primeiros 200 chars...',
  ts: 'ISO 8601',
  status: 'pending' | 'replied',
  url: 'https://nubank.slack.com/archives/...',
  replyCount: number
}
```

Dados injetados via bloco `<!-- SYNC-SLACK:DATA -->` pelo skill `sync-slack`.
Armazenados também em `localStorage('esther-slack-threads')`.

---

## Blocos de dados injetáveis (skill injection points)

Os skills automatizados injetam dados via blocos marcados no HTML:

```html
<!-- SYNC-SLACK:DATA -->
<script>(function(){ var _slData=[...]; SLACK_THREADS_DATA=_slData; localStorage.setItem('esther-slack-threads',...); renderSlack(); })();</script>
<!-- /SYNC-SLACK:DATA -->

<!-- SYNC-EPICOS:DATA -->
<script>(function(){ var _epData=[...]; /* injeta nas iniciativas */ })();</script>
<!-- /SYNC-EPICOS:DATA -->
```

Para JIRA e transcrições, os dados são injetados diretamente nos arrays `JIRA_SYNC_TASKS_DATA` e `TRANSCRIPT_TASKS_DATA` antes do `];`.

---

## Theme / Design

- **Paleta:** Mauve Mist — roxo suave
- **Variáveis CSS principais:**
  ```css
  --bg: #F5F3FA
  --bg-page: #F0EDF7
  --sidebar-bg: #7A6A98
  --accent: #7B5EA7
  --accent-light: #EDE9F4
  --text-dark: #1e1b4b
  --border: #E6E0F0
  ```
- **Radius:** 18px (cards maiores), 10px (elementos menores)
- **Font:** Inter (Google Fonts)

---

## Times no Nubank

| Código | Nome completo | Descrição |
|---|---|---|
| IMDS | Investments & Monitoring | Time principal da Esther |
| IPD | Investments Product Design | Colaboração frequente |
| POC | Portfolio Overview & Context | Colaboração |
| DC | Design Chapter | Capítulo de design |

---

## Skills de sincronização (Cowork/Claude)

Os skills rodam dentro do Cowork (Claude desktop) e modificam o arquivo HTML diretamente:

| Skill | O que faz |
|---|---|
| `sync-jira` | Busca issues Jira atribuídas à Esther (JQL), injeta em `JIRA_SYNC_TASKS_DATA` |
| `sync-slack` | Busca threads Slack com @esther.saraiva, classifica pending/replied, injeta em `SYNC-SLACK:DATA` |
| `sync-transcripts` | Lê Google Calendar da semana, busca docs das reuniões no Drive, extrai tasks, injeta em `TRANSCRIPT_TASKS_DATA` |
| `sync-epicos` | Busca épicos Jira dos projetos, mapeia para o Gantt semestral, injeta em `SYNC-EPICOS:DATA` |
| `analisar-reuniao` | Analisa uma reunião específica por docId, extrai tasks da Esther |

---

## Hospedagem online — considerações importantes

### O que funciona estático
- Todo o dashboard visual
- Navegação entre seções
- Drag & drop entre colunas
- Modais de criação/edição de tasks
- Filtros, busca, bulk actions
- Gantt semestral
- Visualização de calendário (dados seedados no HTML)
- Visualização de Slack (dados seedados no HTML)

### O que NÃO funciona estático (precisa de adaptação)
- **localStorage**: funciona no browser do visitante, mas dados não persistem entre dispositivos
- **Sync skills**: os skills do Cowork modificam o arquivo local — em hosting online, precisaria de um fluxo alternativo (ex: exportar HTML atualizado e fazer redeploy)
- **MCPs**: Jira, Slack, Google Calendar são acessados via MCP do Cowork — não funcionam diretamente num site

### Fluxo recomendado para GitHub Pages
1. Arquivo `meu-dashboard.html` no repositório
2. GitHub Pages serve o arquivo estático
3. Quando quiser atualizar dados: rodar os skills no Cowork → arquivo local é atualizado → git push → GitHub Pages atualiza automaticamente

### Recomendação de deploy
```
GitHub Pages (grátis)
URL: https://[usuario].github.io/meu-planner/
Setup: ~10 min com ajuda do Claude no Cowork
```

---

## Funções principais para referência

| Função | O que faz |
|---|---|
| `load()` | Carrega `S` do localStorage |
| `save()` | Salva `S` no localStorage |
| `render()` | Re-renderiza o kanban (My Tasks) |
| `renderWeek()` | Re-renderiza a view semanal |
| `renderSemester()` | Re-renderiza o Gantt |
| `renderCalendar()` | Re-renderiza o calendário de reuniões |
| `renderSlack()` | Re-renderiza a seção Slack |
| `nav(id, el)` | Navega para uma seção |
| `openM(id)` / `closeM(id)` | Abre/fecha modais |
| `showToast(msg)` | Exibe notificação temporária |
| `undo()` | Desfaz última ação (Cmd+Z) |

---

## Prompt rápido para retomar desenvolvimento

```
Você está ajudando a evoluir o dashboard pessoal da Esther Saraiva (Product Designer no Nubank).

O dashboard é um único arquivo HTML (~5700 linhas) em /meu-dashboard.html.
Stack: HTML + CSS + JS vanilla, sem frameworks. Persistência via localStorage (chave: esther-v3).

Times: IMDS (principal), IPD, POC, DC.
Seções: My Tasks (kanban com tabs Analisar/No Radar/Ativas), My Teams, Week, Roadmap (Gantt), Meetings (Calendar), Slack.

Regras importantes:
- Sempre ler o arquivo antes de editar
- Manter o tema Mauve Mist (roxo suave, --accent: #7B5EA7)
- Não quebrar drag & drop (HTML5 API, atributos ondragstart/ondrop nos cards)
- Skills de sync (sync-jira, sync-slack, sync-transcripts) injetam dados via blocos marcados no HTML
- localStorage key: esther-v3

O que você precisa fazer: [DESCREVA AQUI]
```
