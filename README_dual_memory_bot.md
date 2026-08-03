# 🤖 Бот с короткой и длинной памятью (aiogram + OpenAI + ChromaDB)

Telegram-бот, объединяющий **короткую память** (in-memory история диалога)
и **длинную память** (RAG по загруженным документам). Бот ведёт осмысленную
беседу, помня контекст, и отвечает на вопросы, ссылаясь на релевантные фрагменты
PDF / TXT / DOCX-файлов.

Архитектура двухуровневая:

1. **Short memory** — `deque` с ограничением длины для каждого пользователя.
   Хранит пары `user / assistant` и передаёт их в LLM как историю диалога.
2. **Long memory** — загруженный документ разбивается на фрагменты,
   векторизуется через `text-embedding-3-small`, сохраняется в ChromaDB,
   а `document_id` привязывается к пользователю в SQLite. При вопросе
   извлекаются топ-K релевантных фрагментов и подаются в LLM вместе с историей.


---

## ✨ Возможности

- 💬 **Диалог с ИИ** — поддерживает осмысленную беседу с историей.
- 📎 **Загрузка документов** — PDF, TXT и DOCX прямо в Telegram-чате.
- 🔍 **RAG-поиск** — семантический поиск по фрагментам документа (ChromaDB, cosine).
- 🧠 **Двухуровневая память** — короткая (deque) + длинная (вектор + SQLite).
- 🧑‍🤝‍🧑 **Персональная память** — каждый пользователь имеет свой активный документ
  и собственную историю диалога.
- ⚙️ **Гибкая настройка** через `.env` (токен, ключ API, модели, параметры чанкинга).
- 🛡️ **Обработка ошибок** — все сбои логируются, пользователь получает понятное
  сообщение.


---

## 🧱 Стек технологий

