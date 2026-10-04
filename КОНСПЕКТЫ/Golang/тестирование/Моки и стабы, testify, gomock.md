### Терминология

- **Stub** (заглушка): возвращает заранее заданные ответы, ничего не проверяет.
- **Fake** (подделка): упрощённая рабочая реализация (in-memory хранилище вместо БД).
- **Mock**: объект, который ещё и **проверяет вызовы** (какие методы, с какими аргументами, сколько раз).
- **Spy**: записывает вызовы для последующей проверки.

На практике все их часто называют «моками».

### Основа в Go: интерфейсы

Подменять можно только то, что передано через интерфейс. Поэтому интерфейс объявляют **на стороне потребителя** и маленьким:
```go
type UserStore interface {
    GetUser(ctx context.Context, id int) (*User, error)
}

type Service struct{ store UserStore }

func (s *Service) Greeting(ctx context.Context, id int) (string, error) {
    u, err := s.store.GetUser(ctx, id)
    if err != nil {
        return "", err
    }
    return "Hello, " + u.Name, nil
}
```

Ручной стаб/фейк (часто достаточно)
```go
type stubStore struct {
    user *User
    err  error
}

func (s stubStore) GetUser(ctx context.Context, id int) (*User, error) {
    return s.user, s.err
}

func TestGreeting(t *testing.T) {
    svc := &Service{store: stubStore{user: &User{Name: "Ann"}}}
    got, err := svc.Greeting(context.Background(), 1)
    if err != nil || got != "Hello, Ann" {
        t.Fatalf("got %q, %v", got, err)
    }
}
```

Для одного-двух методов это проще и понятнее генераторов. Приём с функциями-полями: `type mockStore struct{ GetUserFn func(...) (...) }`, тогда в каждом тесте поведение задаётся своей лямбдой.

### `testify` (`github.com/stretchr/testify`)

Набор из нескольких пакетов.

**`assert` и `require`**: удобные проверки вместо `if got != want`.
```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestX(t *testing.T) {
    got, err := Divide(10, 2)
    require.NoError(t, err)      // при провале останавливает тест (как Fatal)
    assert.Equal(t, 5, got)      // при провале продолжает (как Error)
    assert.Len(t, items, 3)
    assert.ErrorIs(t, err, ErrNotFound)
    assert.Contains(t, "hello", "ell")
}
```

Порядок аргументов: `(t, expected, actual)`. Правило: `require` для условий, без которых дальше нет смысла (ошибка, nil), `assert` для остальных.

**`mock`**: моки с ожиданиями, написанные вручную или сгенерированные `mockery`.
```go
type MockStore struct{ mock.Mock }

func (m *MockStore) GetUser(ctx context.Context, id int) (*User, error) {
    args := m.Called(ctx, id)
    u, _ := args.Get(0).(*User)
    return u, args.Error(1)
}

func TestGreeting(t *testing.T) {
    m := new(MockStore)
    m.On("GetUser", mock.Anything, 1).Return(&User{Name: "Ann"}, nil)

    svc := &Service{store: m}
    got, _ := svc.Greeting(context.Background(), 1)

    assert.Equal(t, "Hello, Ann", got)
    m.AssertExpectations(t) // проверить, что ожидаемые вызовы были
}
```

### `gomock` (`go.uber.org/mock`)

Генерирует моки из интерфейсов, ожидания проверяются **при завершении теста**. Оригинальный `golang/mock` архивирован, поддерживаемая версия это форк от Uber (`go.uber.org/mock`).

```bash
go install go.uber.org/mock/mockgen@latest
mockgen -source=store.go -destination=mock_store_test.go -package=service
```
(или директива `//go:generate mockgen ...` и `go generate ./...`).
```go
func TestGreeting(t *testing.T) {
    ctrl := gomock.NewController(t)
    m := NewMockUserStore(ctrl)

    m.EXPECT().
        GetUser(gomock.Any(), 1).
        Return(&User{Name: "Ann"}, nil).
        Times(1)

    svc := &Service{store: m}
    got, err := svc.Greeting(context.Background(), 1)
    require.NoError(t, err)
    assert.Equal(t, "Hello, Ann", got)
}
```

### Сравнение подходов

| |Ручной stub/fake|testify/mock (+mockery)|gomock|
|---|---|---|---|
|Генерация|нет|опционально|да (`mockgen`)|
|Проверка вызовов|вручную|`AssertExpectations`|автоматически|
|Типобезопасность|да|слабая (`mock.Anything`, `Get(0).(T)`)|лучше (сгенерированные методы)|
|Когда|простые зависимости|много методов, нужны ожидания|много интерфейсов, строгие ожидания|

### Практика и ловушки

- **Мокайте границы системы** (БД, HTTP, очередь, время), а не внутреннюю логику. Тестировать «что вызвали у мока» вместо результата хрупко.
- Слишком строгие моки привязывают тест к реализации: рефакторинг ломает тесты, хотя поведение то же. Часто **fake** (in-memory репозиторий) надёжнее.
- Много моков в одном тесте означает слишком много зависимостей у тестируемого типа.
- Для БД часто лучше **реальная БД в контейнере** (`testcontainers`), а не мок `database/sql` (`sqlmock` существует, но проверяет строки SQL, а не поведение).
- Время и случайность выносят в интерфейс или параметр (`now func() time.Time`).
- Для HTTP-клиентов: `httptest.NewServer` вместо мока клиента.