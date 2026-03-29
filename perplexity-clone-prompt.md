# PERPLEXITY CLONE — COMPREHENSIVE BUILD PROMPT

## PROJECT IDENTITY

You are building **Nexus** — a full-stack AI-powered research platform that is a feature-complete clone of Perplexity AI with extended developer capabilities. It combines real-time web search, multi-LLM support, multi-mode research (Fast → Deep), an in-browser/cloud sandbox environment for code execution and artifact generation, and a fully configurable settings system. The application must be production-ready, visually stunning, and built with modern best practices.

---

## TECH STACK

### Frontend
- **Framework**: Next.js 14+ (App Router)
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS + shadcn/ui component library
- **State Management**: Zustand for global state, TanStack Query for server state
- **Animations**: Framer Motion for transitions, micro-interactions, streaming text
- **Icons**: Lucide React
- **Markdown/Code Rendering**: react-markdown + rehype-highlight + rehype-katex for math
- **Editor**: Monaco Editor (for sandbox code editor)
- **File Preview**: react-pdf, react-syntax-highlighter

### Backend
- **Runtime**: Next.js API Routes (Edge Runtime where applicable)
- **Auth**: NextAuth.js with email/password + OAuth (Google, GitHub)
- **AI Orchestration**: Vercel AI SDK (handles streaming, tool use, multi-provider)
- **Search Clients**: Custom wrappers for SearXNG, Tavily, Brave Search APIs
- **Sandbox**: E2B SDK (default cloud sandbox) + optional local Docker sandbox via dockerode
- **Queue / Background Jobs**: BullMQ with Redis (for deep research multi-step pipelines)
- **Rate Limiting**: Upstash Redis

### Storage (Configurable — see SETTINGS section)
- **Default**: Supabase (PostgreSQL + Supabase Storage for files + Supabase Auth)
- **Alternative A**: PocketBase (self-hosted, SQLite-backed)
- **Alternative B**: Local SQLite via Prisma + local file system
- **File Storage**: Supabase Storage (default) / local `/uploads` / S3-compatible (configurable)

### Deployment
- **Default**: Vercel
- **Self-hosted**: Docker Compose file with all services (app, Redis, optional PocketBase)

---

## DATABASE SCHEMA

Design the following tables/collections. Use Prisma ORM with adapters for both Supabase (PostgreSQL) and SQLite.

```
User { id, email, name, avatar, plan, createdAt, settings (JSON) }

Workspace { id, userId, name, description, icon, createdAt }

Thread { id, workspaceId, userId, title, mode, createdAt, updatedAt, isPublic, shareToken }

Message {
  id, threadId, role (user|assistant|tool),
  content (text), sources (JSON array), 
  images (JSON array), artifacts (JSON array),
  model, searchEngine, searchResults (JSON),
  thinkingSteps (JSON), duration, tokenCount,
  createdAt
}

Source { id, messageId, url, title, snippet, favicon, rank }

Artifact {
  id, messageId, threadId, type (code|image|pdf|csv|chart|file),
  name, content, language, sandboxId, downloadUrl, createdAt
}

SearchHistory { id, userId, query, engine, resultCount, createdAt }

ApiKey { id, userId, provider, keyHash, label, lastUsed, createdAt }

UserSettings {
  userId (PK),
  defaultLLM, defaultSearchEngine, defaultMode,
  storageBackend, supabaseUrl, supabaseKey, pocketbaseUrl,
  sandboxProvider, e2bApiKey, dockerEndpoint,
  tavilyApiKey, braveApiKey, searxngUrl,
  openaiApiKey, anthropicApiKey, groqApiKey,
  googleApiKey, mistralApiKey, togetherApiKey,
  customLLMEndpoint, customLLMModel,
  theme, fontSize, codeTheme,
  enableTelemetry, enableMemory, enableWebAccess
}
```

---

## CORE FEATURE SPECIFICATIONS

---

### 1. SEARCH ENGINE INTEGRATION

Build a unified `SearchOrchestrator` class with a common interface:

```typescript
interface SearchResult {
  url: string
  title: string
  snippet: string
  favicon?: string
  publishedDate?: string
  score?: number
  raw?: any
}

interface SearchOptions {
  query: string
  engine: 'searxng' | 'tavily' | 'brave' | 'auto'
  maxResults?: number
  freshness?: 'day' | 'week' | 'month' | 'any'
  safeSearch?: boolean
  focus?: 'web' | 'news' | 'academic' | 'reddit' | 'youtube' | 'images'
}
```

