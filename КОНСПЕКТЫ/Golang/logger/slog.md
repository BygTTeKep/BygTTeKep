### Пакет `log/slog` (Go 1.21)

Стандартный **структурный** логгер с уровнями:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

slog.Info("user created", "id", 42, "email", "a@b.c")
slog.Error("db failed", "err", err)
```

Вывод JSON-handler:

```json
{"time":"2026-09-30T10:00:00Z","level":"INFO","msg":"user created","id":42,"email":"a@b.c"}
```

**Уровни:** `Debug (-4)`, `Info (0)`, `Warn (4)`, `Error (8)`. Минимальный уровень задаётся в `HandlerOptions.Level` (можно менять на лету через `slog.LevelVar`).

**Handlers** определяют формат и место вывода:

- `slog.NewTextHandler` формат `key=value`, удобно читать глазами.
- `slog.NewJSONHandler` JSON, удобно для сбора и поиска.
- Можно написать свой (интерфейс `slog.Handler`), например для отправки в внешнюю систему.
**Атрибуты и группы:**

```go
l := logger.With("service", "billing")            // постоянные поля
l.Info("paid", slog.Int("amount", 100))           // типизированные атрибуты
l.Info("req", slog.Group("http", "method", "GET", "status", 200))
```

**Контекст:** `logger.InfoContext(ctx, "msg")` передаёт `ctx` в handler, где можно достать, например, request ID или trace ID.

**Скрытие чувствительных данных:** тип может реализовать `slog.LogValuer`, чтобы контролировать, что попадёт в лог (пароли, токены).

**Производительность:** для горячих участков есть `LogAttrs` (без лишних аллокаций) и проверка `logger.Enabled(ctx, level)`.

### Сторонние библиотеки

- **zap** (Uber): очень быстрый, минимум аллокаций, строго типизированные поля (`zap.String`, `zap.Int`) и «сахарный» `SugaredLogger`.
- **zerolog**: тоже zero-allocation, fluent API (`log.Info().Str("k","v").Msg("...")`), JSON по умолчанию.
- **logrus**: старый, популярный, но в режиме поддержки и медленнее; в новых проектах обычно не берут.

**Что выбрать:** для нового проекта начинайте со **`slog`**: он в стандартной библиотеке, без зависимостей, и его можно использовать как единый фасад (у zap и zerolog есть адаптеры-handlers для slog). Zap или zerolog берут, когда нужна максимальная производительность или специфические возможности (сэмплирование, готовые интеграции).

### Практика

- Логируйте в **stdout** в JSON, а сбор и доставку оставьте инфраструктуре (Docker, Kubernetes, Loki, ELK).
- Добавляйте **request ID / trace ID**, чтобы связывать записи одного запроса.
- Не логируйте секреты и персональные данные.
- Логируйте ошибку **один раз**, на границе, где решается, что с ней делать.
- Уровни: `Debug` для отладки, `Info` для значимых событий, `Warn` для подозрительного, но не критичного, `Error` для сбоев.