| Технология | Назначение |
|---|---|
| [Python 3.10+](https://www.python.org/) | Язык разработки |
| [aiogram 3.x](https://docs.aiogram.dev/) | Асинхронный фреймворк Telegram-бота |
| [openai](https://pypi.org/project/openai/) | SDK для эмбеддингов и генерации ответов |
| [chromadb](https://www.trychroma.com/) | Векторное хранилище для поиска по фрагментам |
| [pypdf](https://pypi.org/project/pypdf/) | Извлечение текста из PDF |
| [python-docx](https://pypi.org/project/python-docx/) | Извлечение текста из DOCX |
| [sqlite3](https://docs.python.org/3/library/sqlite3.html) | Привязка активного документа к пользователю |
| [python-dotenv](https://pypi.org/project/python-dotenv/) | Загрузка переменных окружения из `.env` |


---

## 🔄 Как это работает

1. **Запуск** — бот инициализирует SQLite-базу, подключается к ChromaDB,
   начинает long-polling через `aiogram`.

2. **Загрузка документа** — пользователь отправляет PDF, TXT или DOCX.
   Бот:
   - скачивает файл из Telegram и сохраняет на диск
     (`data/documents/<user_id>/<document_id>_<filename>`);
   - извлекает текст (`pypdf`, `python-docx` или `read_text`);
   - нормализует и разбивает на фрагменты (`CHUNK_SIZE=500`, `CHUNK_OVERLAP=80`);
   - создаёт эмбеддинги batch-запросом через OpenAI API;
   - сохраняет фрагменты в ChromaDB с метаданными
     (`user_id`, `document_id`, `filename`, `chunk_index`);
   - привязывает `document_id` к пользователю в SQLite.

3. **Текстовое сообщение** — бот:
   - извлекает историю диалога из short memory (`deque`);
   - запрашивает `document_id` из SQLite;
   - если документ есть — векторизует вопрос и ищет top-5 релевантных
     фрагментов в ChromaDB (с `where`-фильтром по `user_id` и `document_id`);
   - формирует адаптивный системный промпт (с контекстом / без контекста);
   - отправляет в LLM: `system` → `history` → `user (context + question)`;
   - сохраняет ответ в short memory.

4. **Сброс** — команда `/new` очищает историю диалога и сбрасывает
   `document_id` в SQLite.


---

## 🚀 Установка и запуск

### 1. Открыть проект



### 2. Создать виртуальное окружение (рекомендуется)

```bash
python3 -m venv .venv
source .venv/bin/activate      # macOS / Linux
# .venv\Scripts\activate       # Windows
```

### 3. Установить зависимости

```bash
pip install aiogram openai chromadb pypdf python-docx python-dotenv
```

### 4. Настроить `.env`

В корне директории есть файл-шаблон `.env.example`. Скопируйте его в `.env`
и заполните реальными значениями:

```bash
cp .env.example .env
```

Затем откройте `.env` и укажите свои ключи:

```env
BOT_TOKEN=your_bot_token_here
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_BASE_URL=https://openai.api.proxyapi.ru/v1
OPENAI_RESPONSE_MODEL=gpt-4o-mini
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
LOG_LEVEL=INFO
SHORT_MEMORY_LIMIT=40
```

> Если `OPENAI_BASE_URL` пуст — клиент использует стандартный эндпоинт OpenAI.
> Для доступа из России удобно указать URL ProxyAPI или другого провайдера.

### 5. Запустить бота

```bash
python bot_dual_memory.py
```

При успешном запуске создаются каталоги `data/documents/`, `data/chroma/`
и `data/bot.sqlite3`, а в консоли появятся логи aiogram о начале polling.


---

## 🔑 Где взять ключи

| Ключ | Где получить |
|---|---|
| `BOT_TOKEN` | У [@BotFather](https://t.me/BotFather): `/newbot` |
| `OPENAI_API_KEY` | В личном кабинете OpenAI-совместимого провайдера |


---

## ⚙️ Переменные окружения

Значения читаются из `.env` через `python-dotenv`.

| Переменная | Обязательна | По умолчанию | Описание |
|---|---|---|---|
| `BOT_TOKEN` | ✅ | — | Токен Telegram-бота от BotFather |
| `OPENAI_API_KEY` | ✅ | — | API-ключ для доступа к LLM и эмбеддингам |
| `OPENAI_BASE_URL` | ❌ | — | Базовый URL API-провайдера |
| `OPENAI_RESPONSE_MODEL` | ❌ | `gpt-4o-mini` | Модель для генерации ответов |
| `OPENAI_EMBEDDING_MODEL` | ❌ | `text-embedding-3-small` | Модель для создания эмбеддингов |
| `LOG_LEVEL` | ❌ | `INFO` | Уровень логирования (`DEBUG` / `INFO` / `WARNING` / ...) |
| `SHORT_MEMORY_LIMIT` | ❌ | `40` | Максимальное число сообщений в истории диалога (short memory) |


---

## ⚙️ Параметры чанкинга и поиска

Хардкоднуты в начале файла, легко менять под задачу:

| Параметр | По умолчанию | Описание |
|---|---|---|
| `CHUNK_SIZE` | `500` | Размер каждого фрагмента в символах |
| `CHUNK_OVERLAP` | `80` | Перекрытие между соседними фрагментентами |
| `TOP_K` | `5` | Количество релевантных фрагментов в ответе |

> `CHUNK_OVERLAP` всегда должен быть меньше `CHUNK_SIZE` и неотрицательным.
> `TOP_K` влияет на точность ответа и расход токенов — чем больше, тем больше
> контекст, но и больше cost.


---

## 💬 Команды бота

| Команда / действие | Действие |
|---|---|
| `/start` | Приветствие и краткая справка |
| `/help` | Подробная инструкция по использованию |
| `/new` | Сброс текущего документа и очистка истории диалога |
| PDF / TXT / DOCX | Сохранение, извлечение текста и векторная индексация |
| любое текстовое сообщение | Вопрос по активному документу или продолжение беседы |


---

## 🧠 Архитектура памяти

### Short memory (короткая память)

**Хранилище:** `conversation_memory: dict[int, deque[dict]]` — in-memory.

```python
conversation_memory: dict[int, deque[dict]] = defaultdict(
    lambda: deque(maxlen=SHORT_MEMORY_LIMIT),
)
```

- Ключ — `user_id` (Telegram ID).
- Значение — `deque(maxlen=40)` с попарными сообщениями `{"role": ..., "content": ...}`
  в формате OpenAI.
- При переполнении старые сообщения автоматически вытесняются.
- Сбрасывается при `/new` и при перезапуске бота (только in-memory).

### Long memory (длинная память)

Состоит из трёх слоёв:

| Слой | Технология | Назначение |
|---|---|---|
| **SQLite** | `sqlite3` | Привязка активного `document_id` к `user_id` |
| **ChromaDB** | `chromadb.PersistentClient` | Векторное хранилище эмбеддингов с cosine-поиском |
| **Файлы** | `pathlib` / `open()` | Сырые документы на диске |

#### SQLite — таблица `user_sessions`

| Поле | Тип | Описание |
|---|---|---|
| `user_id` | INTEGER (PK) | ID Telegram-пользователя |
| `document_id` | TEXT | ID текущего активного документа |
| `updated_at` | TIMESTAMP | Время последнего обновления |

Функции:
- `set_current_document(user_id, document_id)` — INSERT … ON CONFLICT DO UPDATE
- `get_current_document(user_id)` — SELECT, возвращает `document_id` или `None`
- `reset_current_document(user_id)` — устанавливает `document_id = NULL`

#### ChromaDB — коллекция `document_memory`

- Метрика: `cosine` (`hnsw:space = cosine`).
- Каждый документ-фрагмент содержит:
  - `ids` — `f"{document_id}_{index}"`;
  - `documents` — текст фрагмента;
  - `embeddings` — вектор `text-embedding-3-small`;
  - `metadatas` — `user_id`, `document_id`, `filename`, `chunk_index`.

#### Поиск релевантных фрагментов

```python
result = collection.query(
    query_embeddings=[question_embedding],
    n_results=TOP_K,                     # 5
    where={
        "$and": [
            {"user_id": str(user_id)},   # только этот пользователь
            {"document_id": document_id}, # только этот документ
        ]
    },
)
```


---

## 📦 Структура данных

```
Папка проекта/
├── bot_dual_memory.py         # Основной файл бота (этот README)
├── README_dual_memory_bot.md  # Этот файл
├── .env                       # Переменные окружения (не публикуется)
├── .env.example               # Шаблон .env с placeholder-значениями
├── data/                      # Создаётся автоматически
│   ├── bot.sqlite3            # SQLite: user_sessions
│   ├── chroma/                # ChromaDB (PersistentClient)
│   │   └── document_memory/   # Коллекция эмбеддингов
│   └── documents/
│       └── <user_id>/         # Сохранённые файлы пользователей
│           └── <document_id>_<filename>
└── .venv/                     # Виртуальное окружение (если создано)
```


---

## 📄 Поддерживаемые форматы файлов

| Формат | Библиотека | Примечание |
|---|---|---|
| `.pdf` | `pypdf` | Извлекает текст со всех страниц |
| `.docx` | `python-docx` | Учитывает параграфы и таблицы |
| `.txt` | `Path.read_text` | UTF-8 с `errors="ignore"` |

Имена файлов очищаются функцией `sanitize_filename`, чтобы исключить
небезопасные символы (включая пробелы, кириллические символы и пути `../`).


---

## 🏗️ Архитектура функций

| Функция | Асинхронность | Описание |
|---|---|---|
| `initialize_database()` | синхронная | Создаёт таблицу `user_sessions` при первом запуске |
| `set_current_document()` | синхронная | Сохраняет / обновляет `document_id` для пользователя |
| `get_current_document()` | синхронная | Возвращает `document_id` или `None` |
| `reset_current_document()` | синхронная | Сбрасывает `document_id` в `NULL` |
| `sanitize_filename()` | синхронная | Очищает имя файла от опасных символов |
| `extract_text_from_pdf()` | синхронная | Извлекает текст из PDF через `pypdf` |
| `extract_text_from_docx()` | синхронная | Извлекает текст из DOCX (параграфы + таблицы) |
| `load_document()` | синхронная | Универсальная загрузка по расширению + нормализация |
| `split_text_into_chunks()` | синхронная | Делит текст на фрагменты с перекрытием |
| `embed_and_index_chunks()` | `async` | Создаёт эмбеддинги batch-запросом и сохраняет в ChromaDB |
| `retrieve_relevant_chunks()` | `async` | Векторный поиск top-K релевантных фрагментов |
| `generate_answer()` | `async` | Формирует запрос к LLM: адаптивный system + history + context + question |
| `save_and_index_document()` | `async` | Полный цикл: скачивание → текст → чанкинг → эмбеддинги → индексация |
| `command_start_handler()` | `async` | Хендлер `/start` |
| `command_help_handler()` | `async` | Хендлер `/help` |
| `command_new_handler()` | `async` | Хендлер `/new` — сброс документа и истории |
| `document_handler()` | `async` | Хендлер загрузки документа (`F.document`) |
| `text_message_handler()` | `async` | Главный хендлер текстовых сообщений — точка слияния short и long memory |
| `unsupported_message_handler()` | `async` | Хендлер неподдерживаемых типов сообщений |
| `error_handler()` | `async` | Глобальный обработчик ошибок |
| `main()` | `async` | Точка входа: `initialize_database()` → `start_polling` |

> **Все синхронные операции** (SQLite, ChromaDB, извлечение текста) выносятся
> в отдельный поток через `asyncio.to_thread()`, чтобы не блокировать
> событийный цикл `aiogram`.


---

## 📝 Логирование

- Уровень — `LOG_LEVEL` из `.env` (по умолчанию `INFO`).
- Формат: `%(asctime)s | %(levelname)s | %(name)s | %(message)s`.
- Ошибки загрузки/индексации логируются через `logger.exception(...)`.
- Глобальный обработчик `@router.error()` ловит непредусмотренные исключения
  и уведомляет пользователя ответом «Произошла внутренняя ошибка».


---

## ⚠️ Ограничения и нюансы

- **Short memory — in-memory** — при перезапуске бота история диалога сбрасывается.
  ChromaDB и SQLite сохраняются на диске и восстанавливаются.
- **Ответы основаны на фрагментах документа** — если релевантных фрагментов нет,
  LLM честно сообщает об этом (адаптивный системный промпт).
- **Файлы хранятся на диске** в `data/documents/<user_id>/` — следите за
  свободным местом.
- **Эмбеддинги создаются по запросу** — загрузка большого документа может
  занять несколько секунд.
- **Контекст диалога не сохраняется между вопросами** — каждый вопрос
  обрабатывается независимо с новым поиском в ChromaDB. История диалога
  передаётся в LLM, но поиск релевантных фрагментов выполняется заново.
- **RAG-инлайн** — фрагменты документа добавляются в текущий `user`-сообщение
  одной строкой, а не как отдельные сообщения истории. Это экономит токены,
  но теоретически LLM может не учитывать фрагменты при «забывчивости».


---