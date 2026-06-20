# PRD: Расширение trellis-context + инъекция контекста в ruttybob subagent

## Цель и ценность

Разделить текущее монолитное расширение `.pi/extensions/trellis/index.ts` (1601 строка,
смешивает generic subagent-dispatch и Trellis-специфичную инъекцию контекста) на:

1. **`extensions/trellis-context/`** (в ruttybob) — лёгкое расширение, отвечающее
   ТОЛЬКО за Trellis-специфичную инъекцию: env-сигналы для bash, инъекция контекста
   в главный агент (`before_agent_start`), setup env-контракта для subagent.
2. **`extensions/subagent/`** (в ruttybob, уже существует) — generic subagent-dispatch,
   к которому «приклеивается» способность инъекции контекста в дочерний процесс
   через env-контракт `PI_SUBAGENT_CONTEXT_CMD`.

Активация Trellis-функциональности — **только при наличии `.trellis/`** в проекте
(guard с early-return), без неё ruttybob subagent остаётся чисто generic.

## Подтверждённые факты (из исследования)

- Текущий `.pi/extensions/trellis/index.ts` = 1601 строка в одном файле.
- ruttybob `extensions/subagent/` уже существует и модульный (~1393 строки, 8 файлов):
  `index.ts`, `agents.ts`, `runner.ts`, `config.ts`, `types.ts`, `render.ts`,
  `schema.ts`, `utils.ts`.
- ruttybob subagent имеет discovery: builtin (из `extensions/subagent/agents/`),
  user (`~/.pi/agents/`), project (`findNearestProjectAgentsDir` → `.pi/agents/`),
  + кастомная директория через `--agents` flag.
- ruttybob subagent по умолчанию `agentScope: "user"` → проектные `.pi/agents/`
  НЕ грузятся, `.trellis/agents/` НЕ известен.
- Trellis-проект содержит ДВА разных пула агентов:
  - `.pi/agents/trellis-{check,implement,research}.md` (3 шт) — pi-specific,
    с `tools:` allowlist, рассчитаны на инъекцию контекста.
  - `.trellis/agents/{architect,check,implement,plan,research}.md` (5 шт) —
    channel-runtime, с `provider/labels`, для `trellis channel spawn`.
- `.trellis/scripts/get_context.py` существует (16 строк, делегирует в
  `common.git_context.main`) и кросс-платформенный — отдаёт session-overview.
- Богатый per-agent контекст (PRD + design + curated JSONL specs) сейчас зашит
  только в TypeScript (`buildContext` в trellis/index.ts) — python-скрипта НЕТ.

## Функциональные требования

### FR-1: Env-контракт ruttybob subagent (generic, без знания про Trellis)

ruttybob `extensions/subagent/` получает три новых env-переменных, читаемых
в момент исполнения (не load-time):

- `PI_SUBAGENT_AGENT_DIRS` — доп. директории с `.md` агентами, разделённые
  `path.delimiter`. Высший приоритет в discovery (как `--agents`).
- `PI_SUBAGENT_AGENT_SCOPE` — override `agentScope` (`user|project|both`).
  Имеет приоритет над settings.json.
- `PI_SUBAGENT_CONTEXT_CMD` — команда, stdout которой дописывается к
  `--append-system-prompt` каждого запускаемого субагента. Выполняется через
  `spawnSync` с timeout, stderr игнорируется, ненулевой exit → skip без краша.

### FR-1.1: Per-agent opt-out через frontmatter `trellis: false`

Агент может явно отказаться от context injection, указав в frontmatter
опциональное поле `trellis: true/false`:
- `trellis: false` → opt-out: `PI_SUBAGENT_CONTEXT_CMD` пропускается для
  этого агента.
- `trellis: true` или поле отсутствует → обычное поведение.

Назначение: агенты общего назначения (не завязанные на Trellis) не получают
лишний session-overview, экономя токены. Discovery агента не затрагивается.

### FR-2: Расширение `trellis-context` (в ruttybob/extensions/)

Фабрика расширения с early-return guard:

```
const root = findTrellisRoot(process.cwd());   // ищет ТОЛЬКО .trellis
if (!root) return;                              // вне .trellis → неактивно
```

При активации:
- выставляет env-контракт для subagent:
  - `PI_SUBAGENT_AGENT_SCOPE = existing ?? "both"`
  - `PI_SUBAGENT_AGENT_DIRS = append(<root>/.trellis/agents)`
  - `PI_SUBAGENT_CONTEXT_CMD = python3 <root>/.trellis/scripts/get_context.py`
    (только если скрипт существует)
- вешает хуки (перенесены из trellis/index.ts):
  - `session_start` → notify + `getKey`
  - `tool_call` (bash без `TRELLIS_CONTEXT_ID`) → 🟢 `export TRELLIS_CONTEXT_ID=<key>;`
  - `before_agent_start` → 🟢 inject `buildContext` + `workflowBreadcrumb`
    + `sessionOverview` в `systemPrompt`
  - `context` → `getKey`
  - `session_shutdown` → cleanup in-memory state

### FR-3: ruttybob package.json auto-discovery

ruttybob `package.json` уже указывает `"extensions": ["./extensions"]`, поэтому
`trellis-context/index.ts` подхватывается автоматически. Никаких изменений
в package.json ruttybob не требуется.

## Acceptance criteria

- [ ] ruttybob `extensions/subagent/runner.ts` дёргает `PI_SUBAGENT_CONTEXT_CMD`
      (если задана) и дописывает stdout к `--append-system-prompt` субагента.
- [ ] ruttybob `extensions/subagent/agents.ts` учитывает `PI_SUBAGENT_AGENT_DIRS`
      с высшим приоритетом.
- [ ] ruttybob `extensions/subagent/config.ts` учитывает `PI_SUBAGENT_AGENT_SCOPE`.
- [ ] Создан ruttybob `extensions/trellis-context/` (index + context + workflow +
      shared + AGENTS.md + README.md).
- [ ] trellis-context полностью неактивен вне `.trellis/` (early-return, хуки
      не вешаются, env не выставляется).
- [ ] При наличии `.trellis/`: главный агент получает `<workflow-state>` +
      `<session-overview>` + buildContext; субагенты (через ruttybob subagent)
      получают session-overview в system-prompt.
- [ ] ruttybob `npm test` проходит (новый функционал покрыт stubs/тестами).
- [ ] ruttybob AGENTS.md обновлён: добавлена ссылка на subtree-guide для
      `trellis-context/`.

## Out of scope

- Миграция Trellis-проекта (`.pi/extensions/trellis/index.ts` → подключение
  ruttybob-расширений, правка `.pi/settings.json`) — отдельный шаг после
  готовности ruttybob-стороны.
- Python-порт `buildContext` для богатого контекста в субагента (Уровень 2) —
  сейчас используется существующий `get_context.py` (session-overview,
  Уровень 1). Полная паритета по токенам — follow-up task.
- Удаление subagent-части из текущего `.pi/extensions/trellis/index.ts` —
  часть миграции (out of scope выше).
