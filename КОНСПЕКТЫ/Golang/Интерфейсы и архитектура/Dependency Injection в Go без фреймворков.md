**DI** означает, что объект получает свои зависимости **снаружи**, а не создаёт сам. В Go это делается обычными конструкторами и интерфейсами, фреймворк не нужен.

### Плохо: зависимости создаются внутри

```go
type Service struct{}

func (s *Service) Do() error {
    db, _ := sql.Open("pgx", os.Getenv("DSN")) // жёсткая связь
    ...
}
```

Нельзя подменить в тесте, нельзя переиспользовать пул, скрытые зависимости от окружения.

### Хорошо: конструктор принимает зависимости

```go
type Service struct {
    users  userGetter
    mailer Mailer
    log    *slog.Logger
}

func NewService(users userGetter, mailer Mailer, log *slog.Logger) *Service {
    return &Service{users: users, mailer: mailer, log: log}
}
```

### Composition root: сборка в `main`

Всё связывается **в одном месте**, в `main` (точка входа). Остальной код про сборку ничего не знает.

```go
func main() {
    cfg := config.Load()
    log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    db, err := sql.Open("pgx", cfg.DSN)
    if err != nil {
        log.Error("open db", "err", err)
        os.Exit(1)
    }
    defer db.Close()

    // слои снизу вверх
    userRepo := postgres.NewUserRepo(db)
    mailer := smtp.NewMailer(cfg.SMTP)
    svc := billing.NewService(userRepo, mailer, log)
    handler := httpapi.NewHandler(svc, log)

    srv := &http.Server{Addr: cfg.Addr, Handler: handler.Routes()}
    // запуск, graceful shutdown...
}
```

Порядок создания очевиден из кода, граф зависимостей виден глазами, циклические зависимости не скомпилируются (нельзя использовать ещё не созданное).

### Способы внедрения

**1. Через конструктор** (основной).

**2. Через функциональные опции**, когда параметров много и часть необязательна:

```go
type Option func(*Server)

func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }
func WithLogger(l *slog.Logger) Option   { return func(s *Server) { s.log = l } }

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{addr: addr, timeout: 30 * time.Second, log: slog.Default()}
    for _, o := range opts {
        o(s)
    }
    return s
}
```

**3. Через поле или сеттер**: редко, делает объект частично инициализированным.

**4. Функции как зависимости** (для простых случаев):

```go
type Service struct{ now func() time.Time }  // время подменяется в тестах
```

**5. Структура конфигурации** вместо длинного списка параметров:

```go
type Deps struct {
    Users  userGetter
    Mailer Mailer
    Log    *slog.Logger
}
func NewService(d Deps) *Service { ... }
```

### Тестирование

Подставляем fake или мок через тот же конструктор:

```go
svc := billing.NewService(fakeUsers{}, fakeMailer{}, slog.Default())
```

### Что не стоит делать

- **Глобальные переменные и синглтоны** (`var DB *sql.DB`, `init()` с подключением): скрытые зависимости, порядок инициализации, сложные тесты, состояния между тестами.
- **`context.WithValue` для зависимостей** (БД, сервисы): теряется проверка типов, зависимость неявная. В контексте только данные запроса.
- **Service locator** (глобальный реестр, откуда достают что угодно): те же проблемы, что с глобалами.
- Передавать зависимость «насквозь» через слои, которым она не нужна.
- Конструкторы с 10+ параметрами: признак, что тип делает слишком много.

### DI-фреймворки (когда нужны)

В больших проектах ручная сборка в `main` разрастается. Тогда используют:

- **`google/wire`** генерирует код сборки на этапе компиляции (рантайм-магии нет, ошибки при генерации);
- **`uber-go/fx`** / **`dig`** рантайм-контейнер на рефлексии с управлением жизненным циклом (start/stop).

Сообщество Go в целом предпочитает ручную сборку: она явная, быстро читается и отлаживается. Фреймворк оправдан от десятков компонентов.