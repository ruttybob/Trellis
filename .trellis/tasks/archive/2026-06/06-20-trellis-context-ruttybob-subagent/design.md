# Design: Расширение trellis-context + инъекция в ruttybob subagent

## Архитектура

```
┌─ ruttybob/extensions/ ─────────────────────────────────────────────────┐
│                                                                        │
│  subagent/                    ← GENERIC, не знает про Trellis         │
│  ├── index.ts                                                       │   │
│  ├── agents.ts   ← +PI_SUBAGENT_AGENT_DIRS  (приоритет)              │   │
│  ├── config.ts   ← +PI_SUBAGENT_AGENT_SCOPE (override)               │   │
│  ├── runner.ts   ← +PI_SUBAGENT_CONTEXT_CMD (inject system-prompt)   │   │
│  └── ...                                                            │   │
│                                                                      │   │
│  trellis-context/             ← НОВОЕ, знает про .trellis/           │   │
│  ├── index.ts    ← фабрика + guard + env-setup + hooks wiring        │   │
│  ├── context.ts  ← contextKey/adoptKey/buildContext/readTaskDir      │   │
│  ├── workflow.ts ← workflowBreadcrumb/sessionOverview                │   │
│  ├── shared.ts   ← readText/exists/splitFM/hash/shellQuote           │   │
│  ├── AGENTS.md                                                     │   │
│  └── README.md                                                     │   │
│                                                                      │   │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              │ env-контракт (process.env)
                              ▼
              ┌───────────────────────────────────────┐
              │  PI_SUBAGENT_AGENT_SCOPE              │
              │  PI_SUBAGENT_AGENT_DIRS               │
              │  PI_SUBAGENT_CONTEXT_CMD              │
              └───────────────────────────────────────┘
```

## Env-контракт (точка склейки)

Три переменные, ruttybob subagent читает их в **момент исполнения** (не load),
trellis-context выставляет при **load** (если `.trellis/` найден).

| Переменная | Кто пишет | Кто читает | Семантика |
|---|---|---|---|
| `PI_SUBAGENT_AGENT_SCOPE` | trellis-context | ruttybob `config.ts` | override scope: `user\|project\|both`. Приоритет над settings.json. |
| `PI_SUBAGENT_AGENT_DIRS` | trellis-context | ruttybob `agents.ts` | доп. директории (path.delimiter), высший приоритет в discovery (как `--agents`). |
| `PI_SUBAGENT_CONTEXT_CMD` | trellis-context | rutlybob `runner.ts` | команда, stdout → `--append-system-prompt` субагента. spawnSync + timeout, stderr skip, nonzero exit → skip без краша. |

### Почему env, а не settings.json / inter-ext API

- **vs settings.json** — требует ручного редактирования в каждом проекте; env
  выставляется автоматически при наличии `.trellis/`.
- **vs inter-ext API** — pi не предоставляет канонического меж-расширенческого
  реестра; env — process-level контракт без связи в коде.
- **vs `--agents` flag** — flag требует CLI-изменений; env наследуется всем
  дочерним процессам автоматически (важно: ruttybob subagent spawn'ит `pi` с
  `spawnEnv = { ...process.env }`, так что env доходит до внука-субагента).

### Наследование env

`runner.ts:runSingleAgent` уже делает `const spawnEnv = { ...process.env }`.
trellis-context выставляет `process.env.PI_SUBAGENT_*` при load → они автоматически
доходят до:
1. ruttybob subagent (читает при исполнении tool)
2. дочернего `pi` процесса (через `spawnEnv`)
3. sub-sub-agents (рекурсивно через тот же механизм)

## Механика guard в trellis-context

```typescript
function findTrellisRoot(start: string): string | null {
  let dir = resolve(start);
  while (true) {
    if (existsSync(join(dir, ".trellis"))) return dir;
    const parent = dirname(dir);
    if (parent === dir) return null;
    dir = parent;
  }
}

export default function (pi: ExtensionAPI) {
  const root = findTrellisRoot(process.cwd());
  if (!root) return;                  // ← ВНЕ .trellis: расширение полностью неактивно
  setupSubagentContract(root);        // выставляет env
  registerHooks(pi, root);            // session_start, tool_call, before_agent_start, ...
}
```

Отличие от оригинала: `findRoot` в trellis/index.ts искал `.trellis` **ИЛИ** `.pi`,
что приводило к активации в любом pi-проекте. Новая версия ищет **только** `.trellis`.

## Per-turn кэш (перенос из оригинала)

`before_agent_start` дёргает `get_context.py` через `spawnSync`. Чтобы не
спавнить python каждый ход — per-turn кэш с TTL 1500ms:

