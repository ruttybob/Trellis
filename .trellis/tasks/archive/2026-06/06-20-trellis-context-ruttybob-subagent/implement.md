# Implement: Расширение trellis-context + инъекция в ruttybob subagent

## Порядок реализации

### Шаг 1: ruttybob subagent — env-контракт + per-agent opt-out (generic хуки)

- [x] `extensions/subagent/agents.ts`:
      - расширить `AgentConfig` полем `trellis?: boolean`;
      - в `loadAgentsFromDir` парсить `frontmatter.trellis` (трим+lower,
        `=== "true"` → true, `=== "false"` → false, прочее/отсутствует → undefined);
      - после блока `customDir` добавить чтение `PI_SUBAGENT_AGENT_DIRS`
        (split по `path.delimiter`), каждая валидная директория грузится как
        `source: "user"` с высшим приоритетом в `agentMap`.
- [x] `extensions/subagent/config.ts`: в `buildConfig` добавить override scope
      из `PI_SUBAGENT_AGENT_SCOPE` (приоритет над merged settings).
- [x] `extensions/subagent/runner.ts`: в `runSingleAgent` перед финальной
      сборкой args дёргать `PI_SUBAGENT_CONTEXT_CMD` через `spawnSync`
      (timeout 3000ms, shell, наследовать env). **Guard per-agent:**
      пропускать если `agent.trellis === false`. Ненулевой exit / таймаут /
      пустой stdout → skip. Успешный stdout → temp file через существующую
      `writePromptToTempFile("context", stdout)`, добавить как ещё один
      `--append-system-prompt`, зарегистрировать cleanup в `finally`.

### Шаг 1.1: Переименование tool name `subagent` → `trellis_subagents` при наличии `.trellis/` (вариант B) — ✅ ВЫПОЛНЕНО

Подход **B**: subagent сам проверяет наличие `.trellis/` через `findTrellisRoot`
(детерминированно, без гонки порядка загрузки расширений). Не зависит от
выполнения фабрики trellis-context — импорт модуля не запускает default-фабрику.

- [x] `extensions/subagent/utils.ts`: добавлен общий хелпер `resolveSubagentToolName(cwd)`
      — `findTrellisRoot(cwd) ? "trellis_subagents" : "subagent"`. Импорт
      `findTrellisRoot` из `../trellis-context/index.js` (УЖЕ экспортируется,
      дубликат функции НЕ добавлялся). Зависимость однонаправленная
      (trellis-context НЕ импортирует subagent) — цикла нет.
- [x] `extensions/subagent/index.ts`: `const TOOL_NAME = resolveSubagentToolName();`
      на верхнем уровне фабрики; `name: TOOL_NAME` в `registerTool`.
      Error-prefix литералы `"subagent"` / `` `subagent chain step ${i+1}` ``
      в `resolvePromptFile` (строки ~144, 150) и description у `discover_agents`
      НЕ тронуты — это не имя tool.
- [x] `extensions/subagent/render.ts`: `const RENDER_TITLE = resolveSubagentToolName() + " ";`
      на top-level модуля; `theme.bold(RENDER_TITLE)` в 2 местах renderCall
      (chain + single). Заголовок TUI консистентен с именем tool.

**Тесты (ruttybob vitest) — `npm test` зелёный (116 файлов, 1567 тестов):**

- Важное уточнение к изначальному плану: **ruttybob сам является
  Trellis-проектом** (`.trellis/` в корне). Поэтому `process.cwd()` во время
  `npm test` имеет `.trellis` → `findTrellisRoot` вернул бы ruttybob → имя стало
  бы `trellis_subagents`. Изначальное предположение «существующие тесты
  запускаются без `.trellis`» оказалось неверным. Решение — зафиксировать
  generic-путь явно в существующих тестах:
  - `tests/subagent/promptFile.test.ts`, `discover.test.ts`: эти тесты УЖЕ мокают
    `utils.js` целиком — добавлен stub `resolveSubagentToolName: () => "subagent"`.
    Тестируют generic promptFile/discovery, не условие `.trellis`.
  - `tests/subagent/render.test.ts`, `render-spinner.test.ts`: импортируют
    реальный `render.ts` (RENDER_TITLE top-level) — добавлен
    `vi.mock("trellis-context", () => ({ findTrellisRoot: () => null }))` для
    детерминизма заголовка (`"subagent "`).
- [x] `tests/subagent/tool-name.test.ts` (НОВЫЙ, 4 теста): управляемый mock
  `findTrellisRoot` через `vi.hoisted` — проверяет `registerTool.name === "subagent"`
  (root=null), `=== "trellis_subagents"` (root=fake), стабильность при повторном
  вызове фабрики, и согласованность `resolveSubagentToolName` с именем tool.

