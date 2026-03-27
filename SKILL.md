---
name: openclaw-architecture
description: >
  Аудит и настройка конфигурации OpenClaw по стандартам your organization. Используй этот скилл
  когда: (1) настраиваешь нового агента на сервере, (2) аудит конфигурации openclaw.json,
  models.json, auth-profiles.json, (3) изменяешь модели или credentials агента,
  (4) проверяешь соответствие конституции §8, (5) принц говорит «проверь конфиг»,
  «аудит openclaw», «настрой агента», «приведи в порядок». Скилл содержит правила
  архитектуры, чеклисты и антипаттерны.
---

# OpenClaw Architecture -- Стандарты конфигурации

Этот скилл описывает как правильно содержать конфигурацию OpenClaw.
Применимо ко всем серверам your organization.

---

## Карта серверов (конституция §8.1)

| Сервер | Агент | id | Роль |
|--------|-------|----|------|
| Mac mini | Сильвана | `silvana` | coordinator + creator |
| Arthas VPS | Артас | `arthas` | monitor, watchdog |
| Thrall | Тралл | `main` | coder |
| Illidan | Иллидан | `main` | devops |

Каждый сервер -- отдельный инстанс OpenClaw. Один агент на сервер.

---

## Шаг 0: Определи контекст

```bash
# Узнай агента и сервер
hostname
grep -A5 '"list"' ~/.openclaw/openclaw.json | head -10

# Узнай текущие модели
openclaw models list

# Статус системы
openclaw status
```

---

## Правило 1: Агенты в `agents.list`

Каждый агент ЯВНО прописан в `openclaw.json` -> `agents.list`.
Без `list` OpenClaw создаёт анонимный `main` -- антипаттерн.

```json
{
  "agents": {
    "list": [
      {
        "id": "silvana",
        "name": "Сильвана",
        "default": true,
        "workspace": "~/.openclaw/workspace"
      }
    ]
  }
}
```

**Проверка:**
```bash
# id и name должны совпадать с таблицей §8.1
openclaw agents
```

---

## Правило 2: Директория агента = его `id`

```
agents.list[].id: "silvana"  -->  agents/silvana/    (Mac mini)
agents.list[].id: "arthas"   -->  agents/arthas/     (Arthas VPS)
agents.list[].id: "main"     -->  agents/main/       (Thrall, Illidan)
```

**Проверка:**
```bash
ls ~/.openclaw/agents/
# Должна быть ровно 1 директория, совпадающая с id агента
```

Запрещено:
- Директория не совпадает с id (например `agents/main/` при id `silvana`)
- Директории без записи в `agents.list`
- Лишние директории (бэкапы типа `_main_backup` -- удалить после проверки)

---

## Правило 3: Два файла конфигурации моделей

| Файл | Что хранит | Роль |
|------|-----------|------|
| `openclaw.json` -> `models.providers` | Провайдеры, baseUrl, каталог моделей | Глобальный реестр |
| `openclaw.json` -> `agents.defaults` | Primary, fallbacks, алиасы, heartbeat | Дефолты агента |
| `agents/<id>/agent/models.json` | Провайдеры и модели конкретного агента | Связка агента с auth |

**Критично:** `models.json` агента обязателен. Без него все модели `Auth: no`.

**Проверка:**
```bash
# Должны быть все 6 моделей, все Auth:yes, все alias:*
openclaw models list
```

Ожидаемый результат:
```
anthropic/claude-opus-4-6                  ... Auth yes  default,configured,alias:opus
openai-codex/gpt-5.3-codex                ... Auth yes  fallback#1,configured,alias:codex
openrouter/x-ai/grok-4.1-fast             ... Auth yes  fallback#2,configured,alias:grok
openrouter/google/gemini-3.1-pro-preview   ... Auth yes  configured,alias:gemini
openrouter/google/gemini-3-flash-preview   ... Auth yes  configured,alias:gemini-flash
openrouter/moonshotai/kimi-k2.5            ... Auth yes  configured,alias:kimi
```

**Что НЕ хранить в `models.json`:**
- `apiKey` для провайдеров с OAuth в `auth-profiles.json`
- Модели не из каталога конституции §1 (например `openrouter/auto`)
- Мёртвые/просроченные ключи

---

## Правило 4: Credentials в `auth-profiles.json`

```
agents/<id>/agent/auth-profiles.json
```