**SearXNG Integration:**
- Connect to a user-configured SearXNG instance URL (can be self-hosted or public)
- Support all SearXNG categories: general, news, images, videos, files, social media
- Handle SearXNG JSON API output format
- Implement retry logic and timeout handling
- Support multiple SearXNG instances with fallback

**Tavily Integration:**
- Use the official Tavily API (`https://api.tavily.com/search`)
- Support `search_depth`: basic (fast) or advanced (deep)
- Include `include_raw_content` for deep research mode
- Include `include_images` for visual results
- Support `include_domains` and `exclude_domains` filtering

**Brave Search Integration:**
- Use Brave Search API v1 (`https://api.search.brave.com/res/v1/web/search`)
- Handle Brave's unique `Discussions`, `FAQ`, `Infobox`, and `News` result clusters
- Extract rich data from Brave's structured results
- Implement goggles support for custom result rankings

**Auto Mode:**
- In "auto", the orchestrator selects the engine based on availability (keys configured) and query type
- News queries → prefer Brave or Tavily with freshness filter
- Academic queries → prefer SearXNG with scholar category
- General → round-robin or preference order from settings

**Post-Processing Pipeline:**
- Deduplicate results by URL similarity (fuzzy match)
- Re-rank results using a lightweight scoring model based on query relevance
- Fetch full page content for top N results in deep research mode (using Cheerio/JSDOM)
- Extract structured data (author, date, word count) from fetched pages
- Build source citation index mapped to message

---

### 2. LLM PROVIDER SYSTEM

Build a `LLMRouter` that provides a unified streaming interface across all providers via Vercel AI SDK.

**Supported Providers:**

| Provider | Models to Include |
|---|---|
| OpenAI | gpt-4o, gpt-4o-mini, gpt-4-turbo, o1-preview, o1-mini |
| Anthropic | claude-opus-4-5, claude-sonnet-4-5, claude-haiku-3-5 |
| Google | gemini-2.0-flash, gemini-1.5-pro, gemini-1.5-flash |
| Groq | llama-3.3-70b, mixtral-8x7b, gemma2-9b |
| Mistral | mistral-large, mistral-medium, mistral-small, codestral |
| Together AI | meta-llama/Llama-3.3-70B, Qwen/Qwen2.5-72B |
| Ollama | Any local model (user specifies endpoint + model name) |
| Custom OpenAI-compatible | Any endpoint, any model |

**LLM Router Interface:**
```typescript
interface LLMConfig {
  provider: string
  model: string
  apiKey?: string
  baseURL?: string
  temperature?: number
  maxTokens?: number
  streaming: boolean
}
```

**System Prompt Construction:**
- Inject current date/time
- Inject user memory snippets (if memory enabled)
- Inject search results as formatted context blocks with citation markers `[1]`, `[2]`, etc.
- Inject sandbox context if in sandbox mode
- Add mode-specific instructions (fast = concise, deep = thorough with sections)

**Streaming:**
- Use Vercel AI SDK `streamText` for all providers
- Stream tokens to the client via Server-Sent Events
- Stream thinking steps separately for models that support extended thinking (claude)
- Track time-to-first-token, total tokens, and cost estimate

---

### 3. RESEARCH MODES

Implement the following modes selectable per-thread or per-query:

#### **Fast Mode** (default)
- 1 search call, top 5 results, no page fetching
- Model: fastest available (flash/mini/haiku tier)
- Response target: < 5 seconds
- No multi-step reasoning
- Concise, direct answers with inline citations

#### **Balanced Mode**
- 1-2 search calls, top 10 results, fetch top 3 pages
- Model: mid-tier (sonnet/gpt-4o-mini/flash)
- Response target: < 15 seconds
- One reasoning pass before answering

#### **Deep Research Mode**
- Multi-step pipeline using BullMQ:
  1. **Query Decomposition**: LLM breaks query into 3-5 sub-questions
  2. **Parallel Search**: Search all sub-questions simultaneously across configured engines
  3. **Page Fetching**: Fetch and extract full content from top 5 results per sub-question
  4. **Synthesis**: LLM synthesizes all gathered information
  5. **Fact Check Pass**: LLM cross-references key claims against sources
  6. **Report Generation**: Generate a structured markdown report with sections, headers, and citations