### Шаг 2: ruttybob trellis-context — новое расширение

- [x] `extensions/trellis-context/shared.ts`: перенести `readText`, `exists`,
      `splitFM`, `stripFM`, `hash`, `shellQuote`, `cmdHasTrellisCtx`, `str`,
      `isObj`, `lookupStr`, `callStr`. Типизировать под `ExtensionAPI`.
- [x] `extensions/trellis-context/context.ts`: перенести `TRELLIS_AGENT_JSONL`,
      `contextKey`, `sessionHasTask`, `readTaskDir`, `adoptKey`, `buildContext`.
      В `buildContext` убрать зависимость от subagent-функций.
- [x] `extensions/trellis-context/workflow.ts`: перенести `WF_RE`,
      `workflowBreadcrumb`, `sessionOverview` (spawnSync get_context.py).
- [x] `extensions/trellis-context/index.ts`: фабрика с
      `findTrellisRoot` (только `.trellis`), guard `if (!root) return;`,
      `setupSubagentContract(root)` (выставляет 3 env), `registerHooks`
      (session_start, tool_call bash, before_agent_start, context,
      session_shutdown), per-turn кэш `getTurnCtx`.
- [x] `extensions/trellis-context/AGENTS.md`: subtree-guide в стиле соседних
      (`extensions/subagent/AGENTS.md`): назначение, env-контракт, guard,
      миграция из trellis/index.ts.
- [x] `extensions/trellis-context/README.md`: user-facing обзор, активация
      при .trellis/, что инъектируется.

### Шаг 3: ruttybob верхнеуровневые правки

- [x] `AGENTS.md` (ruttybob root): добавить bullet в "Subtree guides" со
      ссылкой на `extensions/trellis-context/AGENTS.md`.
- [x] `package.json`: проверить, что `"extensions": ["./extensions"]`
      покрывает trellis-context (ничего менять не должно).

### Шаг 4: Тесты (ruttybob vitest)

- [x] `tests/stubs/`: убедиться что `spawnSync` есть в stub node:child_process
      (используется в runner.ts + workflow.ts). Если нет — добавить минимальный.
- [x] Тесты `runner.ts`: context-cmd stdout → args, ненулевой exit skip,
      таймаут skip, пустой stdout skip, `agent.trellis === false` → context-cmd
      НЕ вызывается.
- [x] Тесты `agents.ts`: поле `trellis` парсится корректно (true/false/absent),
      env dir добавляется, приоритет, невалидный путь skip.
- [x] Тесты trellis-context `index.ts`: guard (нет .trellis → return без
      вызова pi.on), env-setup (есть .trellis → 3 env выставлены).

### Шаг 5: Валидация

- [x] `npm test` в ruttybob — зелёный.
- [x] Линтер/typecheck ruttybob (если есть) — зелёный.
- [x] Ручная проверка: `cd pets/Trellis && pi` → вызов `discover_agents` /
      subagent видит trellis-агентов (`trellis-implement` из .pi + `implement`
      из .trellis); subagent получает session-overview в system-prompt; главный
      агент видит `<workflow-state>` + `<session-overview>`.
      **Note (2026-06-20, archive):** ручная проверка НЕ проводилась; принято к
      архивации на основании зелёных автотестов (1590 passed) + код в проде
      ruttybob origin/main. Ручной smoke-test → follow-up при миграции Trellis-проекта.

## Validation commands

```sh
cd /Users/sergeykostrov/pets/ruttybob
npm test                      # vitest unit
npm run lint 2>/dev/null || true
npm run typecheck 2>/dev/null || true
```

## Рисковые места / rollback

- ruttybob subagent — существующее и стабильное расширение. Изменения
  аддитивные (новые env), обратно совместимы. Rollback — revert конкретных
  коммитов в `agents.ts`/`config.ts`/`runner.ts`.
- trellis-context — НОВОЕ расширение, не трогает существующий код. Rollback —
  `rm -rf extensions/trellis-context` + убрать bullet из AGENTS.md.
- auto-discovery ruttybob подхватит trellis-context автоматически — это
  нормально, guard защищает не-Trellis проекты.

## Out of scope (отдельный task)

- Миграция Trellis-проекта: убрать `.pi/extensions/trellis/index.ts`,
  подключить ruttybob-extensions в `.pi/settings.json`.
- Python-порт `buildContext` для богатого контекста в субагента.
