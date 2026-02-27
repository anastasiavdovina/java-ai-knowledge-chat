# AI-чат по базе знаний

## 1. Описание проекта

Веб-приложение, позволяющее пользователям подключать облачные хранилища (Google Drive, Яндекс.Диск), автоматически индексировать документы и задавать вопросы по их содержимому через AI-чат. Система использует подход RAG (Retrieval-Augmented Generation) для поиска релевантных фрагментов документов и генерации ответов с указанием источников.

### Целевая аудитория
- Индивидуальные пользователи, которые хотят быстро находить информацию в своих документах
- Группы пользователей (команды), работающие с общей базой знаний

### Поддерживаемые типы документов
- **MVP:** TXT, PDF, DOCX
- **Расширение:** PPTX, XLSX/CSV

---

## 2. Архитектура

### Архитектурный подход: Модульный монолит

Один Spring Boot сервис, разделённый на чётко изолированные модули. Это оптимальный выбор для команды из 5 человек на 3 месяца разработки — минимальный операционный overhead при сохранении чистоты архитектуры и возможности масштабирования в будущем.

### Компоненты системы

#### Frontend (React SPA)
- **Chat UI** — интерфейс чата для вопросов и ответов с отображением источников
- **Admin Panel** — управление подключёнными дисками, просмотр индекса, управление группами
- **Auth UI** — вход через Google OAuth 2.0

#### Backend (Spring Boot 3, Java 21)
Пять модулей:

| Модуль | Ответственность |
|--------|----------------|
| **Auth Module** | Google OAuth 2.0, JWT-сессии, регистрация пользователей |
| **Storage Module** | Клиенты Google Drive API и Яндекс.Диск API, управление OAuth-токенами, скачивание файлов |
| **Indexing Module** | Парсинг документов (Apache PDFBox, Apache POI), разбиение на чанки, генерация embeddings, асинхронная индексация по расписанию и по запросу |
| **Chat Module** | RAG pipeline: векторный поиск → сборка контекста → запрос к LLM → атрибуция источников |
| **Admin Module** | CRUD пользователей и групп, управление подключёнными дисками, мониторинг статуса индексации |

#### База данных (PostgreSQL 16 + pgvector)
- **Users & Groups** — аккаунты, группы, подключённые диски, OAuth-токены
- **Vector Store** — чанки документов с векторными embeddings, метаданные для фильтрации по пользователю/группе

### Внешние сервисы
- **Google OAuth 2.0** — аутентификация пользователей
- **Google Drive API** — доступ к файлам пользователей
- **Яндекс.Диск API** — доступ к файлам пользователей
- **LLM API (OpenAI / Anthropic)** — генерация ответов и embeddings (через абстракцию LangChain4j)

### C4-диаграммы
Диаграммы архитектуры находятся в файлах `model.c4` и `views.c4` в формате LikeC4.
Предусмотрены следующие виды:
- **System Context (C1)** — общий вид системы и внешних зависимостей
- **Container (C2)** — внутренние контейнеры: Frontend, Backend, Database
- **Component (C3)** — компоненты Backend, Frontend и Database

---

## 3. Ключевые технологии

| Категория | Технология | Обоснование |
|-----------|-----------|-------------|
| Backend | Java 21, Spring Boot 3 | Требование курса, зрелая экосистема |
| AI/RAG Framework | LangChain4j | Абстракция над LLM-провайдерами, встроенный RAG pipeline, document splitters |
| Frontend | React (Vite) | Популярный, хорошо генерируется AI-агентами |
| Database | PostgreSQL 16 + pgvector | Единая БД для данных и векторов, достаточно для сотен документов |
| Парсинг PDF | Apache PDFBox | Стандарт для Java |
| Парсинг DOCX | Apache POI | Стандарт для Java |
| Аутентификация | Spring Security + OAuth 2.0 | Встроенная поддержка Google OAuth |
| Контейнеризация | Docker, Docker Compose | Простой деплой, воспроизводимость окружения |
| LLM | OpenAI API / Anthropic API | Облачные, через LangChain4j абстракцию |

---

## 4. RAG Pipeline (ключевой процесс)

### Индексация документов
1. Пользователь подключает Google Drive / Яндекс.Диск через OAuth 2.0
2. Storage Module скачивает файлы
3. Indexing Module парсит документы → извлекает текст
4. Текст разбивается на чанки (~500 токенов с перекрытием ~100 токенов)
5. Для каждого чанка генерируется embedding через LLM API
6. Embeddings сохраняются в pgvector с метаданными (user_id, group_id, document_id, source)

### Ответ на вопрос
1. Пользователь задаёт вопрос в чате
2. Вопрос преобразуется в embedding
3. Векторный поиск в pgvector (similarity search) с фильтрацией по user_id/group_id
4. Top-K релевантных чанков собираются в контекст
5. Контекст + вопрос отправляются в LLM
6. Ответ возвращается пользователю с указанием источников (документ, страница)