- Model: most capable available (opus/gpt-4o/gemini-pro)
- Response target: 2-10 minutes (show live progress)
- Output: Long-form report with table of contents, executive summary, detailed sections, and full bibliography
- Show live progress steps in UI: "Searching...", "Reading 8 pages...", "Synthesizing...", etc.

#### **Focus Modes** (like Perplexity)
- **Web**: General web search
- **Academic**: Scholar-focused (SearXNG scholar category + arxiv)
- **YouTube**: Search YouTube, embed video results with timestamps
- **Reddit**: Search Reddit threads, surface community insights
- **News**: Real-time news focus with freshness filters
- **Code**: Prioritize GitHub, StackOverflow, documentation sites
- **Math**: Enable KaTeX rendering, route to a math-capable model

#### **Pro Search Toggle**
- Adds one clarifying question before searching (like Perplexity Pro)
- Multi-query expansion

---

### 4. SANDBOX FEATURE

The Sandbox is a full code execution and artifact generation environment. It must work in two modes configurable in Settings.

#### **Cloud Sandbox (E2B — Default)**
- Use the `@e2b/code-interpreter` SDK
- Create sandboxes on-demand per session
- Support running: Python, JavaScript/Node.js, Bash, R
- Auto-install packages with pip/npm as needed
- Capture stdout, stderr, file outputs, and rich outputs (plots, dataframes)
- 30-minute sandbox lifetime with heartbeat extension
- Upload files to sandbox from user uploads
- Download generated files from sandbox

#### **Local Docker Sandbox (Self-hosted)**
- Use `dockerode` to manage local Docker containers
- Use official images: `python:3.11-slim`, `node:20-alpine`
- Mount a temp volume per session
- Execute commands via `docker exec`
- Capture output streams
- Auto-cleanup containers after session

#### **Sandbox UI**
Build a split-pane sandbox view:
- **Left pane**: Chat / instructions input
- **Right pane**: Monaco Editor with language selector + Terminal output panel
- Toolbar: Run, Stop, Reset, Download files, Install package
- File Explorer panel: tree view of sandbox filesystem
- Output tabs: Terminal, Preview (for HTML output), Files, Plots
- Inline artifact rendering: show matplotlib plots, dataframe tables, HTML previews inline in chat
- LLM can autonomously write and run code in the sandbox (agent loop)

#### **Agent Loop (Autonomous Sandbox)**
- When in sandbox mode, use a tool-calling agent loop:
  - Tool: `run_code(language, code)` → returns output
  - Tool: `write_file(path, content)` → writes to sandbox
  - Tool: `read_file(path)` → reads from sandbox
  - Tool: `install_package(manager, packages)` → runs pip/npm install
  - Tool: `list_files(path)` → returns directory listing
- Agent iterates until task is complete or max_iterations (20) reached
- Show each tool call and result as a collapsible step in the UI
- Final artifacts are saved to the Artifacts database table

#### **Supported Artifact Types**
Generated artifacts are saved, rendered inline, and downloadable:
- Python plots (matplotlib, seaborn, plotly) → PNG/interactive HTML
- Data analysis results → CSV + rendered table
- Generated files → PDF, DOCX, XLSX (via python-docx, openpyxl, reportlab)
- Code outputs → syntax-highlighted code blocks with copy button
- HTML outputs → sandboxed iframe preview
- Generated images → lightbox gallery view

---

### 5. UI / UX DESIGN