**Правила:**
- Anthropic -- только OAuth token ($0). Через OpenRouter = P0 инцидент.
- OpenAI codex (GPT-5.4) -- только OAuth ($0)
- OpenRouter -- apiKey допустим (нет OAuth)
- `apiKey` в `models.json` при наличии OAuth -- ЗАПРЕЩЕНО (конфликт credentials)

**Проверка:**
```bash
# Не должно быть apiKey для anthropic или openai-codex
grep -n "apiKey" ~/.openclaw/agents/*/agent/models.json
```

---

## Правило 5: Workspace отделён от конфига

```
~/.openclaw/           # Конфиги, auth, sessions
~/.openclaw/workspace/ # Рабочая зона: AGENTS.md, память, скиллы
```

Стандартные файлы воркспейса:
- `AGENTS.md` -- инструкции агента (загружается при старте)
- `SOUL.md` -- персона и границы
- `IDENTITY.md` -- имя
- `USER.md` -- информация о пользователе
- `MEMORY.md` -- долгосрочная память
- `memory/` -- ежедневные логи
- `skills/` -- скиллы

---

## Правило 6: Shared-слой (your organization-specific)

> Кастомная архитектура your organization, не стандартная фича OpenClaw.

```
~/.openclaw/shared/               # Общая память команды
~/.openclaw/workspace/shared -> ~/.openclaw/shared   # Symlink
~/.openclaw/workspace/_shared -> ~/.openclaw/shared/_ref
```

---

## Правило 7: Алиасы моделей (§8.2)

В `openclaw.json` -> `agents.defaults.models` -- ровно 6 записей:

```json
{
  "anthropic/claude-opus-4-6": {"alias": "opus"},
  "openai-codex/gpt-5.3-codex": {"alias": "codex"},
  "openrouter/x-ai/grok-4.1-fast": {"alias": "grok"},
  "openrouter/google/gemini-3.1-pro-preview": {"alias": "gemini"},
  "openrouter/google/gemini-3-flash-preview": {"alias": "gemini-flash"},
  "openrouter/moonshotai/kimi-k2.5": {"alias": "kimi"}
}
```

Ключ = полный `provider/model-id`. Инвертированный формат (ключ = алиас) -- ЗАПРЕЩЁН.

---

## Правило 8: Бэкап ДО изменений

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d%H%M%S)
cp ~/.openclaw/agents/<id>/agent/auth-profiles.json ~/.openclaw/agents/<id>/agent/auth-profiles.json.bak.$(date +%Y%m%d%H%M%S)
```

Всегда. Без исключений. P1.

---

## Полный аудит конфигурации

Запускай при: настройке нового агента, еженедельном аудите (§8.4), после изменений конфига.

### Шаг 1: Валидация конфига

```bash
openclaw status 2>&1 | head -5
# Не должно быть "Invalid config" или "Unrecognized key"
```

### Шаг 2: Проверка агента

```bash
openclaw agents
# id и name совпадают с §8.1?
# workspace указан?
```

### Шаг 3: Проверка моделей

```bash
openclaw models list
# Все 6 моделей configured?
# Все Auth:yes?
# Все alias:* ?
```

### Шаг 4: Проверка models.json

```bash
# Нет apiKey для провайдеров с OAuth?
grep "apiKey" ~/.openclaw/agents/*/agent/models.json

# Нет модели auto или вне каталога?
grep '"id"' ~/.openclaw/agents/*/agent/models.json
```

### Шаг 5: Проверка auth-profiles

```bash
# Профили для всех провайдеров?
grep '"provider"' ~/.openclaw/agents/*/agent/auth-profiles.json
```

### Шаг 6: Проверка директорий

```bash
# Только одна директория агента, совпадающая с id?
ls ~/.openclaw/agents/
```

### Вердикт

Если все шаги PASS -- конфигурация соответствует конституции.
Если хоть один FAIL -- исправить и перепроверить.

---

## Антипаттерны

| Антипаттерн | Последствие |
|-------------|------------|
| Агент без `agents.list` | id = `main`, нет имени, нет маршрутизации |
| `apiKey` в `models.json` при OAuth | Конфликт credentials, непредсказуемый auth |
| Удалённый `models.json` агента | Все модели Auth:no |
| Модель не из каталога (`auto`) | Нарушение конституции §1 |
| Два агента на один workspace | Перезапись памяти, конфликт файлов |
| Директория agents/ != id | Агент не находит sessions и auth |
| Конфиг без бэкапа | Нет отката, нарушение P1 |
| Opus через OpenRouter | P0 инцидент |
| Инвертированные алиасы | Провайдер не резолвится, /model пикер сломан |