### Синхронизация
- **По запросу** — кнопка "Переиндексировать" в Admin Panel
- **По расписанию** — ежедневно (Spring @Scheduled)
- Индексация выполняется **асинхронно** (Spring @Async)

---

## 5. Тонкие места и риски

| Риск | Описание | Митигация |
|------|----------|-----------|
| **Качество RAG** | Результаты сильно зависят от стратегии чанкинга | Настраиваемый размер чанка и overlap, разные стратегии для разных типов файлов |
| **OAuth-токены** | Access token Google живёт 1 час, refresh token может протухнуть | Автоматическое обновление, graceful handling при протухании, уведомление пользователю |
| **Изоляция данных** | Утечка данных между пользователями при векторном поиске | Обязательная фильтрация по user_id/group_id на уровне SQL-запроса |
| **Стоимость API** | Каждый запрос = embedding API + LLM API | Кэширование ответов, использование дешёвых моделей (GPT-4o-mini) |
| **Индексация больших файлов** | PDF на 200 страниц — долгая операция | Асинхронная индексация, прогресс-бар в UI |
| **Галлюцинации LLM** | LLM может генерировать неверную информацию | Показ источников (документ + фрагмент), prompt engineering |

---



## 6. Структура модулей Backend

```
src/main/java/com/project/
├── auth/                    # Auth Module
│   ├── controller/          # OAuth endpoints
│   ├── service/             # Authentication logic
│   ├── config/              # Spring Security config
│   └── model/               # User, Session entities
├── storage/                 # Storage Module
│   ├── client/              # GoogleDriveClient, YandexDiskClient
│   ├── service/             # File sync orchestration
│   └── model/               # ConnectedDrive, OAuthToken
├── indexing/                # Indexing Module
│   ├── parser/              # DocumentParser interface + implementations
│   ├── chunker/             # TextChunker (split + overlap)
│   ├── embedding/           # EmbeddingService (LangChain4j)
│   ├── scheduler/           # IndexScheduler (@Scheduled)
│   └── model/               # DocumentChunk, IndexTask
├── chat/                    # Chat Module
│   ├── controller/          # Chat REST endpoints
│   ├── service/             # RAG pipeline orchestration
│   ├── retriever/           # VectorSearchService (pgvector queries)
│   └── model/               # ChatMessage, ChatResponse, Source
├── admin/                   # Admin Module
│   ├── controller/          # Admin REST endpoints
│   ├── service/             # Group/Drive management
│   └── model/               # Group, GroupMembership
└── common/                  # Shared utilities
    ├── config/              # App-wide config
    ├── exception/           # Global exception handling
    └── dto/                 # Shared DTOs
```

## 7. Паттерны проектирования

| Паттерн | Где применяется |
|---------|----------------|
| **Strategy** | `DocumentParser` — интерфейс с реализациями для PDF, DOCX, TXT |
| **Template Method** | Базовый класс для OAuth-клиентов (Google, Яндекс) |
| **Observer** | Уведомление UI о прогрессе индексации через WebSocket/SSE |
| **Repository** | Spring Data JPA для доступа к данным |
| **Builder** | Построение промптов для LLM из чанков и вопроса |
| **Facade** | `ChatService` как единая точка входа в RAG pipeline |

## 8. API Design (основные эндпоинты)

```
POST   /api/auth/google              # OAuth callback
GET    /api/auth/me                   # Current user info

GET    /api/drives                    # List connected drives
POST   /api/drives/google             # Connect Google Drive
POST   /api/drives/yandex             # Connect Yandex Disk
DELETE /api/drives/{id}               # Disconnect drive
POST   /api/drives/{id}/reindex       # Trigger reindexing

GET    /api/documents                 # List indexed documents
GET    /api/documents/{id}/status     # Indexing status

POST   /api/chat                      # Send question, get answer
GET    /api/chat/history              # Chat history

GET    /api/groups                    # List user's groups
POST   /api/groups                    # Create group
POST   /api/groups/{id}/members       # Add member
DELETE /api/groups/{id}/members/{uid} # Remove member
POST   /api/groups/{id}/drives        # Connect drive to group
```

## 9. Оценка трудозатрат

| Задача | Оценка (ч-ч) |
|--------|-------------|
| Настройка проекта (Spring Boot, Docker, CI) | 8-12 |
| Auth Module (Google OAuth + JWT) | 16-24 |
| Storage Module (Google Drive + Яндекс.Диск) | 20-30 |
| Indexing Module (парсинг + чанкинг + embeddings) | 24-32 |
| Chat Module (RAG pipeline) | 20-28 |
| Admin Module (группы, диски) | 12-16 |
| PostgreSQL schema + pgvector | 8-12 |
| Frontend: Chat UI | 16-24 |
| Frontend: Admin Panel | 16-24 |
| Frontend: Auth flow | 8-12 |
| Интеграционное тестирование | 16-24 |
| **Итого** | **~164-238** |