#### **Design Language**
- **Theme**: Dark by default, light mode available, system-adaptive
- **Aesthetic**: Refined dark glassmorphism — deep navy/charcoal backgrounds (`#0A0B0F`, `#111318`), glass cards with subtle borders (`rgba(255,255,255,0.06)`), clean white typography, electric blue accents (`#3B82F6`), purple-to-blue gradients for interactive elements
- **Font**: `Geist` (Vercel's font) for UI, `JetBrains Mono` for code
- **Border radius**: Consistently 12px for cards, 8px for inputs, 24px for pills
- **Shadows**: Glow effects on active elements using box-shadow with accent color at low opacity

#### **Layout — Main Application Shell**

```
┌──────────────────────────────────────────────────────┐
│  SIDEBAR (240px, collapsible)  │  MAIN CONTENT AREA  │
│                                │                      │
│  [Logo — Nexus]                │  [Thread View /      │
│                                │   Home / Settings]   │
│  [New Thread button]           │                      │
│  [Search threads]              │                      │
│                                │                      │
│  Workspaces                    │                      │
│    ▸ Personal                  │                      │
│    ▸ Work                      │                      │
│                                │                      │
│  Recent Threads                │                      │
│    - Thread 1                  │                      │
│    - Thread 2                  │                      │
│    ...                         │                      │
│                                │                      │
│  [Settings]                    │                      │
│  [User Profile]                │                      │
└──────────────────────────────────────────────────────┘
```

#### **Home Screen**
- Centered logo with animated gradient glow on page load
- Large search input with rounded corners, prominent placeholder: "Ask anything..."
- Below input: Mode pills (Fast · Balanced · Deep · Focus ▾) + Engine selector
- Suggested prompts grid (4 cards): trending, curated, or personalized queries
- Recent threads list below suggestions
- Keyboard shortcut: `/` to focus search from anywhere

#### **Thread View (Chat)**

The thread view is the core experience:

```
┌─────────────────────────────────────────────────────────┐
│  Thread Title (editable on click)        [Share] [⋯]    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [User Message]                                         │
│  ──────────────────────────────                         │
│  [Sources Bar] ○ source1.com  ○ source2.com  ○ +3 more  │
│  [Answer Block — streaming markdown with citations]     │
│  [Image Grid — if images found]                         │
│  [Related Questions — 4 clickable pills]                │
│  [Artifact Cards — if sandbox generated files]          │
│                                                         │
│  [User Message 2]                                       │
│  ...                                                    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  [Input Area]                                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Ask a follow-up...                   [⊕] [Send]  │   │
│  └──────────────────────────────────────────────────┘   │
│  Mode: Fast ▾  Engine: Auto ▾  Model: GPT-4o ▾          │
└─────────────────────────────────────────────────────────┘
```

**Sources Bar:**
- Horizontal scrollable list of source favicon + domain pills
- Click to expand source panel (shows full snippet + link)
- Source numbers `[1]` `[2]` inline in answer text are clickable, highlight the source card

**Answer Streaming:**
- Tokens stream in with a subtle cursor animation
- Markdown rendered in real-time (headings, bold, lists, tables, code blocks)
- Code blocks: syntax highlighted, copy button, language badge, "Run in Sandbox" button
- Citations `[1]` superscripted and highlighted, hover shows source tooltip
- Math equations rendered with KaTeX
- Images displayed in a responsive masonry grid below the answer

**Deep Research Progress:**
- Show a live progress panel above the answer while deep research runs:
  - Animated step indicators: ✓ Decomposing query → ✓ Searching 5 sub-questions → ⟳ Reading 12 pages → ...
  - Show found sources count in real time
  - Estimated time remaining

**Related Questions:**
- After each answer, show 4 clickable follow-up question pills
- Generated by LLM based on the conversation context
- Click any to instantly send as next message

**Thread Actions:**
- Share thread → generate public link (if user makes it public)
- Rename thread (auto-named by LLM on first message)
- Export thread → PDF or Markdown
- Delete thread

#### **Sandbox View**
Accessible via "Open Sandbox" button in thread or from sidebar:
- Full-screen split view with resizable panes
- Chat pane on left (thread continues here)
- Workspace pane on right with tabs: Editor / Terminal / Files / Preview
- Persistent sandbox session per thread

#### **Settings Panel**
Full-screen settings with sidebar navigation:

```
Settings
├── General
│   ├── Theme (Dark / Light / System)
│   ├── Font size
│   ├── Code theme (VS Dark / Dracula / Monokai / GitHub Light)
│   ├── Language / locale
│   └── Default workspace
│
├── AI Models
│   ├── Default model selector (dropdown of all configured providers)
│   ├── OpenAI API Key + model list
│   ├── Anthropic API Key + model list
│   ├── Google API Key + model list
│   ├── Groq API Key + model list
│   ├── Mistral API Key + model list
│   ├── Together AI API Key + model list
│   ├── Ollama endpoint URL + model (auto-detected)
│   └── Custom OpenAI-compatible endpoint + model
│
├── Search Engines
│   ├── Default engine (Auto / SearXNG / Tavily / Brave)
│   ├── SearXNG instance URL + test button
│   ├── Tavily API Key + test button
│   ├── Brave Search API Key + test button
│   └── Safe search toggle
│
├── Research
│   ├── Default mode (Fast / Balanced / Deep)
│   ├── Max search results (5 / 10 / 20)
│   ├── Enable Pro Search by default
│   └── Max deep research duration
│
├── Sandbox
│   ├── Sandbox provider (E2B Cloud / Local Docker / Disabled)
│   ├── E2B API Key + sandbox template selector
│   ├── Docker endpoint (for local: unix:///var/run/docker.sock)
│   ├── Default sandbox image
│   ├── Max sandbox lifetime (minutes)
│   └── Auto-run code toggle
│
├── Storage
│   ├── Backend (Supabase / PocketBase / Local SQLite)
│   ├── Supabase URL + anon key + service role key
│   ├── PocketBase URL + admin credentials
│   └── Local data directory path
│
├── Memory
│   ├── Enable memory (stores facts about user across threads)
│   ├── View and delete stored memories
│   └── Memory size limit
│
├── Privacy & Security
│   ├── Enable telemetry
│   ├── Clear all history
│   ├── Export all data (GDPR)
│   └── Delete account
│
└── About / Version
```

All settings are stored encrypted in UserSettings. API keys are AES-256 encrypted before storage. Show/hide key toggle on each key field.

---

### 6. ADDITIONAL PERPLEXITY-PARITY FEATURES

#### **Memory System**
- After each thread, an LLM pass extracts key facts about the user (profession, preferences, ongoing projects)
- Facts stored in a `Memory` table with recency scoring
- Injected into system prompt on new threads (top 5 most relevant memories)
- User can view, edit, and delete memories in settings

#### **Collections / Spaces**
- Users create named Spaces (like Perplexity Spaces / Workspaces)
- Spaces have: name, icon, description, system prompt override, default model, default search engine
- All threads belong to a Space
- Spaces can be shared with other users (collaborative)

#### **Thread Sharing**
- Generate a public read-only share link for any thread
- Shared threads rendered without sidebar (clean reading view)
- Optional password protection for shared threads

#### **Image Search & Rendering**
- Search for images using engine image capabilities
- Display image results in a masonry grid
- Click to open full-screen lightbox with source attribution
- AI can describe and analyze uploaded images (multimodal)

#### **File Upload & Analysis**
- Users upload files (PDF, CSV, DOCX, images, code files)
- Files processed: PDFs text-extracted, CSVs parsed to JSON, images encoded as base64
- File content injected into context or sent to sandbox for analysis
- Drag-and-drop zone in input area

#### **Voice Input**
- Microphone button in input area
- Use Web Speech API for browser-native transcription
- Optional: OpenAI Whisper API for higher accuracy
- Auto-submit on silence detection toggle

#### **Citation Formatting**
- Toggle between citation styles: Inline `[1]`, Footnote, or Academic (APA/MLA)
- "Copy with citations" button on each answer

#### **Keyboard Shortcuts**
- `/` → focus search
- `Cmd+K` → command palette (new thread, search threads, change model)
- `Cmd+Shift+S` → open sandbox
- `Cmd+Enter` → send message
- `Esc` → close panels
- `Cmd+/` → toggle sidebar

#### **Command Palette**
- `Cmd+K` opens a Raycast-style palette
- Actions: New thread, Search threads, Switch mode, Switch engine, Switch model, Open settings, Toggle sidebar

---

### 7. API ROUTES STRUCTURE

```
/api
  /auth
    /[...nextauth]         — NextAuth handlers
  /threads
    /route.ts              — GET (list), POST (create)
    /[id]/route.ts         — GET, PATCH, DELETE
    /[id]/messages/route.ts — GET messages, POST new message
    /[id]/share/route.ts   — POST create share link
    /[id]/export/route.ts  — GET export as PDF/MD
  /search
    /route.ts              — POST unified search endpoint
  /llm
    /stream/route.ts       — POST streaming LLM endpoint (Edge Runtime)
  /sandbox
    /create/route.ts       — POST create sandbox session
    /[id]/run/route.ts     — POST run code in sandbox
    /[id]/files/route.ts   — GET/POST files in sandbox
    /[id]/destroy/route.ts — DELETE destroy sandbox
  /memory
    /route.ts              — GET list, DELETE clear
  /settings
    /route.ts              — GET, PATCH user settings
  /upload
    /route.ts              — POST file upload handler
  /discover
    /route.ts              — GET trending topics / suggestions
```

---

### 8. ENVIRONMENT VARIABLES

```bash
# App
NEXTAUTH_SECRET=
NEXTAUTH_URL=

# OAuth (optional)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=

# Default Storage (Supabase)
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Redis (for BullMQ + rate limiting)
REDIS_URL=

# Search engines (server-side fallback defaults)
SEARXNG_URL=
TAVILY_API_KEY=
BRAVE_API_KEY=

# LLM defaults (server-side fallback)
OPENAI_API_KEY=
ANTHROPIC_API_KEY=

# Sandbox
E2B_API_KEY=

# Encryption key for storing user API keys
ENCRYPTION_KEY=

# Feature flags
ENABLE_TELEMETRY=false
ENABLE_SIGNUP=true
```

Note: Per-user API keys configured in Settings override these server-side defaults. This allows self-hosting where the admin sets defaults, but users can bring their own keys.

---

### 9. SECURITY REQUIREMENTS

- All API routes protected by NextAuth session middleware
- API keys encrypted at rest using AES-256-GCM with per-user derived keys
- Sandbox environments fully isolated (E2B handles this; Docker uses no-network option for sensitive ops)
- Rate limiting on all API routes: 60 req/min per user (configurable)
- Input sanitization on all user-provided content
- CSP headers configured for iframe sandboxing of previews
- SSRF protection: validate and whitelist SearXNG/PocketBase URLs
- Share links use opaque tokens (UUID v4), not guessable IDs

---

### 10. DEVELOPER-SPECIFIC FEATURES

For users identified as developers (based on settings toggle or detected via usage patterns):

- **API Access**: Generate personal API keys to access their Nexus instance as an API (OpenAI-compatible endpoint). Developers can call their Nexus as an AI search API from their own apps.
- **Webhook Support**: Configure webhooks on thread events (message created, research completed)
- **Custom System Prompts per Space**: Full control over system prompt for each workspace
- **Plugin System** (stretch goal): A simple plugin interface where JS modules can be dropped in to add new tools (e.g., a custom database query tool)
- **Debug Mode**: In settings, toggle debug mode to see raw search results, full prompts sent to LLM, token counts, and latency breakdown for every message
- **Export Thread as JSON**: Full structured export including all sources, search queries, model used, tokens
- **Self-Hosting Docker Compose**: Provide a `docker-compose.yml` that spins up the full stack (Next.js app + Redis + PocketBase as storage alternative) with one command

---

### 11. PERFORMANCE REQUIREMENTS

- **First Contentful Paint**: < 1.5s (use Next.js static generation for shell, stream data)
- **Streaming latency**: Time to first token < 800ms for fast mode
- **Search latency**: < 2s for SearXNG/Brave; < 3s for Tavily advanced
- **Lighthouse score**: > 90 on all metrics
- **Bundle size**: Keep client JS under 300KB (code split aggressively)
- **Search results cached**: 5-minute TTL with Redis for identical queries
- **LLM responses cached**: Optional semantic caching with embedding similarity

---

### 12. FOLDER STRUCTURE

```
nexus/
├── app/
│   ├── (auth)/login/page.tsx
│   ├── (auth)/register/page.tsx
│   ├── (app)/layout.tsx            — App shell with sidebar
│   ├── (app)/page.tsx              — Home / search page
│   ├── (app)/thread/[id]/page.tsx  — Thread view
│   ├── (app)/sandbox/[id]/page.tsx — Sandbox view
│   ├── (app)/settings/page.tsx     — Settings
│   ├── (app)/discover/page.tsx     — Discover / trending
│   ├── share/[token]/page.tsx      — Public share view
│   └── api/...
├── components/
│   ├── ui/                         — shadcn/ui base components
│   ├── thread/
│   │   ├── MessageList.tsx
│   │   ├── MessageBubble.tsx
│   │   ├── SourcesBar.tsx
│   │   ├── AnswerBlock.tsx
│   │   ├── RelatedQuestions.tsx
│   │   ├── ArtifactCard.tsx
│   │   ├── DeepResearchProgress.tsx
│   │   └── ThreadInput.tsx
│   ├── sandbox/
│   │   ├── SandboxLayout.tsx
│   │   ├── CodeEditor.tsx
│   │   ├── Terminal.tsx
│   │   ├── FileExplorer.tsx
│   │   └── PreviewPane.tsx
│   ├── sidebar/
│   │   ├── Sidebar.tsx
│   │   ├── ThreadList.tsx
│   │   └── WorkspaceSelector.tsx
│   ├── settings/
│   │   └── ...settings panels
│   └── shared/
│       ├── CommandPalette.tsx
│       ├── ThemeProvider.tsx
│       └── ...
├── lib/
│   ├── search/
│   │   ├── orchestrator.ts
│   │   ├── searxng.ts
│   │   ├── tavily.ts
│   │   └── brave.ts
│   ├── llm/
│   │   ├── router.ts
│   │   ├── providers/
│   │   └── prompts/
│   ├── sandbox/
│   │   ├── e2b.ts
│   │   └── docker.ts
│   ├── storage/
│   │   ├── adapter.ts
│   │   ├── supabase.ts
│   │   ├── pocketbase.ts
│   │   └── sqlite.ts
│   ├── queue/
│   │   ├── deepResearch.ts
│   │   └── workers/
│   ├── memory/
│   │   └── extractor.ts
│   ├── crypto.ts               — API key encryption
│   ├── rateLimit.ts
│   └── utils.ts
├── prisma/
│   └── schema.prisma
├── store/
│   ├── useAppStore.ts          — Zustand global state
│   └── useSettingsStore.ts
├── hooks/
│   ├── useThread.ts
│   ├── useSearch.ts
│   ├── useSandbox.ts
│   └── useStream.ts
├── types/
│   └── index.ts
├── public/
├── docker-compose.yml
├── .env.example
└── README.md
```

---

### 13. BUILD INSTRUCTIONS

Build this application feature by feature in the following order:

1. **Project scaffolding**: Next.js 14 + TypeScript + Tailwind + shadcn/ui + Prisma setup
2. **Auth system**: NextAuth with email/password, session middleware
3. **Database schema**: Prisma models, migrations for both PostgreSQL and SQLite
4. **Storage abstraction layer**: Adapter pattern so all DB calls go through one interface
5. **Settings UI + storage**: Full settings panel with API key management
6. **Search engines**: SearXNG → Tavily → Brave implementations + orchestrator
7. **LLM router**: All providers via Vercel AI SDK, streaming endpoint
8. **Home page + basic thread**: Simple search → stream answer flow (Fast mode only)
9. **Sources bar + citation rendering**: Inline citations, source cards
10. **Sidebar + thread management**: Thread list, workspaces, CRUD
11. **Balanced and Deep research modes**: Multi-step pipeline with BullMQ
12. **Focus modes**: Web / Academic / News / Reddit / Code / Math
13. **Sandbox (E2B)**: Cloud sandbox integration, code editor, terminal
14. **Sandbox agent loop**: Tool-calling agent that autonomously uses sandbox
15. **Docker sandbox**: Local sandbox alternative
16. **File upload + analysis**: Drag-drop upload, PDF/CSV/image processing
17. **Memory system**: Extraction, storage, injection
18. **Spaces / Workspaces**: Full workspace management with custom settings
19. **Thread sharing**: Public links, password protection
20. **Command palette**: Cmd+K quick actions
21. **Voice input**: Speech API integration
22. **Developer features**: API access, debug mode, webhooks
23. **Self-hosting**: Docker Compose, README
24. **Performance optimization**: Caching, bundle splitting, Lighthouse audit

---

### 14. QUALITY STANDARDS

- Every component must have TypeScript types — no `any` unless unavoidable
- Error states handled gracefully everywhere — no unhandled promise rejections
- Loading skeletons for every async operation
- Responsive design: works on mobile (collapse sidebar to drawer), tablet, and desktop
- Accessible: ARIA labels, keyboard navigation, focus management
- Empty states: every list/view has a meaningful empty state with a CTA
- Optimistic UI updates where possible
- Toast notifications for all async actions (success / error)
- Console.log statements removed before any commit

---

*Build Nexus to be the definitive open-source Perplexity alternative — more configurable, more developer-friendly, self-hostable, and visually superior.*
