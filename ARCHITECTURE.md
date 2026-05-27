# Архитектура Audit Hypothesis Agent

## Обзор

RAG-агент для генерации аудиторских гипотез на основе внутренних документов банка.

```
┌──────────────────────────────────────────────────────────┐
│                      AuditAgent                          │
│                                                          │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────┐  │
│  │DocumentLoader│   │DocumentRetrie│    │   LLM      │  │
│  │             │   │ver (FAISS)   │    │(Qwen2.5-14B│  │
│  │  PDF/DOCX   │──▶│              │    │  4-bit)    │  │
│  │  TXT        │   │  BGE-M3      │    │            │  │
│  └─────────────┘   │  embeddings  │    └────────────┘  │
│                    └──────┬───────┘          ▲          │
│                           │                  │          │
│                    ┌──────▼───────┐    ┌────────────┐  │
│                    │  Multi-Query │    │   Prompt   │  │
│                    │  Expansion   │    │  Builder   │  │
│                    │  (LLM-based) │    │            │  │
│                    └──────┬───────┘    └────────────┘  │
│                           │                  ▲          │
│                    ┌──────▼───────┐          │          │
│                    │  CrossEncoder│──────────┘          │
│                    │  Reranker    │                      │
│                    │(bge-reranker)│                      │
│                    └─────────────┘                      │
└──────────────────────────────────────────────────────────┘
                           │
                    ┌──────▼───────┐
                    │HypothesesRepo│
                    │rt (Pydantic) │
                    └─────────────┘
```

## Компоненты

### DocumentLoader
- Поддерживает PDF (pdfplumber), DOCX (mammoth), TXT
- Единая точка входа для загрузки документов

### DocumentRetriever
- Хранит чанки в JSON (безопасно, читаемо)
- FAISS-индекс: `FlatIP` (<100k чанков) или `HNSW` (>50k чанков)
- Инкрементальное добавление документов
- Параметры: `chunk_size=800`, `chunk_overlap=100`

### Multi-Query Expansion
- Генерирует 3 перефразировки через тот же LLM (temperature=0.3)
- Увеличивает recall для коротких/неточных запросов

### Reranker (bge-reranker-v2-m3)
- CrossEncoder ранжирует результаты по паре [query, chunk]
- Работает после объединения и дедупликации всех результатов

### LLM (Qwen2.5-14B-Instruct, 4-bit)
- temperature=0.6 для структурированных гипотез
- Управление длиной контекста: MAX_CONTEXT_CHARS=12000

### HypothesesReport (Pydantic)
- Типизированный вывод: список `AuditHypothesis`
- Каждая гипотеза: title, rationale, consequences, verification_steps
- Метод `.to_markdown()` для форматированного вывода

## Исправленные баги (v1 → v2)

| # | Файл | Проблема | Исправление |
|---|------|---------|------------|
| 1 | `DocumentRetriever.search()` | `normalize_embeddigns` (опечатка) | `normalize_embeddings` |
| 2 | `generate_answer()` | `add_generation_promt` (опечатка) | `add_generation_prompt` |
| 3 | `retrieve_with_rerank()` | Дедупликация внутри цикла | Вынесена за пределы цикла |
| 4 | `retrieve_with_rerank()` | `dict_values` в `zip()` | Явный `list()` |
| 5 | `ask_agent()` | Вложенные кавычки в f-string | Одинарные кавычки внутри |
| 6 | Загрузка моделей | `rerunker` vs `reranker` | Единообразное `reranker` |
| 7 | Хранение чанков | `pickle` (security risk) | `JSON` |
| 8 | `expand_query()` | Заглушка, не расширяла запрос | LLM-based expansion |
| 9 | `DocumentRetriever` | Нет загрузки документов | `add_file()` + `DocumentLoader` |

## Потенциальные улучшения (следующие шаги)

1. **HyDE** (Hypothetical Document Embeddings): генерировать гипотетический ответ и эмбеддить его — улучшает recall
2. **Streaming output**: `model.generate()` → `TextStreamer` для интерактивного UX
3. **Кэширование эмбеддингов**: LRU-кэш для частых запросов
4. **Оценка качества**: feedback loop — оценивать релевантность гипотез
5. **API-сервер**: FastAPI wrapper для интеграции в BI/audit-платформы
6. **Логирование**: структурированные логи каждого запроса (вопрос → чанки → гипотезы)