```typescript
let turnCache: { key, ts, wf, ov } | null = null;
const getTurnCtx = (k) => {
  if (turnCache && turnCache.key === k && Date.now() - turnCache.ts < 1500)
    return turnCache;
  turnCache = { key: k, ts: Date.now(), wf: workflowBreadcrumb(root,k), ov: sessionOverview(root,k) };
  return turnCache;
};
```

## Перенос функций из trellis/index.ts → trellis-context

| Функция | Куда | Действие |
|---|---|---|
| `findRoot` | `index.ts` → переименована в `findTrellisRoot`, ищет только `.trellis` | модифицировать |
| `contextKey` | `context.ts` | перенести как есть |
| `adoptKey` | `context.ts` | перенести как есть |
| `sessionHasTask` | `context.ts` | перенести как есть |
| `readTaskDir` | `context.ts` | перенести как есть |
| `buildContext` | `context.ts` | перенести как есть (используется в `before_agent_start` для главного агента) |
| `TRELLIS_AGENT_JSONL` | `context.ts` | перенести как есть |
| `workflowBreadcrumb` | `workflow.ts` | перенести как есть |
| `WF_RE` | `workflow.ts` | перенести как есть |
| `sessionOverview` | `workflow.ts` | перенести как есть |
| `readText/exists/splitFM/stripFM/hash/shellQuote/cmdHasTrellisCtx` | `shared.ts` | перенести |
| `splitModelThinking` | НЕ переносится | используется только в subagent-части |
| `parseAgentFM` | НЕ переносится | subagent-часть |
| `runPi/runSubagent/BBC/applyEvent` | НЕ переносятся | subagent-generic, уходит |
| `renderProgressCard/nativeCards/alt+o` | НЕ переносятся | subagent-generic |
| `resolvePiCli/buildPiArgs` | НЕ переносятся | subagent-generic |
| `trellis_subagent` tool registration | НЕ переносится | заменяется на ruttybob `subagent` |

## Хуки: точный перенос

```typescript
pi.on("session_start", (event, ctx) => {
  getKey(event, ctx);
  ctx?.ui?.notify?.("Trellis project context is available. ...", "info");
});

pi.on("tool_call", (event, ctx) => {
  const k = getKey(event, ctx);
  const ev = event as { toolName?: string; input?: JsonObject };
  if (ev.toolName === "bash" && isObj(ev.input) && typeof ev.input.command === "string"
      && !cmdHasTrellisCtx(ev.input.command)) {
    ev.input.command = `export TRELLIS_CONTEXT_ID=${shellQuote(k)}; ${ev.input.command}`;
  }
});

pi.on("before_agent_start", (event, ctx) => {
  const k = getKey(event, ctx);
  const cur = (event as { systemPrompt?: string }).systemPrompt ?? "";
  const ctxText = buildContext(root, "trellis-implement", k);
  const { wf, ov } = getTurnCtx(k);
  return { systemPrompt: [cur, ctxText, wf, ov].filter(Boolean).join("\n\n") };
});

pi.on("context", (event, ctx) => { getKey(event, ctx); });
```

## Per-agent opt-out: frontmatter `trellis: false`

Агент может явно отказаться от context injection, указав в frontmatter:

```yaml
---
name: my-general-agent
description: ...
trellis: false      # ← не инъектировать контекст этому агенту
---
```

Семантика:
- `trellis: false` → opt-out: `PI_SUBAGENT_CONTEXT_CMD` **пропускается** для этого
  агента, `--append-system-prompt` с context-cmd не добавляется.
- `trellis: true` или поле отсутствует → обычное поведение (env-контракт решает).

Влияет **только на context injection**. Discovery агента не затрагивается — агент
остаётся доступен для вызова. `PI_SUBAGENT_AGENT_DIRS` / `PI_SUBAGENT_AGENT_SCOPE`
действуют независимо.

Назначение: агенты общего назначения (не завязанные на Trellis — например
`code-reviewer`, `web-searcher`) не получают лишний session-overview в контексте,
экономя токены.

## ruttybob subagent: точки изменений

### `agents.ts` — `discoverAgents(...)` + чтение env + frontmatter `trellis`

После существующей логики `customDir` добавить env-based директории
с высшим приоритетом:

### `config.ts` — `loadSubagentConfig` учитывает env override

В `buildConfig` после слияния global+project:

```typescript
const envScope = process.env.PI_SUBAGENT_AGENT_SCOPE;
if (envScope && validScopes.has(envScope)) agentScope = envScope as AgentScope;
```

```typescript
// PI_SUBAGENT_AGENT_DIRS: env-контракт (высший приоритет в discovery)
const envDirs = (process.env.PI_SUBAGENT_AGENT_DIRS ?? "")
  .split(path.delimiter).map(s => s.trim()).filter(Boolean);
for (const d of envDirs) {
  if (isDirectory(d)) {
    const envAgents = loadAgentsFromDir(d, "user");
    for (const a of envAgents) agentMap.set(a.name, a);
  }
}
```

