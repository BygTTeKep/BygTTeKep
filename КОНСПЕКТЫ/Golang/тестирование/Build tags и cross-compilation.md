### Build tags (build constraints)

Условная компиляция: файл попадает в сборку только при выполнении условия. Директива пишется в самом начале файла, **перед `package`**, с пустой строкой после:

```go
//go:build linux && amd64

package mypkg
```

Синтаксис выражений: `&&`, `||`, `!`, скобки.

```go
//go:build (linux || darwin) && !cgo
//go:build integration
//go:build !windows
```
(Старая форма `// +build` устарела; `gofmt` сам синхронизирует обе.)

**Неявные ограничения по имени файла:** суффиксы `_GOOS`, `_GOARCH`, `_GOOS_GOARCH` перед расширением работают как теги без директивы:

```
file_linux.go          // только Linux
file_windows.go        // только Windows
file_linux_arm64.go    // только Linux на arm64
file_test.go           // тесты (это отдельное правило)
```

**Свои теги** включают флагом `-tags`:

```bash
go build -tags integration ./...
go test -tags "integration e2e" ./...
```

**Типичные применения:**

- **Интеграционные тесты**: `//go:build integration`, обычный `go test` их пропускает.
- **Платформенный код**: разные реализации для Linux/Windows/macOS.
- **Заглушки**: `feature_enabled.go` (`//go:build feature`) и `feature_disabled.go` (`//go:build !feature`) с одинаковым API.
- Отладочные сборки, разные редакции продукта (community/enterprise).
- Версии Go: `//go:build go1.22`.

**Ловушка:** файл с тегом, который не выполняется, **вообще не компилируется и не проверяется** (в том числе линтерами и `go vet` без нужного `-tags`). Ошибки обнаружатся поздно. Линтеру нужно передавать теги (`--build-tags`).

### Cross-compilation

Go компилирует под другую ОС и архитектуру **одной переменной окружения**, без отдельного тулчейна.

```bash
GOOS=linux   GOARCH=amd64 go build -o app-linux ./cmd/app
GOOS=windows GOARCH=amd64 go build -o app.exe ./cmd/app
GOOS=darwin  GOARCH=arm64 go build -o app-mac ./cmd/app
GOOS=linux   GOARCH=arm64 go build -o app-arm ./cmd/app
```

- **`GOOS`**: целевая ОС (`linux`, `windows`, `darwin`, `freebsd`, `js`, `wasip1` и др.).
- **`GOARCH`**: архитектура (`amd64`, `arm64`, `386`, `arm`, `riscv64`, `wasm`).
- Полный список поддерживаемых пар: `go tool dist list`.
- Текущие значения: `go env GOOS GOARCH`.
- Для ARM 32-бит есть `GOARM=7`.

#### CGO

По умолчанию при кросс-компиляции **`CGO_ENABLED=0`** (cgo отключён), потому что для C-кода нужен C-компилятор под целевую платформу. Если зависимости используют cgo (например, `mattn/go-sqlite3`), простой `GOOS=... go build` не сработает или тихо исключит файлы.

bash

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app ./cmd/app
```

Для сборки с cgo нужен кросс-компилятор (`CC=aarch64-linux-gnu-gcc`) или `zig cc`, либо замена на чистые Go-библиотеки (`modernc.org/sqlite`).

#### Практика: Docker и статические бинарники

С `CGO_ENABLED=0` получается **полностью статический бинарник** без зависимостей от libc, который можно положить в минимальный образ:

dockerfile

```dockerfile
FROM golang:1.24 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app ./cmd/app

FROM scratch          # или gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

- `-ldflags="-s -w"` убирает таблицу символов и отладочную информацию, бинарник меньше.
- Встроить версию на этапе сборки: `-ldflags="-X main.version=1.2.3"`.
- `-trimpath` убирает локальные пути из бинарника (воспроизводимые сборки).
- Для мультиархитектурных образов: `docker buildx` с `--platform linux/amd64,linux/arm64` (в Dockerfile доступны `TARGETOS` / `TARGETARCH`).
- Типичная ошибка: собрали на macOS (arm64) и запустили на Linux (amd64), получили `exec format error`. Нужен `GOOS=linux GOARCH=amd64`.