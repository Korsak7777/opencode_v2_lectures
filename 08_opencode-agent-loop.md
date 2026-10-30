#AI_gen #opencode #ai-agents

# 08. Agent loop в OpenCode: когда агент останавливается и что управляет циклом

Агент — это, в сущности, `while`: модель вызывает инструменты, результаты возвращаются в контекст, модель вызывается снова. Но у любого `while` должен быть выход — и три вопроса этой заметки (они же три вопроса владельца, замкнувшие цикл доклада): **(1)** когда OpenCode останавливается — каковы критерии; **(2)** что раздувает и сжимает цикл — промпт, скиллы, модификации агента; **(3)** чем это отличается от других harness. Есть и красивая параллель с [заметкой 01](../01_how-to-read-llm-architecture.md): там генерация — цикл по одному токену, здесь harness — цикл по целому шагу.

Достоверность помечена по всему тексту: **[доки]** — официальный источник, **[код, не доки]** — исходники OpenCode, **[вывод]** — моя реконструкция из косвенных фактов. Главный пробел честно: явного единого критерия остановки в доках V2 нет — раздел «Вопрос 1» собран из косвенных формулировок и кода.

## Мини-глоссарий

- **agent loop** — цикл «модельный вызов → tool-вызовы → результат → следующий модельный вызов».
- **шаг (step)** — одна итерация цикла: модельный вызов с его tool-вызовами.
- **tool-driven continuation** — продолжение модельных вызовов, пока их продолжают tool-вызовы.
- **`steps`** — максимум модельных шагов агента; на последнем шаге инструменты отнимаются, модель просит подытожить текстом.
- **plain rejection vs rejection with feedback** — «голое» отклонение инструмента vs отклонение с причиной, которую видит модель.
- **компакция (compaction)** — замена старой части разговора саммари, чтобы сессия жила за пределами контекста модели.
- **circuit breaker** — автоматический прерыватель хода (у Codex: после серии отказов).

## Канон: Reason → Act → Observe