Плюс расширить `AgentConfig` и парсинг frontmatter:

```typescript
export interface AgentConfig {
  // ... существующие поля
  trellis?: boolean;        // opt-out из context injection
}

// в loadAgentsFromDir, после парсинга thinking:
const trellisRaw = frontmatter.trellis;
const trellis = trellisRaw === undefined ? undefined
  : String(trellisRaw).trim().toLowerCase() === "true";
// передать trellis в объект агента
```

```typescript
const ctxCmd = process.env.PI_SUBAGENT_CONTEXT_CMD;
if (ctxCmd?.trim() && agent.trellis !== false) {   // ← per-agent opt-out
  try {
    const r = spawnSync(ctxCmd, [], {
      cwd: cwd ?? defaultCwd, encoding: "utf-8",
      timeout: 3000, shell: true, windowsHide: true,
      env: { ...process.env },   // наследуем TRELLIS_CONTEXT_ID и пр.
    });
    if (r.status === 0 && r.stdout?.trim()) {
      const tmp = await writePromptToTempFile("context", r.stdout);
      args.push("--append-system-prompt", tmp.filePath);
      // cleanup в finally (расширить существующий блок)
    }
  } catch { /* skip без краха */ }
}
```

Важно: stdout context-cmd наследует `TRELLIS_CONTEXT_ID` (env передаётся), так
что `get_context.py` отдаёт контекст именно для текущей сессии/задачи.

## Риски и совместимость

- **R-1: ruttybob без .trellis** — env не выставляется, subagent работает как
  раньше. ✅ Обратно совместимо.
- **R-2: ruttybob с .trellis, но без get_context.py** — trellis-context не
  выставляет `PI_SUBAGENT_CONTEXT_CMD` (проверка `exists(script)`). Субагенты
  запускаются без session-overview, главный агент всё равно получает
  buildContext. ✅ Graceful degradation.
- **R-3: context-cmd таймаут/ненулевой exit** — runner.ts skip'ает без краха.
  ✅ Resilient.
- **R-4: дубликаты имён агентов .pi vs .trellis** — в Trellis-проекте имена
  разные (`trellis-implement` vs `implement`), конфликта нет. Если в другом
  проекте имена совпадут — побеждает `.trellis/agents` (высший приоритет env),
  что соответствует приоритету оригинальной ruttybob-логики (custom > project).
- **R-5: env выставляется глобально на process** — если в одном процессе pi
  запущено несколько сессий в разных cwd, env будет от первого load. На
  практике pi-per-process, риск минимальный. Принято.

- **R-6 (вариант B): subagent становится минимально Trellis-aware** — для
  переименования tool name `subagent` → `trellis_subagents` при наличии
  `.trellis/`, расширение `extensions/subagent/` импортирует `findTrellisRoot`
  из `extensions/trellis-context/index.js` (через общий хелпер
  `resolveSubagentToolName` в `subagent/utils.ts`).
  - Зависимость однонаправленная: `subagent → trellis-context`,
    `trellis-context` НЕ импортирует `subagent` → **цикла нет**.
  - Импорт модуля НЕ запускает default-фабрику trellis-context (вызывается
    только loader'ом) — только top-level декларации, побочных эффектов нет.
  - Порядок загрузки расширений не важен: subagent сам зовёт `findTrellisRoot`,
    не зависит от того, выполнилась ли фабрика trellis-context.
  - Принято пользователем как детерминированная альтернатива гонке порядка
    загрузки (вариант A — trellis-context перерегистрирует tool — отвергнут).
  - Тестовая инфраструктура: ruttybob сам имеет `.trellis/`, поэтому тесты
    generic-пути фиксируют `findTrellisRoot → null` (vi.mock / stub в utils-mock)
    — см. implement.md Шаг 1.1.

## Тестирование (ruttybob vitest)

- `tests/stubs/` — добавить stub для `spawnSync` если отсутствует (используется
  в runner.ts для context-cmd).
- Юнит-тесты:
  - `agents.ts`: env dir добавляется, высший приоритет, невалидный путь skip.
  - `config.ts`: env override > settings > default.
  - `runner.ts`: context-cmd stdout → args, ненулевой exit skip, таймаут skip.
- trellis-context: guard (нет .trellis → return), env-setup (есть .trellis →
  выставлены 3 переменные), hook wiring (mock pi.on).

## Развёртывание

1. Реализация в ruttybob (этот task).
2. `npm test` в ruttybob.
3. Ручная проверка: `cd pets/Trellis && pi` → subagent видит trellis-агентов +
   получает session-overview.
4. Миграция Trellis-проекта — отдельный task (out of scope).
