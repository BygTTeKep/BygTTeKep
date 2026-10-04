**Агрегатор линтеров**: запускает десятки анализаторов за один проход, параллельно и с кэшем. Это стандарт в CI.

```bash
golangci-lint run ./...
golangci-lint run --fix        # автоисправление, где возможно
```

Конфиг в `.golangci.yml` (в v2 формат изменился, указывается `version: "2"`):

```yaml
version: "2"
linters:
  default: standard        # govet, errcheck, staticcheck, unused, ineffassign
  enable:
    - revive               # стиль
    - gosec                # безопасность
    - errorlint            # правильная работа с обёрнутыми ошибками (%w, errors.Is)
    - bodyclose            # незакрытый resp.Body
    - contextcheck         # потеря контекста в цепочке вызовов
    - gocritic             # разные советы по коду
    - prealloc             # слайсы, которым стоит задать cap
    - sqlclosecheck        # незакрытые rows/stmt
  settings:
    errcheck:
      check-blank: true
issues:
  max-issues-per-linter: 0
```

**Популярные линтеры:**

- `errcheck`: проигнорированные ошибки (`f.Close()` без проверки);
- `staticcheck`: глубокий анализ багов и устаревших API;
- `unused`, `ineffassign`: неиспользуемый код и бесполезные присваивания;
- `gosec`: уязвимости (слабая криптография, SQL-конкатенация);
- `govet`: тот самый `go vet`;
- `revive`: замена устаревшего `golint`;
- `gofumpt`, `goimports`: форматирование.

**Практика:**

- Включать постепенно: на старом проекте сначала `new-from-rev` (проверять только изменённый код), иначе утонете в замечаниях.
- Исключения помечать `//nolint:errcheck // причина`, обязательно с обоснованием.
- Запускать в CI и в редакторе (gopls).
- Не включать всё подряд (`enable-all`), шума будет больше пользы.