Каноническая рамка — ReAct: чередование рассуждений и действий, цикл Reason → Act → Observe ([ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)). Anthropic формулирует короче: агенты — это «LLMs using tools based on environmental feedback in a loop», и в цикл «common to include stopping conditions (such as a maximum number of iterations)» ([Anthropic: building effective agents](https://www.anthropic.com/research/building-effective-agents)). Вся заметка — о том, как эти stopping conditions устроены у OpenCode и его соседей.

## Петля в OpenCode V2

Формальное описание петли в доках косвенное: хук `context` «runs for the agent loop, including tool-driven continuations» ([server plugin docs](https://opencode.ai/v2/docs/build/plugins/server/)) — модельные вызовы продолжаются, пока их продолжают tool-вызовы **[доки, косвенно]**. Завершение сессии — событие `session.idle` («Session completed»), а эндпоинт `POST /api/session/{sessionID}/prompt` описан как «schedule agent-loop execution» ([API reference](https://opencode.ai/v2/docs/api/); [migrate-v1](https://opencode.ai/v2/docs/build/plugins/migrate-v1/)) **[доки]**. И заглянув под капот V1-раннера **[код, не доки, ветка v1]**: там цикл — буквально `while(true)` с break «exiting loop» и комментарием «Some providers return "stop" even when the assistant message contains tool calls. Keep the loop running…» ([prompt.ts](https://github.com/anomalyco/opencode/blob/v1/packages/opencode/src/session/prompt.ts)).

```text
          пользовательский ввод (сбрасывает счётчик steps)
                   |
                   v
          +------------------+
    +---->|  МОДЕЛЬНЫЙ ВЫЗОВ |
    |     +------------------+
    |           |
    |           |  tool-вызовы?  -- нет -->  session.idle
    |           v                                («Session completed»)
    |     +------------------+
    |     |    ИНСТРУМЕНТЫ   |     (Code Mode может свернуть
    |     +------------------+      несколько вызовов в одну
    |           |                   программу - заметка 07)
    +-----------+
    результат попадает в контекст ->
    следующий модельный вызов (tool-driven continuation)
```

## Вопрос 1: критерии остановки

Прямой формулы «сессия завершается, когда X» в доках нет. Реконструкция даёт **четыре механизма [вывод]**:

| # | Механизм | Что происходит | Достоверность |
|---|---|---|---|
| а | модель перестала звать инструменты | ответ без tool-вызовов → событие `session.idle` | [доки, косвенно] |
| б | исчерпан `steps` | инструменты отнимаются, модель принуждена подытожить текстом; новый ввод пользователя сбрасывает счётчик | [доки] |
| в | plain rejection | «ends the current step» — конец текущего шага | [доки], дословно |
| г | терминальная ошибка модели | после политики retry — циклу нечем продолжаться | [доки] + [вывод] |

Разбор каждого.

**`steps` — лимит с мягким финалом [доки].** [Agents docs](https://opencode.ai/v2/docs/agents/): `steps` — «positive maximum number of model steps»; «On the final step, OpenCode removes tools and asks the model to summarize in text. New user input resets the allowance». Схема конфига ([config.json](https://opencode.ai/config.json)) говорит то же: «Maximum number of agentic iterations before forcing text-only response»; `maxSteps` и `experimental.continue_loop_on_deny` помечены deprecated. Важно: исчерпание — **не ошибка**, а принудительный текстовый финал. Механика последнего шага видна в коде **[код, не доки]**: инструменты не материализуются, ставится `toolChoice: "none"`, модели уходит промпт «MAXIMUM STEPS REACHED… Tools are disabled until next user input. Respond with text only» ([issue #45328](https://github.com/anomalyco/opencode/issues/45328); [max-steps.ts](https://github.com/anomalyco/opencode/blob/dev/packages/core/src/session/runner/max-steps.ts)). Дефолт `steps` в доках не указан — не будем его выдумывать.

**Deny — два разных исхода [доки].** Дословно из [server plugin docs, Permissions](https://opencode.ai/v2/docs/build/plugins/server/): «A deny rule or a rejection with feedback rejects with a Tool.Error that carries the reason, so the model sees it and continues. A plain rejection ends the current step». То есть deny-правило и отклонение с фидбеком цикл **не рвут** — модель видит причину и продолжает; «голое» отклонение кончает текущий шаг. Что «конец шага» значит для цикла в целом, в доках не раскрыто — оставляем дословную формулировку с этой оговоркой. Родственные факты: явный настроенный deny финален и хук `evaluate` не вызывается (формулировка «does not invoke the hook» — тоже из [server plugin docs, Permissions](https://opencode.ai/v2/docs/build/plugins/server/); общий контекст прав — [permissions docs](https://opencode.ai/v2/docs/permissions/); механика — в [заметке 05](05_opencode-commands-hooks.md)); а V1-флаг `continue_loop_on_deny` в V2 «accepted but unsupported» — игнорируется с warning ([migrate-v1](https://opencode.ai/v2/docs/migrate-v1/); переименования — в [заметке 02](02_opencode-v1-vs-v2.md)).

**Компакция — не остановка [доки].** [Compaction docs](https://opencode.ai/v2/docs/compaction/): компакция — «replacing older conversation with a summary, so the session can keep going past the model's context limit» — цикл продолжается. Авто-компакция «pauses the session» — пауза, не завершение; `experimental.background_compaction` (default false, знакомый по [заметке 03](03_opencode-architecture-config.md) зонтик `experimental.*`) — «the session keeps going; the summary is held». Переполнение контекста: дословная формула из [compaction docs](https://opencode.ai/v2/docs/compaction/) (Limits → Recovery) — «compacted and retried once. A second rejection is returned as an error».

**Retry — политика, не критерий [доки].** Retry-хук (тема [заметки 05](05_opencode-commands-hooks.md)) касается только сбоев провайдера («Override the retry decision for a provider failure»), может сделать ошибку повторяемой или наложить вето, но «built-in maximum attempt count remains a hard limit». Это регулирование повторов внутри витка, а не условие выхода из цикла.

## Вопрос 2: что управляет размером и числом циклов

| Рычаг | Эффект на цикл | Достоверность |
|---|---|---|
| **AGENTS.md / инструкции** | глобальный файл + все от рабочей директории вверх грузятся сразу, вложенные — лениво по ходу ([instructions docs](https://opencode.ai/v2/docs/instructions/)); изменения инжектируются «before the next model request», «at the next step boundary without queuing a message turn» — **не добавляют модельный виток**; `CLAUDE.md` не используется | [доки] |
| **Скиллы** | на каждом шаге — лёгкий листинг «only the ID, name, and description, not the full Markdown body»; тело догружается инструментом `skill` («Adds the Markdown body… to the conversation») ([skills docs](https://opencode.ai/v2/docs/skills/)). **[вывод]** загрузка тела = +1 виток цикла и постоянное место в контексте; качество `description` определяет выбор и число ложных витков | [доки] + [вывод] |
| **Поля агента** | system prompt + model + permissions + steps ([agents docs](https://opencode.ai/v2/docs/agents/); поля — в [заметке 04](04_opencode-multiagent.md)): промпт и права решают, как часто модель зовёт инструменты; `steps` — жёсткий потолок | [доки] |
| **Субагенты** | «Subagents run with fresh context in foreground or background child sessions» ([agents docs](https://opencode.ai/v2/docs/agents/)) — собственный цикл с чистым контекстом; `experimental.subagent_depth` — «Defaults to 1, so a subagent cannot start another subagent» ([config docs](https://opencode.ai/v2/docs/config/)) | [доки] |
| **Code Mode** | `execute` — «parallel independent calls and for processing results without adding every intermediate value to the model context» ([tools docs](https://opencode.ai/v2/docs/tools)) — меньше витков и меньше контекста на виток (разбор — в [заметке 07](07_opencode-codemode.md)) | [доки] |

Куда всё это попадает: порядок сборки промпта **[доки]** — system → Code Mode guidance → MCP/skill guidance → AGENTS.md + instructions → дата/окружение → session entries ([instructions](https://opencode.ai/v2/docs/instructions/)).

Свести в «регуляторы» **[вывод]** (в доках в таком виде не сформулированы): `steps` — жёсткий потолок; инструкции и описания — меньше ложных витков; Code Mode — схлопывает лестницу вызовов; субагенты — выносят цикл из родительского контекста. Исторический штрих: ещё в сетапе владельца v1 ([Установка openCode & OMO](../../../zettelkasten/AI_start.md)) короткий промпт осознавался как «получается диалог… задача выполняется в несколько итераций» — agent loop был всегда, V2 лишь сделала его настраиваемым.

## Вопрос 3: три harness

| Механизм | OpenCode V2 | Claude Code | Codex |
|---|---|---|---|
| Формула цикла | модельные вызовы, продолжаемые tool-вызовами; финал — `session.idle` [доки, косвенно] | «Receive prompt → Evaluate and respond → Execute tools → Repeat… until it produces a response with no tool calls → Return result»; полный цикл = turn [CC-доки] | работает в цикле: вызывает модель и выполняет действия по её выходу; завершение — выполнение задачи или отмена пользователем [пересказ Codex-док] ([Codex manual](https://developers.openai.com/codex/codex-manual.md)) |
| Лимит ходов | `steps` → мягкий финал: summarize текстом [доки] | `max_turns` (SDK, default no limit; исчерпание → `error_max_turns`), `--max-turns` для headless/CI, `--max-budget-usd`; интерактивным сессиям задокументированного лимита нет [CC-доки] | CLI-лимит ходов отсутствует осознанно — [issue #12336](https://github.com/openai/codex/issues/12336): «unlikely that we'd expose a --max-turns CLI option», closed as not planned |
| «Не дай остановиться» | аналога Stop-хука в session-хуках нет — только событие `session.idle` [вывод-оговорка] | Stop-хук: `decision: "block"` «prevents Claude from stopping» + reason; защита — `stop_hook_active` и «8-consecutive-continuation cap» со сбросом при любом tool-вызове [CC-доки; версионно-текучее — на октябрь 2026] | — |
| Границы вместо лимита | permissions/deny как границы шага [доки] | read-only параллелятся, мутирующие — последовательно [CC-доки] | sandbox (read-only / workspace-write / danger-full-access) × approval policy (untrusted / on-request / never / granular); circuit breaker — «In the current open-source implementation» — прерывание хода после 3 подряд denials или 10 за последние 50 reviews [Codex-доки] |
| Хуки по каденциям | session-уровень (`prompt`, `context`, …), per-tool, retry [доки] | per session (SessionStart/End), per turn (UserPromptSubmit, Stop, StopFailure), per tool call (PreToolUse/PostToolUse) [CC-доки] | авто-ревью как встроенный перехватчик [Codex-доки] |

Источники колонок: Claude Code — [agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop) и [hooks](https://code.claude.com/docs/en/hooks); Codex — [Codex manual](https://developers.openai.com/codex/codex-manual.md) и разделы agent-approvals-security / sandboxing-auto-review.

**Философии остановки [вывод]:** OpenCode — лимит с мягким финалом (summarize); Claude Code — лимит с ошибкой плюс уникальный режим «не дай остановиться» (Stop-хук с предохранителем); Codex — сознательный отказ от лимита в пользу sandbox × approval + circuit breaker. Прямая аналогия «Stop-хук ↔ approval policy» некорректна: общее только то, что остановка — управляемое событие.

Контекст для Code Mode: Cloudflare формулирует аргумент так — «the output of each tool call must feed into the LLM's neural network, just to be copied over to the inputs of the next call… When the LLM can write code, it can skip all that» ([Cloudflare code mode](https://blog.cloudflare.com/code-mode/); [developers.cloudflare.com](https://developers.cloudflare.com/agents/tools/codemode/)) — конкретных цифр кратности в первоисточниках нет, не приписываем.

## Agent loop одним взглядом

| Вопрос | Ответ |
|---|---|
| Что такое цикл | модельный вызов → tool-вызовы → результат → следующий вызов; «agent-loop execution» стартует по `POST /api/session/{sessionID}/prompt` [доки] |
| Когда конец | модель без tool-вызовов → `session.idle`; `steps` → summarize; plain rejection → конец шага; терминальная ошибка после retry — четыре механизма [вывод] |
| `steps` | потолок модельных шагов; финальный шаг — без инструментов, текстовый итог; новый ввод сбрасывает; дефолт в доках не указан |
| Deny | deny/rejection-with-feedback → модель видит причину и продолжает; plain rejection → «ends the current step» (смысл для цикла в доках не раскрыт) |
| Компакция | не остановка: пауза/фон, сессия продолжается; переполнение — компакт, retry once, затем ошибка |
| Что раздувает цикл | загрузка тела скилла (+1 виток [вывод]), ложные витки от слабых description |
| Что сжимает цикл | Code Mode (меньше витков и контекста), субагенты (цикл в чистом child-контексте), точные инструкции (без лишних витков) |
| vs Claude Code | CC: turn-based, `max_turns` с ошибкой, Stop-хук «не дай остановиться» (с cap) |
| vs Codex | нет лимита ходов; sandbox × approval policy + circuit breaker |
| Системная параллель | генерация — цикл по токену (заметка 01); harness — цикл по шагу (эта заметка) |

### Ссылки внутренние

- [07. Code Mode в OpenCode V2](07_opencode-codemode.md) - лестница вызовов vs одна программа: главный рычаг сжатия цикла из вопроса 2.
- [05. Команды и хуки в OpenCode V2](05_opencode-commands-hooks.md) - хуки `context` и `retry`, финальность явного deny - инструменты наблюдения за петлёй.
- [04. Мультиагенты в OpenCode V2](04_opencode-multiagent.md) - поля агента (`steps` в том числе) и субагенты как вынос цикла в чистый контекст.
- [03. Установка и настройка OpenCode V2](03_opencode-architecture-config.md) - зонтик `experimental.*`: `subagent_depth`, `background_compaction`, `continue_loop_on_deny` (deprecated).
- [02. OpenCode v1 vs V2](02_opencode-v1-vs-v2.md) - переименование `maxSteps`→`steps` и судьба `continue_loop_on_deny` при миграции.
- [01. Как читать архитектуру LLM-моделей](../01_how-to-read-llm-architecture.md) - параллель уровней: генерация - цикл по токену, harness - цикл по шагу.
- [Установка openCode & OMO](../../../zettelkasten/AI_start.md) - историческая точка: уже в сетапе v1 задача осознавалась как «диалог в несколько итераций».

### Ссылки внешние

**Канон:** [ReAct: Synergizing Reasoning and Acting (Yao et al., 2022)](https://arxiv.org/abs/2210.03629) — Reason→Act→Observe. [Anthropic: building effective agents](https://www.anthropic.com/research/building-effective-agents) — «LLMs using tools based on environmental feedback in a loop», stopping conditions.

**Документация OpenCode V2:** [Server-плагины: хук context, deny-семантика, retry](https://opencode.ai/v2/docs/build/plugins/server/). [Agents: `steps` и summarize-финал](https://opencode.ai/v2/docs/agents/). [Compaction: Limits → Recovery — формула переполнения](https://opencode.ai/v2/docs/compaction/). [Config: background_compaction](https://opencode.ai/v2/docs/config/). [Схема config.json](https://opencode.ai/config.json) — формулировка steps, deprecated-поля. [Permissions: финальность deny](https://opencode.ai/v2/docs/permissions/). [Instructions: AGENTS.md, ленивая загрузка](https://opencode.ai/v2/docs/instructions/). [Skills: листинг per step, tool `skill`](https://opencode.ai/v2/docs/skills/). [Tools: Code Mode `execute`](https://opencode.ai/v2/docs/tools). [API reference: эндпоинт prompt («schedule agent-loop execution»)](https://opencode.ai/v2/docs/api/). [Migrate-v1 для плагинов: событие session.idle](https://opencode.ai/v2/docs/build/plugins/migrate-v1/). [Гайд миграции V1 → V2](https://opencode.ai/v2/docs/migrate-v1/) — судьба `continue_loop_on_deny`.

**Код OpenCode [код, не доки]:** [prompt.ts (ветка v1) — раннер: `while(true)`, «Keep the loop running»](https://github.com/anomalyco/opencode/blob/v1/packages/opencode/src/session/prompt.ts). [Issue #45328 — механика MAXIMUM STEPS REACHED](https://github.com/anomalyco/opencode/issues/45328). [max-steps.ts — реализация последнего шага](https://github.com/anomalyco/opencode/blob/dev/packages/core/src/session/runner/max-steps.ts).

**Claude Code:** [Agent loop (SDK)](https://code.claude.com/docs/en/agent-sdk/agent-loop) — turn, no-tool-calls, `max_turns`. [Hooks](https://code.claude.com/docs/en/hooks) — каденции, Stop-хук и cap.

**Codex:** [Codex manual](https://developers.openai.com/codex/codex-manual.md) — работа в цикле; разделы agent-approvals-security и sandboxing/auto-review — circuit breaker. [Issue #12336 (openai/codex)](https://github.com/openai/codex/issues/12336) — отказ от `--max-turns`, closed as not planned.

**Cloudflare:** [Code mode (блог)](https://blog.cloudflare.com/code-mode/) и [developers.cloudflare.com: codemode](https://developers.cloudflare.com/agents/tools/codemode/) — аргумент «skip all that» без цифр кратности.
