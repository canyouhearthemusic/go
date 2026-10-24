# Go Modules: Deep Dive для Senior Go-разработчика

> Актуально для **Go 1.27** (go1.27.0, тег от 18 августа 2026). В локальном репозитории есть тег go1.27.1, а база уязвимостей Go уже ссылается на go1.27.2 как на версию с исправлениями (§14.1) — значит, go1.27.2 вышел.
> Факты сверены с исходниками `go1.27.0` (`src/cmd/go/internal/{modload,mvs,modfetch,gover,toolchain,load,vcs}`, `src/cmd/vendor/golang.org/x/mod/{modfile,sumdb}`, `go.env`, `doc/godebug.md`) и официальными release notes Go 1.11–1.27, спецификацией https://go.dev/ref/mod.
> Все выводы команд — **реальные прогоны** на `go1.27.0 darwin/arm64, Apple M3 Pro`. Для экспериментов без интернета использовался локальный прокси модулей (`GOPROXY=file://...`, см. §7.1) с выдуманными модулями `example.com/...`; для примеров с настоящими модулями — `proxy.golang.org`.
> Места, где факт не удалось проверить по первоисточнику или запуском, помечены **⚠️ не проверено**.

## Оглавление

- [Словарь терминов](#словарь-терминов)
- [TL;DR](#tldr)
- [1. Модель: модуль, пакет, версия](#1-модель-модуль-пакет-версия)
- [2. Файл go.mod: директивы](#2-файл-gomod-директивы)
- [3. Minimal Version Selection](#3-minimal-version-selection)
- [4. Обрезка графа и ленивая загрузка](#4-обрезка-графа-и-ленивая-загрузка)
- [5. Версии: semver, /v2, псевдоверсии, retract](#5-версии-semver-v2-псевдоверсии-retract)
- [6. go.sum и проверка целостности](#6-gosum-и-проверка-целостности)
- [7. Откуда берутся модули: прокси, sumdb, приватные репозитории](#7-откуда-берутся-модули-прокси-sumdb-приватные-репозитории)
- [8. Кэш модулей](#8-кэш-модулей)
- [9. Vendoring](#9-vendoring)
- [10. Workspaces: go.work](#10-workspaces-gowork)
- [11. Версия Go: директивы go и toolchain, GOTOOLCHAIN, godebug](#11-версия-go-директивы-go-и-toolchain-gotoolchain-godebug)
- [12. Инструменты: директива tool и go install](#12-инструменты-директива-tool-и-go-install)
- [13. Информация о сборке в бинарнике](#13-информация-о-сборке-в-бинарнике)
- [14. Безопасность зависимостей: govulncheck](#14-безопасность-зависимостей-govulncheck)
- [15. Команды go get, go mod tidy и компания](#15-команды-go-get-go-mod-tidy-и-компания)
- [16. Организация кода: internal, монорепозиторий, мультимодуль](#16-организация-кода-internal-монорепозиторий-мультимодуль)
- [17. Настройки и параметры](#17-настройки-и-параметры)
- [18. Production: Docker, CI, Kubernetes](#18-production-docker-ci-kubernetes)
- [19. Наблюдаемость и диагностика](#19-наблюдаемость-и-диагностика)
- [20. Грабли, заблуждения и советы: сводка](#20-грабли-заблуждения-и-советы-сводка)
- [21. Сравнение с другими платформами](#21-сравнение-с-другими-платформами)
- [22. Вопросы для собеседования](#22-вопросы-для-собеседования)
- [23. Чеклист и шпаргалка](#23-чеклист-и-шпаргалка)

---

## Словарь терминов

| Термин | Простыми словами |
|---|---|
| **Пакет (package)** | Папка с `.go`-файлами, у которых одинаковая строка `package x`. Единица импорта: `import "example.com/logx"`. |
| **Модуль (module)** | Набор пакетов, которые выпускаются (версионируются) вместе. Корень модуля — папка с файлом `go.mod`. |
| **Путь модуля (module path)** | Имя модуля, например `github.com/jackc/pgx/v5`. Это одновременно и «адрес», откуда его скачать, и префикс путей импорта всех его пакетов. |
| **Главный модуль (main module)** | Модуль, в папке которого вы запускаете `go build`. Только его `go.mod` управляет сборкой полностью (§2.2). |
| **Зависимость (dependency)** | Любой другой модуль, нужный для сборки. **Прямая** — её пакеты импортирует ваш код; **косвенная (indirect)** — её импортируют только ваши зависимости. |
| **Требование (requirement)** | Строка `require example.com/logx v1.2.0` — «мне нужна версия **не ниже** v1.2.0». |
| **Граф модулей (module graph)** | Все модули и стрелки «кто кого требует и какой версии». |
| **Build list** | Итоговый список «модуль → выбранная версия», по которому идёт сборка. Его показывает `go list -m all`. |
| **MVS (Minimal Version Selection)** | Алгоритм Go, который из графа требований выбирает версии: для каждого модуля — максимальную из **требуемых** (не из существующих!). §3. |
| **SemVer (семантическое версионирование)** | Формат `vMAJOR.MINOR.PATCH`: несовместимые изменения — MAJOR, новые возможности — MINOR, исправления — PATCH. |
| **Major version suffix** | Суффикс `/v2`, `/v3`… в пути модуля для версий v2+. §5.2. |
| **Псевдоверсия (pseudo-version)** | Версия для коммита без тега, например `v1.6.1-0.20241114170450-2d3c2a9cc518`. §5.3. |
| **Retract** | Отзыв опубликованной версии автором: «не используйте v1.4.0». §5.5. |
| **go.sum** | Файл с криптографическими хешами каждой скачанной версии модуля. Защищает от подмены кода. §6. |
| **h1-хеш** | Формат хеша в `go.sum`: SHA-256 от списка SHA-256 всех файлов модуля, в base64. §6.1. |
| **Прокси модулей (module proxy)** | HTTP-сервер, который раздаёт модули в простом формате (`/@v/list`, `.info`, `.mod`, `.zip`). По умолчанию — `proxy.golang.org`. §7. |
| **Checksum database (sumdb)** | Публичный журнал хешей всех публичных модулей — `sum.golang.org`. Гарантирует, что все в мире получают одинаковый код для одной версии. §6.2. |
| **Transparency log** | Журнал, в который можно только дописывать, а подделку можно доказать математически (дерево Меркла). Так устроен `sum.golang.org`. |
| **Кэш модулей (module cache)** | Папка `$GOMODCACHE` (по умолчанию `~/go/pkg/mod`), куда скачиваются модули. Файлы в ней только для чтения. §8. |
| **Vendoring** | Копирование кода зависимостей в папку `vendor/` внутри проекта. §9. |
| **Workspace (рабочее пространство)** | Файл `go.work`, который объединяет несколько локальных модулей в одну сборку. §10. |
| **Тулчейн (toolchain)** | Конкретная версия Go: компилятор, `go`-команда, стандартная библиотека. `go1.27.0` — тулчейн. §11. |
| **GODEBUG** | Переключатели поведения рантайма и стандартной библиотеки (например, `http2client=0`). Их значения по умолчанию зависят от строки `go` в `go.mod`. §11.5. |
| **Tidy** | «Прибраться»: `go mod tidy` добавляет недостающие и удаляет лишние требования. §15.2. |

---

## TL;DR

1. **Модуль** — единица версионирования, **пакет** — единица импорта. Версию имеет модуль целиком, а не отдельный пакет.
2. **MVS выбирает для каждого модуля максимальную из требуемых версий**, а не последнюю доступную. Новая версия зависимости сама «не приезжает»: сборка воспроизводима без lock-файла (§3).
3. **`replace` и `exclude` работают только в главном модуле.** В зависимостях они молча игнорируются — так написано прямо в парсере `go.mod` (§2.2). `go install pkg@version` и вовсе отказывается собирать модуль с `replace` (§2.3).
4. **С go 1.17 граф обрезается (pruning)**: Go не читает `go.mod` зависимостей зависимостей, если они не нужны. Поэтому в `go.mod` теперь перечислены все косвенные зависимости (§4).
5. **Версия v2+ — это другой модуль с другим путём (`/v2`).** Забыл суффикс — `invalid version: should be v0 or v1, not v2` (§5.2).
6. **`go.sum` — не lock-файл, а список хешей.** Подмена кода на прокси даёт `SECURITY ERROR` (воспроизведено в §6.3). Конфликты слияния в `go.sum` разрешаются объединением строк и `go mod tidy` (§6.5).
7. **Без `GOPRIVATE` путь вашего приватного модуля уходит на `proxy.golang.org` и `sum.golang.org`** — видно в `go mod download -x` (§7.3). Переменной `GONOSUMCHECK` не существует: правильно `GONOSUMDB`.
8. **Кэш модулей только для чтения**: `rm -rf` не работает, нужен `go clean -modcache`. Правку файлов в кэше сборка **не замечает**, а `go mod verify` — замечает (§8).
9. **`vendor/` включается сам**, если он есть и `go` в `go.mod` ≥ 1.14. Ручная правка `go.mod` без `go mod vendor` → `inconsistent vendoring` (§9).
10. **С Go 1.21 строка `go` — жёсткий минимум**, а `GOTOOLCHAIN=auto` (по умолчанию) сам скачивает нужную версию Go. `go 1.21` и `go1.21` — разные вещи: первое — версия языка, второе — несуществующий тулчейн (§11).
11. **Директива `tool` (Go 1.24) заменяет `tools.go`**, но зависимости инструментов попадают в общий граф и **могут поднять версии зависимостей приложения** — воспроизведено в §12.2. Лечится отдельным `-modfile`.
12. **Go сам вшивает версию в бинарник** (Go 1.18 — коммит, Go 1.24 — версия из git-тега). `+dirty` появляется даже из-за неотслеживаемого файла, например собранного бинарника в корне (§13.3).
13. **`go mod tidy` учитывает файлы со всеми build-тегами**, кроме `ignore`. Миф «tidy удаляет зависимость из `_linux.go`» неверен (§15.3). С go 1.27 tidy ещё и склеивает лишние блоки `require` (§15.4).
14. **Расхождение документации и кода:** release notes Go 1.26 обещают, что `go mod init` пишет `go 1.(N-1).0`, но в go1.26.1 и go1.27 это откатили — пишется текущая версия (§11.1).
15. **`govulncheck` смотрит на граф вызовов**: он отличает «уязвимый модуль в зависимостях» от «ваш код реально вызывает уязвимую функцию» (§14).

---

## 1. Модель: модуль, пакет, версия

### 1.1. Из чего состоит модуль

```text
github.com/acme/orders            ← корень репозитория = корень модуля
├── go.mod                        ← имя модуля, версия Go, требования
├── go.sum                        ← хеши зависимостей
├── cmd/orders/main.go            ← package main (пакет github.com/acme/orders/cmd/orders)
├── internal/storage/pg.go        ← пакет, видимый только внутри модуля (§16.1)
└── kafka/consumer.go             ← пакет github.com/acme/orders/kafka
```

Модуль — это **дерево папок с `go.mod` в корне**. Все пакеты внутри дерева (кроме вложенных папок со своим `go.mod`) принадлежат этому модулю и получают одну версию. Версия модуля — это **git-тег** вида `v1.4.2` (или `subdir/v1.4.2` для модуля в подпапке репозитория).

**Инвариант:** путь импорта пакета = путь модуля + путь папки внутри модуля. Поэтому, увидев `import "github.com/jackc/pgx/v5/pgxpool"`, Go знает: модуль — `github.com/jackc/pgx/v5`, пакет — папка `pgxpool`.

### 1.2. История: от GOPATH к модулям

| Версия Go | Что произошло | Источник |
|---|---|---|
| до 1.11 | Только **GOPATH**: весь код лежит в `$GOPATH/src`, версий нет — `go get` берёт последний коммит. Сторонние менеджеры: `godep`, `glide`, `dep`. | — |
| 2018 | Расс Кокс публикует серию статей **vgo** и алгоритм MVS (research.swtch.com/vgo-mvs — ссылка стоит прямо в комментарии `src/cmd/go/internal/mvs/mvs.go`). | исходник |
| **1.11** | «preliminary support for a new concept called "modules"… Module support is considered experimental». | release notes 1.11 |
| **1.13** | `GOPROXY` по умолчанию `https://proxy.golang.org,direct`, появились `GOSUMDB`, `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB`. | release notes 1.13 |
| **1.14** | «Module support in the go command is now ready for production use». `-mod=vendor` по умолчанию при наличии `vendor/`. | release notes 1.14 |
| **1.16** | `GO111MODULE=on` по умолчанию; сборка больше не меняет `go.mod` (`-mod=readonly`); `go install pkg@version`; директива `retract`; `GOVCS`. | release notes 1.16 |
| **1.17** | Обрезка графа (pruning) и ленивая загрузка; косвенные зависимости в отдельном блоке `require`; `// Deprecated:` в `go.mod`. | release notes 1.17 |
| **1.18** | Workspaces (`go.work`); `go get` больше не собирает и не устанавливает; VCS-информация в бинарнике. | release notes 1.18 |
| **1.21** | Строка `go` — жёсткий минимум; директива `toolchain`; автоскачивание тулчейнов; первая версия релиза — `1.N.0`. | release notes 1.21 |
| **1.22** | `go work vendor`; `go get` вне модуля в режиме GOPATH больше не работает. | release notes 1.22 |
| **1.23** | Директива `godebug` в `go.mod`/`go.work`; `go mod tidy -diff`; `go env -changed`. | release notes 1.23 |
| **1.24** | Директива `tool`, `go get -tool`; `GOAUTH`; версия главного модуля из git-тега; `GODEBUG=toolchaintrace=1`. | release notes 1.24 |
| **1.25** | Директива `ignore`; шаблон пакетов `work`; `go` больше не дописывает строку `toolchain` сам; `go version -m -json`. | release notes 1.25 |
| **1.26** | `go mod init` пишет `go 1.(N-1).0` — **откатили в 1.26.1** (§11.1). | release notes 1.26, git |
| **1.27** | `go mod tidy` склеивает дублирующиеся блоки `require`; удалённые GODEBUG-настройки принимаются, если стоят в финальном значении; поддержка `bzr` удалена; `go doc pkg@version`. | release notes 1.27, `gover/version.go` |

**Почему отказались от GOPATH.** В GOPATH у каждого пакета ровно одна копия на машину, без версии. Два проекта, которым нужны разные версии одной библиотеки, на одной машине не уживались, а сборка зависела от того, что когда-то скачал `go get`. Сторонние менеджеры (`dep`) решали это lock-файлом и SAT-солвером, но у каждого был свой формат. Модули сделали версии частью самой `go`-команды.

### Простыми словами

Модуль — это «коробка» с пакетами, на которой написан номер версии. Вы говорите «мне нужна коробка logx не старее v1.2.0», а Go решает, какую коробку взять, скачивает её и проверяет, что её никто не подменил.

### Что важно для Senior

- Модуль, а не пакет, — единица версии. Разбивать репозиторий на несколько модулей стоит только тогда, когда части действительно выпускаются независимо (§16.3).
- Режим GOPATH (`GO111MODULE=off`) ещё жив для `go build`, но `go get` в нём с Go 1.22 не работает. На собеседовании полезно знать вехи: 1.11 (появление), 1.13 (прокси), 1.16 (по умолчанию), 1.17 (pruning), 1.21 (тулчейны).
- Все правила этой темы — свойства **`go`-команды**, а не компилятора: компилятор про модули ничего не знает, ему передают готовые пути к пакетам.

---

## 2. Файл go.mod: директивы

### 2.1. Полный пример

```text
module github.com/acme/orders

// Deprecated: используйте github.com/acme/orders/v2   ← пометка устаревшего модуля (1.17)

go 1.27.0                      // минимальная версия Go (1.21+: жёсткое требование)

toolchain go1.27.2             // предпочтительный тулчейн (1.21)

godebug (                      // значения GODEBUG по умолчанию для бинарников (1.23)
	http2client=0
)

require (
	github.com/jackc/pgx/v5 v5.11.0
	github.com/segmentio/kafka-go v0.4.19
)

require github.com/jackc/puddle/v2 v2.2.2 // indirect

replace github.com/segmentio/kafka-go => ../kafka-go-fork

exclude github.com/jackc/pgx/v5 v5.10.0

retract v1.3.0 // опубликована по ошибке

tool github.com/sqlc-dev/sqlc/cmd/sqlc   // инструмент разработки (1.24)

ignore ./web/node_modules                // папки, которые go не сканирует (1.25)
```

Версии здесь иллюстративные.

### 2.2. Какие директивы на что влияют

Парсер `go.mod` (`src/cmd/vendor/golang.org/x/mod/modfile/rule.go`) знает ровно эти глаголы: `module`, `go`, `toolchain`, `godebug`, `require`, `exclude`, `replace`, `retract`, `tool`, `ignore`. Ключевой фрагмент — функция `add`:

```go
// If strict is false, this module is a dependency.
// We ignore all unknown directives as well as main-module-only
// directives like replace and exclude. ...
if !strict {
	switch verb {
	case "go", "module", "retract", "require", "ignore":
		// want these even for dependency go.mods
	default:
		return
	}
}
```

| Директива | С версии | Читается из `go.mod` зависимостей? | Что делает |
|---|---|---|---|
| `module` | 1.11 | да | Путь модуля. Комментарий `// Deprecated:` над ней помечает модуль устаревшим (1.17). |
| `go` | 1.12 (пишется) | да | Версия языка и минимальная версия Go (с 1.21 — жёсткая). Включает/выключает pruning (§4), vendor по умолчанию (§9), значения GODEBUG (§11.5). |
| `require` | 1.11 | да | Минимальная требуемая версия зависимости. |
| `retract` | 1.16 | да (из последней версии модуля) | Отзыв своих версий (§5.5). |
| `ignore` | 1.25 | да | Папки, которые не матчатся шаблонами `./...` и `all`, но остаются в zip модуля. |
| `exclude` | 1.11 | **нет** | Запрет конкретной версии (§3.5). |
| `replace` | 1.11 | **нет** | Подмена модуля другой версией или локальной папкой. |
| `toolchain` | 1.21 | **нет** | Предпочтительный тулчейн. |
| `godebug` | 1.23 | **нет** | Значения GODEBUG по умолчанию. |
| `tool` | 1.24 | **нет** | Инструменты разработки (§12). |

**Почему так.** Если бы `replace` из зависимостей применялись, то любая библиотека могла бы незаметно подменить код в вашей сборке, а два `replace` на один модуль из разных библиотек конфликтовали бы без способа разрешения. Решение: «конфигурацию сборки определяет только тот, кто собирает». Цена — автор библиотеки не может «починить» свою зависимость через `replace` для пользователей.

**Грабли: replace в библиотеке не действует у потребителя**

Модуль `example.com/pg v1.1.0` в своём `go.mod` подменяет `logx`:

```text
module example.com/pg

go 1.22

require example.com/logx v1.2.0

// Автор pg «чинит» баг в logx у себя локально:
replace example.com/logx v1.2.0 => example.com/logx v1.0.0
```

Сервис `orders` зависит от `pg v1.1.0`:

```text
$ go get example.com/pg@v1.1.0
go: upgraded example.com/pg v1.0.0 => v1.1.0
$ go list -m all
example.com/orders
example.com/kafka v1.0.0
example.com/logx v1.2.0
example.com/pg v1.1.0
$ go run .
kafka uses logx v1.2.0
pg uses logx v1.2.0
```

Почему так: `replace` из зависимости выброшен ещё при разборе `go.mod`. Как заметить: никак — ни ошибки, ни предупреждения; только `go list -m all` покажет «не ту» версию. Как исправить: автор библиотеки должен выпустить исправленную версию зависимости или поднять требование (`require ... v1.2.1`), а не полагаться на `replace`.

### 2.3. replace с локальным путём

`replace` бывает двух видов: на другую версию/модуль (`=> example.com/fork v1.2.3`) и на **папку** (`=> ../logx-fork`). Второй вид не скачивается и не проверяется по `go.sum`.

**Грабли: закоммиченный replace на локальную папку**

```text
$ go mod edit -replace example.com/logx=../../libs/logx-fork
$ tail -2 go.mod

replace example.com/logx => ../../libs/logx-fork
$ go build -o /dev/null .
../../gopath/pkg/mod/example.com/kafka@v1.0.0/kafka.go:3:8: example.com/logx@v1.2.0: replacement directory ../../libs/logx-fork does not exist
```

У автора всё собирается, а в CI и у коллег — нет. Как заметить: ревью `go.mod`, проверка в CI `grep -n '=> \.' go.mod`. Как исправить: для локальной разработки нескольких модулей используйте `go.work` (§10), его не коммитят.

**Грабли: go install pkg@version и replace**

```text
$ go install example.com/migrate@v1.0.0
go: downloading example.com/migrate v1.0.0
go: example.com/migrate@v1.0.0 (in example.com/migrate@v1.0.0):
	The go.mod file for the module providing named packages contains one or
	more replace directives. It must not contain directives that would cause
	it to be interpreted differently than if it were the main module.
```

`go install cmd@version` собирает модуль «как зависимость», где `replace` не действует, но и молча проигнорировать его нельзя: инструмент собрался бы не так, как у автора. Поэтому ошибка. Как исправить: не держать `replace` в `go.mod` публикуемых CLI-инструментов.

### Простыми словами

`go.mod` — это анкета модуля: как зовут, на какой версии Go написан, что ему нужно. Часть анкеты (что нужно, какие версии отозваны) читают все, кто его использует. Другая часть (подмены, запреты, настройки) — это «домашние правила», которые действуют, только когда собирают именно этот модуль.

### Что важно для Senior

- Только `module`, `go`, `require`, `retract`, `ignore` влияют на потребителей. `replace`, `exclude`, `toolchain`, `godebug`, `tool` — лишь на главный модуль.
- `replace` на папку в закоммиченном `go.mod` — почти всегда ошибка; для локальной разработки есть `go.work`.
- CLI, которые ставят через `go install ...@latest`, не должны содержать `replace` — иначе установка ломается.
- Директивы старше вашего тулчейна (например, `tool` для Go 1.23) из зависимостей просто игнорируются — это и есть «forward compatibility» парсера.

---
## 3. Minimal Version Selection

### 3.1. Алгоритм

Спецификация (go.dev/ref/mod, раздел MVS):

> MVS starts at the main modules (special vertices in the graph that have no version) and traverses the graph, tracking the highest required version of each module. At the end of the traversal, the highest required versions comprise the build list: they are the minimum versions that satisfy all requirements.

То есть:

1. Начать с главного модуля.
2. Обойти граф по стрелкам `require` (каждая стрелка — «нужна версия не ниже X»).
3. Для каждого пути модуля запомнить **максимальную встреченную требуемую** версию.
4. Эти версии и есть build list.

В коде (`src/cmd/go/internal/mvs/graph.go`, метод `Graph.Require`) это три строки:

```go
for _, dep := range reqs {
	...
	if g.cmp(dep.Path, g.Selected(dep.Path), dep.Version) < 0 {
		g.selected[dep.Path] = dep.Version
	}
}
```

А обход в `mvs.go` (`buildList`) идёт параллельно — `work.Do(10, ...)`, до 10 одновременных чтений `go.mod`, потому что каждое чтение может быть сетевым запросом к прокси.

### 3.2. Пример: сервис orders

Опубликованы версии `example.com/logx`: v1.0.0, v1.1.0, v1.2.0, **v1.3.0**. Клиент Kafka требует `logx v1.1.0`, клиент PostgreSQL — `logx v1.2.0`.

```mermaid
graph LR
  orders["orders (главный)"] -->|"≥ v1.0.0"| kafka["kafka v1.0.0"]
  orders -->|"≥ v1.0.0"| pg["pg v1.0.0"]
  kafka -->|"≥ v1.1.0"| logx11["logx v1.1.0"]
  pg -->|"≥ v1.2.0"| logx12["logx v1.2.0 ✔ выбрана"]
  logx13["logx v1.3.0 (существует, но никто не требует)"]
```

Реальный прогон:

```text
$ go get example.com/kafka@v1.0.0 example.com/pg@v1.0.0
go: added example.com/kafka v1.0.0
go: added example.com/logx v1.2.0
go: added example.com/pg v1.0.0
$ go run .
kafka uses logx v1.2.0
pg uses logx v1.2.0
$ go mod graph
example.com/orders example.com/kafka@v1.0.0
example.com/orders example.com/logx@v1.2.0
example.com/orders example.com/pg@v1.0.0
example.com/orders go@1.27.0
example.com/kafka@v1.0.0 example.com/logx@v1.1.0
example.com/kafka@v1.0.0 go@1.22
example.com/logx@v1.2.0 go@1.22
example.com/pg@v1.0.0 example.com/logx@v1.2.0
example.com/pg@v1.0.0 go@1.22
go@1.27.0 toolchain@go1.27.0
$ go list -m -u all
example.com/orders
example.com/kafka v1.0.0
example.com/logx v1.2.0 [v1.3.0]
example.com/pg v1.0.0
```

Что видно:
- выбрана **v1.2.0** — максимум из требований (v1.1.0 и v1.2.0), а не последняя v1.3.0;
- kafka получила v1.2.0, хотя просила v1.1.0: «не ниже» допускает более новую;
- `go list -m -u` показывает доступное обновление в квадратных скобках — само оно не применяется;
- в графе есть и узлы `go@...`: с Go 1.21 версия Go тоже участвует в MVS как «модуль» `go` и `toolchain`.

### 3.3. Почему не SAT-солвер

В npm, Cargo, Bundler, старом `dep` зависимости задают **диапазоны** (`^1.2`, `>=1.1,<2`), и менеджер ищет комбинацию, удовлетворяющую всем ограничениям. Это задача выполнимости (NP-полная в общем случае), результат зависит от того, какие версии опубликованы **сегодня**, поэтому нужен lock-файл.

| | MVS (Go) | SAT/диапазоны (npm, Cargo) |
|---|---|---|
| Что пишут в манифест | только нижнюю границу | диапазон |
| Сложность | линейна по размеру графа | NP-полная в общем случае |
| Зависит от новых публикаций | **нет**: выбор определяется только `go.mod`-файлами | да, отсюда lock-файл |
| Lock-файл | не нужен (`go.mod` сам по себе воспроизводим) | обязателен |
| Верхние границы («не выше v1.5») | нет; есть только `exclude` в главном модуле | есть |
| Цена | обновления нужно делать явно (`go get -u`); баг в новой версии нельзя «запретить» для потребителей | сложные конфликты, «работало вчера» |

**Цена решения.** Go не умеет выразить «v1.5 сломана, не берите её» для всех потребителей. Автор может только отозвать версию (`retract`, §5.5) или выпустить исправление; потребитель — `exclude` у себя (§3.5).

### 3.4. Upgrade и downgrade: downgrade удаляет зависимых

`go get logx@v1.3.0` поднимает требование — элементарно. А вот понижение устроено интереснее: MVS должен гарантировать, что **никто** в графе не требует версию выше целевой. Если такой модуль есть, его тоже понижают — вплоть до удаления.

**Грабли: go get старой версии молча удаляет зависимость**

```text
$ go get example.com/logx@v1.1.0
go: downloading example.com/logx v1.1.0
go: downgraded example.com/logx v1.2.0 => v1.1.0
go: removed example.com/pg v1.0.0
$ go build ./...
main.go:7:2: no required module provides package example.com/pg; to add it:
	go get example.com/pg
```

Почему так: у `pg` есть только версия v1.0.0, и она требует `logx ≥ v1.2.0`. Совместимой с `logx v1.1.0` версии `pg` нет, поэтому `pg` понижен до `none` (функция `Downgrade` в `mvs.go`). Как заметить: строка `go: removed` в выводе `go get` и последующая ошибка сборки. Как исправить: не понижать общую зависимость — либо поднять, либо `exclude`/`replace` в главном модуле, либо исправить проблему в `pg`.

### 3.5. exclude может опустить версию ниже требуемой

**Грабли: exclude тихо нарушает минимальное требование зависимости**

`pg` требует `logx ≥ v1.2.0`. Добавляем в главный модуль `exclude example.com/logx v1.2.0`:

```text
$ go mod tidy
go: dropping requirement on excluded version example.com/logx v1.2.0
$ cat go.mod
module example.com/orders

go 1.27.0

require (
	example.com/kafka v1.0.0
	example.com/pg v1.0.0
)

require example.com/logx v1.1.0 // indirect

exclude example.com/logx v1.2.0
$ go run .
kafka uses logx v1.1.0
pg uses logx v1.1.0
```

`pg` получила v1.1.0, хотя требовала v1.2.0. Это поведение с Go 1.16: «The go command now ignores requirements on module versions excluded by exclude directives in the main module. Previously, the go command used the next version higher than an excluded version» (release notes 1.16). Как исправить: после `exclude` явно поднять версию выше исключённой (`go get example.com/logx@v1.3.0`).

### Простыми словами

Каждый модуль говорит: «мне нужна logx не старее такой-то». Go выбирает самую новую **из названных** версий — минимальную, которая устроит всех. Если завтра выйдет logx v9, ваша сборка не изменится, пока вы сами не попросите обновиться.

### Что важно для Senior

- MVS — это «максимум из минимумов». Он детерминирован и не зависит от новых публикаций, поэтому lock-файл не нужен.
- Понижение версии через `go get` может удалить зависимые модули — читайте вывод `go get`.
- `exclude` не «перепрыгивает» на следующую версию, а выбрасывает требование; результат может оказаться ниже минимума зависимости.
- Верхних границ версий в Go нет. Это осознанная цена за простоту и воспроизводимость.

---

## 4. Обрезка графа и ленивая загрузка

### 4.1. Что изменилось в Go 1.17

До 1.17 Go читал `go.mod` **всех** версий всех модулей в графе, даже тех, чьи пакеты вам не нужны. В больших проектах это сотни файлов и сетевых запросов, а лишние требования далёких модулей поднимали версии ваших зависимостей.

С 1.17 (go.dev/ref/mod):

> If the main module is at go 1.17 or higher, the module graph used for minimal version selection includes only the immediate requirements for each module dependency that specifies go 1.17 or higher in its own go.mod file…

Чтобы обрезка не сломала сборку, `go.mod` модуля на go ≥ 1.17 обязан перечислять **все** модули, которые дают пакеты, транзитивно импортируемые этим модулем. Отсюда второй блок `require ... // indirect`. Пороговые версии зашиты в `src/cmd/go/internal/gover/version.go`:

```go
// ExplicitIndirectVersion is the Go version at which a
// module's go.mod file is expected to list explicit requirements on every
// module that provides any package transitively imported by that module.
ExplicitIndirectVersion = "1.17"
...
// Note that this default must stay at Go 1.16; it cannot be moved forward.
DefaultGoModVersion = "1.16"
```

`DefaultGoModVersion` — версия, которую Go предполагает для `go.mod` **без** строки `go`: такой файл считается старым, и для него pruning не работает.

### 4.2. Эксперимент: одна и та же цепочка на go 1.16 и go 1.22

Цепочка: сервис `svc` → `a` → `d` → `e`. Пакет `a` (который импортирует сервис) **не** импортирует `d`; `d` нужен только внутреннему пакету `a/internal/admin`. Опубликованы два набора модулей: с `go 1.16` в `go.mod` (`a116`, `d116`, `e116`) и с `go 1.22` (`a122`, `d122`, `e122`). Главный модуль — `go 1.27.0`.

```text
### зависимости с go 1.16: go list -m all
example.com/svc
example.com/a116 v1.0.0
example.com/d116 v1.0.0
example.com/e116 v1.0.0
### go mod graph
example.com/svc example.com/a116@v1.0.0
example.com/a116@v1.0.0 example.com/d116@v1.0.0
example.com/d116@v1.0.0 example.com/e116@v1.0.0
### go.sum
example.com/a116 v1.0.0
example.com/a116 v1.0.0/go.mod
example.com/d116 v1.0.0/go.mod
example.com/e116 v1.0.0/go.mod

### зависимости с go 1.22: go list -m all
example.com/svc
example.com/a122 v1.0.0
example.com/d122 v1.0.0
### go mod graph
example.com/svc example.com/a122@v1.0.0
example.com/a122@v1.0.0 example.com/d122@v1.0.0
### go.sum
example.com/a122 v1.0.0
example.com/a122 v1.0.0/go.mod
```

(В `go.sum` показаны только первые два поля строк.)

Что видно:
- со старыми зависимостями Go прочитал `go.mod` у `d116` и `e116` (их хеши в `go.sum`), и `e116` попал в граф, хотя его код не нужен;
- с новыми — `d122` в графе есть (это прямое требование `a122`), но **его `go.mod` даже не скачан** (нет строки `d122 v1.0.0/go.mod`), а `e122` из графа исчез. Это и есть «lazy loading»: «the go tool no longer reads (or even downloads) go.mod files for dependencies if they are not needed» (release notes 1.17).

### 4.3. Следствия

> **Заблуждение:** «Строки `// indirect` в `go.mod` — мусор, их можно удалить руками». На самом деле в модулях на go ≥ 1.17 они обязательны: без них обрезанный граф не содержит нужных модулей, и сборка или `go mod tidy` их вернёт. Это было «почти мусором» до Go 1.17, когда косвенные зависимости вычислялись из графа целиком.

> **Совет.** Поднимите `go` в `go.mod` хотя бы до 1.17 у всех своих библиотек — тогда потребители перестанут скачивать `go.mod` ваших транзитивных зависимостей. В эксперименте выше это 2 строки `go.sum` вместо 4. Не нужно, если библиотека обязана собираться Go старше 1.17.

**Грабли: go 1.16 в go.mod приложения поднимается сам**

При первом прогоне этого эксперимента в `go.mod` сервиса стояло `go 1.16`, а зависимости были на `go 1.22`. После `go mod tidy` строка стала `go 1.22`. Причина — константа `GoStrictVersion = "1.21"` в `gover/version.go`: «every module must have a go version line ≥ all its dependencies». Как заметить: diff `go.mod` после `tidy`/`get`. Это не баг: модуль, зависящий от кода на Go 1.22, не может собираться Go 1.16.

### Простыми словами

Раньше Go перечитывал «анкеты» всех модулей до самого дна, даже тех, что не нужны. Теперь каждый модуль сам перечисляет всё, что ему нужно, и Go читает анкеты только на один уровень вглубь. Быстрее, меньше скачиваний, меньше случайных повышений версий.

### Что важно для Senior

- Pruning включается строкой `go ≥ 1.17` — и у главного модуля, и у зависимости.
- Поэтому в `go.mod` два блока `require`: прямые и `// indirect`. С go 1.27 `tidy` сам поддерживает ровно два блока (§15.4).
- Модули из обрезанной части графа всё ещё видны в `go list -m all`, но собирать их пакеты напрямую нельзя — сначала `go get`.
- `go.mod` без строки `go` трактуется как 1.16: pruning для него не работает.

---

## 5. Версии: semver, /v2, псевдоверсии, retract

### 5.1. Правило совместимости импорта

Главный принцип модулей (Import Compatibility Rule): **если у старого и нового пакета одинаковый путь импорта, новый обязан быть обратно совместим со старым.** Отсюда:

- v0.x.y — без гарантий совместимости;
- v1.x.y — внутри одной major-версии только совместимые изменения;
- несовместимое изменение → новая major-версия → **новый путь импорта** `/v2`.

### 5.2. Major version suffix: /v2

```text
module example.com/logx/v2     ← обязательно для тегов v2.x.y
```

Пакеты v1 и v2 — **разные модули** и могут жить в одной сборке одновременно:

```text
$ go get example.com/logx/v2@v2.0.0
go: added example.com/logx/v2 v2.0.0
$ go list -m all
example.com/orders
example.com/kafka v1.0.0
example.com/logx v1.2.0
example.com/logx/v2 v2.0.0
example.com/pg v1.1.0
```

(В реальном выводе `go get` была ещё строка с ошибкой про неправильно опубликованную `example.com/logx v2.0.0` из граблей ниже: чтобы найти модуль для пакета `example.com/logx/v2`, Go перебирает кандидатов — `example.com/logx/v2` и `example.com/logx` — и пробует оба.)

Это и решение «diamond dependency problem»: если одна библиотека перешла на logx v2, а другая осталась на v1, сборка не ломается.

**Грабли: тег v2.0.0 без /v2 в go.mod**

Автор поставил тег `v2.0.0`, но в `go.mod` оставил `module example.com/logx`:

```text
$ go get example.com/logx@v2.0.0
go: downloading example.com/logx v2.0.0
go: unzip .../cache/download/example.com/logx/@v/v2.0.0.zip: example.com/logx@v2.0.0: invalid version: should be v0 or v1, not v2
```

Почему так: путь `example.com/logx` без суффикса допускает только v0 и v1. Как исправить (автору): `module example.com/logx/v2` в `go.mod`, обновить все импорты внутри модуля на `.../v2/...`, выпустить новый тег. Альтернатива — подпапка `v2/` со своим `go.mod` (major subdirectory). Как заметить: любой потребитель получит эту ошибку сразу.

> **Совет.** Прежде чем выпускать v2, проверьте, нельзя ли обойтись без несовместимых изменений: новая функция `NewClientWithOptions` рядом со старой дешевле, чем миграция всех потребителей на новый путь импорта. Не нужно, если старый API действительно опасен или необслуживаем.

### 5.3. Псевдоверсии

Для коммита без тега Go вычисляет **псевдоверсию**. Три формы (go.dev/ref/mod):

> - `vX.0.0-yyyymmddhhmmss-abcdefabcdef` is used when there is no known base version…
> - `vX.Y.Z-pre.0.yyyymmddhhmmss-abcdefabcdef` is used when the base version is a pre-release version like `vX.Y.Z-pre`.
> - `vX.Y.(Z+1)-0.yyyymmddhhmmss-abcdefabcdef` is used when the base version is a release version like `vX.Y.Z`.

Реальные примеры (запрос `@master` к `proxy.golang.org`, 22 октября 2026):

```text
$ go list -m golang.org/x/exp@master
golang.org/x/exp v0.0.0-20261014143641-f0251eb4aa3e
$ go list -m github.com/google/uuid@master
github.com/google/uuid v1.6.1-0.20241114170450-2d3c2a9cc518
$ go list -m github.com/jackc/pgx/v5@master
github.com/jackc/pgx/v5 v5.11.1-0.20261021214415-0dd05bbb0340
$ go list -m github.com/redis/go-redis/v9@master
github.com/redis/go-redis/v9 v9.23.1-0.20261021164449-454251f54bc5
```

`golang.org/x/exp` не имеет тегов → форма `v0.0.0-…`. У `uuid` последний тег v1.6.0, а master ушёл вперёд → `v1.6.1-0.…`. Время — UTC-время коммита, хвост — первые 12 символов хеша. С Go 1.13 `go` проверяет, что дата и хеш действительно соответствуют коммиту (release notes 1.13).

**Почему именно так.** Псевдоверсия сортируется **выше** базового тега и **ниже** следующего тега: `v1.6.0 < v1.6.1-0.2024… < v1.6.1`. Поэтому MVS корректно сравнивает её с обычными версиями.

> **Заблуждение:** «Псевдоверсию можно написать руками, например `v1.7.0-…` с любым хешем». С Go 1.13 `go` проверяет соответствие префикса тегам-предкам, даты и длины хеша; самодельная псевдоверсия отвергается. Пишите `go get module@<commit>` — Go вычислит правильную сам.

### 5.4. +incompatible

Репозитории, которые ставили теги v2+ **до** появления модулей и не имеют `go.mod`, получают суффикс `+incompatible`:

```text
$ go list -m -versions github.com/docker/docker | tr ' ' '\n' | grep incompatible | tail -3
v28.5.0+incompatible
v28.5.1+incompatible
v28.5.2+incompatible
```

Спецификация: «+incompatible indicates that a version is part of the same module as versions with lower major version numbers; consequently, the go command may automatically upgrade to higher +incompatible versions even though it may break the build.» То есть для Go это «v1-модуль с большим номером», и MVS может поднять вас с v20 до v28 — несовместимо.

### 5.5. retract: отзыв версии

Автор опубликовал сломанную v1.4.0. Исправить её нельзя (прокси и sumdb хранят версии навсегда), поэтому он выпускает v1.4.1 с директивой `retract`:

```text
module example.com/logx

go 1.22

retract (
	v1.4.0 // ломает формат JSON-логов, используйте v1.3.0
	v1.4.1 // версия содержит только retract
)
```

Сама v1.4.1 тоже отозвана: она нужна только как носитель `retract`. Результат:

```text
$ go list -m -versions example.com/logx
example.com/logx v1.0.0 v1.1.0 v1.2.0 v1.3.0
$ go list -m -versions -retracted example.com/logx
example.com/logx v1.0.0 v1.1.0 v1.2.0 v1.3.0 v1.4.0 v1.4.1
$ go get example.com/logx@latest
$ go get example.com/logx@v1.4.0
go: downloading example.com/logx v1.4.0
go: warning: example.com/logx@v1.4.0: retracted by module author: ломает формат JSON-логов, используйте v1.3.0
go: to switch to the latest unretracted version, run:
	go get example.com/logx@latest
go: upgraded example.com/logx v1.3.0 => v1.4.0
$ go list -m -u example.com/logx
example.com/logx v1.4.0 (retracted)
$ go build
build ok
```

Что видно: `@latest` указывает на v1.3.0 (`go get` ничего не изменил — мы уже на ней); явная v1.4.0 доступна, но с предупреждением; сборка с отозванной версией **работает** — retract ничего не ломает, а только предупреждает. Это совпадает со спецификацией: «Builds that depend on retracted versions should continue to work, but users will be notified».

> **Совет.** Добавьте в CI `go list -m -u -json all` с поиском поля `Retracted` (или просто `go list -m -u all | grep -E 'retracted|deprecated'`) — так вы узнаете об отзыве версии, которую уже используете. Не нужно, если зависимости обновляются ботом, который сам показывает retract.

### Простыми словами

Версия v2 — это для Go другое имя, поэтому старый и новый код могут жить рядом. Коммит без тега получает «псевдоверсию» с датой и хешем. Если автор выпустил плохую версию, удалить её нельзя, но можно повесить табличку «не берите» — `retract`.

### Что важно для Senior

- v2+ требует `/vN` в пути модуля и во всех импортах. Ошибка «should be v0 or v1, not v2» — это забытый суффикс.
- Псевдоверсия всегда между базовым и следующим тегом; генерировать её надо `go get mod@commit`, а не руками.
- `+incompatible` = модуль без `go.mod` с тегами v2+; MVS свободно поднимает такие версии, а значит может сломать сборку.
- Отозванная версия продолжает собираться; отзыв видно в `go get` и `go list -m -u`.

---
## 6. go.sum и проверка целостности

### 6.1. Что лежит в go.sum

```text
example.com/kafka v1.0.0 h1:obEJD1MMjGbifKw37doDODCvkEgZ1yEfXpuMCOz1seM=
example.com/kafka v1.0.0/go.mod h1:pVbzU2TxVCwvnadfzdMHRnUe4l8e+/STOGj/3dZxhRg=
example.com/logx v1.2.0 h1:ImS023vIi87Csc7h1SlY9/np251DVor1Td4mXO5Wbro=
example.com/logx v1.2.0/go.mod h1:52DOL9upR4pOeAS+Mw9jH3u8hIDOocPQ8EKRKvmuxy4=
example.com/pg v1.0.0 h1:FO/PtELbRA099JpG60mci34M2hglzuAMxT8p7FzCTYc=
example.com/pg v1.0.0/go.mod h1:6n9VgSJIo6u6DoPiDahQ1yVhY8j2BaBnxFjSi6OkiCQ=
```

Спецификация: «If the version ends with /go.mod, the hash is for the module's go.mod file only; otherwise, the hash is for the files within the module's .zip file.» Две строки на версию, потому что для построения графа нужен только `go.mod` (его хеш), а для сборки — весь код. Модулям из обрезанной части графа достаточно строки `/go.mod` (§4.2).

**Формат h1.** Из `src/cmd/vendor/golang.org/x/mod/sumdb/dirhash/hash.go`:

```go
// Hash1 is "h1:" followed by the base64-encoded SHA-256 hash of a summary
// prepared as if by the Unix command:
//
//	sha256sum $(find . -type f | sort) | sha256sum
```

Проверка руками — пересчитываем хеш zip-архива `logx v1.2.0` shell-командами:

```bash
unzip -q v1.2.0.zip
(for f in $(find . -type f | sed 's|^\./||' | LC_ALL=C sort); do
  printf '%s  %s\n' "$(shasum -a 256 < $f | cut -d' ' -f1)" "$f"
done) | shasum -a 256 | cut -d' ' -f1 | xxd -r -p | base64
```

```text
94edcf06b4a9ec6bee1de19f1a5e9d4ac01df4620233196737e3e83159ee49cc  example.com/logx@v1.2.0/go.mod
b385e1118ff5de7351c18255c1e25ea6d29e5f754e813520cad60aebbcf93210  example.com/logx@v1.2.0/logx.go
ImS023vIi87Csc7h1SlY9/np251DVor1Td4mXO5Wbro=
```

Совпало с `go.sum`. Имена файлов в сводке — с префиксом `module@version/`, как в zip-архиве.

> **Неожиданное при проверке.** Функция `HashZip` хеширует **все записи zip, включая записи-папки** (как пустые файлы). Первые архивы для экспериментов были собраны `zip -r` с записями-папками, и `go mod verify` сообщал `dir has been modified` для модулей, которые никто не трогал: хеш zip и хеш распакованной папки расходились. Настоящие прокси раздают канонические архивы без записей-папок (их собирает `golang.org/x/mod/zip`), поэтому в жизни это не проявляется. Но если вы сами делаете прокси или зеркало, собирайте zip через `golang.org/x/mod/zip`, а не `zip -r`.

### 6.2. Checksum database и transparency log

Когда модуля ещё нет в `go.sum`, `go` (функция `checkModSum` в `src/cmd/go/internal/modfetch/fetch.go`) идёт в **checksum database**:

```go
// Not listed, so we want to add them.
// Consult checksum database if appropriate.
if useSumDB(mod) {
	// Calls base.Fatalf if mismatch detected.
	if err := checkSumDB(mod, h); err != nil {
```

```mermaid
sequenceDiagram
  participant G as go command
  participant P as proxy.golang.org
  participant S as sum.golang.org
  G->>G: есть ли mod@v в go.sum?
  alt есть
    G->>P: скачать zip
    G->>G: сравнить h1 с go.sum
  else нет
    G->>P: скачать zip
    G->>S: lookup mod@v (+ доказательство включения в журнал)
    G->>G: проверить подпись журнала и h1
    G->>G: дописать строки в go.sum
  end
```

`sum.golang.org` — **transparency log**: журнал «только на дописывание» на основе дерева Меркла (пакет `golang.org/x/mod/sumdb/tlog`). Каждый ответ сервера сопровождается доказательством, что запись входит в подписанное дерево, а публичный ключ журнала зашит в `go`-команду (`src/cmd/go/internal/modfetch/key.go`):

```go
"sum.golang.org": "sum.golang.org+033de0ae+Ac4zctda0e5eza+HJyk9SxEdh+s3Ux18htTTAD8OuAn8",
```

**Что это даёт.** Первая загрузка версии кем угодно в мире фиксирует её хеш навсегда. Если автор (или взломщик) потом перезапишет тег, все получат ошибку. Если сервер журнала попробует показать разным клиентам разную историю, это можно доказать сравнением подписанных состояний дерева. **⚠️ не проверено запуском:** формат ответов `sum.golang.org` и проверку доказательств включения я не трассировал, описание — по коду пакетов `sumdb` и `tlog`.

### 6.3. Воспроизведение: подмена кода на прокси

Подменяем код `logx v1.2.0` в прокси (дописываем «вредоносную» строку), очищаем кэш и собираем:

```text
$ go build
verifying example.com/logx@v1.2.0: checksum mismatch
	downloaded: h1:HAQmkwdHkx1AmsLf9+hjrd5h2CaxwjFRXTob+X7CuQE=
	go.sum:     h1:ImS023vIi87Csc7h1SlY9/np251DVor1Td4mXO5Wbro=

SECURITY ERROR
This download does NOT match an earlier download recorded in go.sum.
The bits may have been replaced on the origin server, or an attacker may
have intercepted the download attempt.

For more information, see 'go help module-auth'.
```

**Грабли: «починить» SECURITY ERROR удалением строки из go.sum**

Частая реакция — удалить строку из `go.sum` и запустить `go mod tidy`. Для публичного модуля Go сверится с `sum.golang.org` и снова откажется, а вот для приватного (`GONOSUMDB`/`GOPRIVATE`) или при `GOSUMDB=off` — молча примет подменённый код. Как правильно: выяснить, почему изменился архив (перезаписанный тег, подмена на внутреннем прокси), и только потом обновлять хеш.

### 6.4. go.sum без строк

```text
$ go build (без go.sum)
main.go:6:2: missing go.sum entry for module providing package example.com/kafka (imported by example.com/orders); to add:
	go get example.com/orders
main.go:7:2: missing go.sum entry for module providing package example.com/pg (imported by example.com/orders); to add:
	go get example.com/orders
```

С Go 1.16 сборка не дописывает `go.sum` сама (`-mod=readonly`), поэтому ошибка, а не тихая загрузка.

### 6.5. Конфликты слияния в go.mod и go.sum

Две ветки добавили разные зависимости:

```text
$ git merge feat-pg
Auto-merging go.sum
CONFLICT (add/add): Merge conflict in go.sum
Automatic merge failed; fix conflicts and then commit the result.
$ cat go.sum
<<<<<<< HEAD
example.com/kafka v1.0.0 h1:obEJD1MMjGbifKw37doDODCvkEgZ1yEfXpuMCOz1seM=
example.com/kafka v1.0.0/go.mod h1:pVbzU2TxVCwvnadfzdMHRnUe4l8e+/STOGj/3dZxhRg=
example.com/logx v1.1.0 h1:Jt12HBfokBPTkcMhgOZaoCKGJTqjo9txgoY4n7JB+nI=
example.com/logx v1.1.0/go.mod h1:52DOL9upR4pOeAS+Mw9jH3u8hIDOocPQ8EKRKvmuxy4=
=======
example.com/logx v1.2.0 h1:ImS023vIi87Csc7h1SlY9/np251DVor1Td4mXO5Wbro=
example.com/logx v1.2.0/go.mod h1:52DOL9upR4pOeAS+Mw9jH3u8hIDOocPQ8EKRKvmuxy4=
example.com/pg v1.1.0 h1:agWVyUUBVuVJu546ylyv9ofEDBeRv/cB20wqwvtNXQk=
example.com/pg v1.1.0/go.mod h1:WMbs7uqgRVeOHXj3/DqqBRtlaysQWd4pQ/2qH/Z0dnk=
>>>>>>> feat-pg
```

Если забыть убрать маркеры из `go.mod`, Go скажет прямо:

```text
go: errors parsing go.mod:
go.mod:5: unknown directive: <<<<<<<
go.mod:9: unknown directive: =======
go.mod:13: unknown directive: >>>>>>>
```

**Как разрешать:** оставить **обе** стороны (просто удалить строки-маркеры) и в `go.mod`, и в `go.sum`, затем `go mod tidy`:

```text
$ go mod tidy && cat go.mod
module example.com/orders

go 1.27.0

require (
	example.com/kafka v1.0.0
	example.com/pg v1.1.0
)

require example.com/logx v1.2.0 // indirect
$ go mod verify
all modules verified
```

Лишние строки (`logx v1.1.0`) `tidy` удалил сам, а MVS выбрал v1.2.0. Обе стороны безопасно объединять, потому что требования в `go.mod` — только нижние границы, а `go.sum` — просто множество хешей.

> **Совет.** Добавьте в `.gitattributes` строку `go.sum merge=union`. Тогда git объединяет строки `go.sum` без конфликта; `go mod tidy` в CI потом удалит лишнее. **⚠️ не проверено запуском** именно в этой сессии: я проверял ручное объединение, а не драйвер `union`.

### Простыми словами

`go.sum` — это «отпечатки пальцев» каждой версии зависимостей. Если код на сервере изменили, отпечаток не совпадёт, и Go остановит сборку. А общий публичный журнал `sum.golang.org` гарантирует, что у всех в мире отпечаток для одной версии одинаковый.

### Что важно для Senior

- `go.sum` — не lock-файл: версии выбирает `go.mod` + MVS, а `go.sum` только проверяет содержимое.
- Хеш h1 — SHA-256 от отсортированного списка SHA-256 файлов архива; его можно пересчитать без Go.
- Для публичных модулей подмену ловит `sum.golang.org` даже без `go.sum`; для приватных единственная защита — закоммиченный `go.sum`.
- Конфликты в `go.sum` разрешаются объединением строк и `go mod tidy`.

---

## 7. Откуда берутся модули: прокси, sumdb, приватные репозитории

### 7.1. Протокол прокси

Прокси модулей — это обычный HTTP (или `file://`) с фиксированной схемой путей. Для экспериментов этого документа хватило папки на диске:

```text
proxy/example.com/logx/@v/list          ← список версий, по одной на строку
proxy/example.com/logx/@v/v1.2.0.info   ← {"Version":"v1.2.0","Time":"..."}
proxy/example.com/logx/@v/v1.2.0.mod    ← go.mod этой версии
proxy/example.com/logx/@v/v1.2.0.zip    ← код, все файлы с префиксом example.com/logx@v1.2.0/
```

и переменных `GOPROXY=file:///.../proxy`, `GONOSUMDB=example.com` (выдуманных модулей нет в `sum.golang.org`). Именно поэтому корпоративные прокси (Athens, JFrog Artifactory, Nexus, GitLab) так легко реализовать.

### 7.2. GOPROXY, GOSUMDB и порядок

Значения по умолчанию лежат в файле `go.env` в корне GOROOT:

```text
GOPROXY=https://proxy.golang.org,direct
GOSUMDB=sum.golang.org
GOTOOLCHAIN=auto
```

- `GOPROXY` — список через запятую или `|`. `direct` — идти прямо в VCS (git); `off` — запрет загрузок. Через запятую следующий прокси пробуется только при 404/410; через `|` — при любой ошибке (поле `fallBackOnError` в `src/cmd/go/internal/modfetch/proxy.go`: `fallBackOnError = goproxy[i] == '|'`). Release notes 1.13: «An unreachable proxy or HTTP status code other than 404 or 410 terminates the search without consulting the remaining proxies».
- `GOSUMDB` — имя (и ключ) checksum database или `off`.
- `GOFLAGS` — флаги по умолчанию для всех команд, например `GOFLAGS=-mod=mod` или `-modcacherw`.

### 7.3. Приватные модули: GOPRIVATE, GONOPROXY, GONOSUMDB

Из `go help private`:

> The GOPRIVATE environment variable controls which modules the go command considers to be private (not available publicly) and should therefore not use the proxy or checksum database. … the GONOPROXY and GONOSUMDB environment variables accept the same kind of glob list and override GOPRIVATE for the specific decision of whether to use the proxy and checksum database, respectively.

> **Исправление к формулировке задания.** Переменной `GONOSUMCHECK` в Go нет. Полный список переменных про загрузку модулей в `src/cmd/go/internal/cfg`: `GOPROXY`, `GONOPROXY`, `GOSUMDB`, `GONOSUMDB`, `GOPRIVATE`, `GOINSECURE`, `GOVCS`, `GOAUTH`, `GOMODCACHE`.

**Грабли: имя приватного модуля уходит в публичный прокси**

```text
### без GOPRIVATE
$ go mod download -x github.com/acme-corp-zq81/billing@v1.0.0
# get https://proxy.golang.org/github.com/acme-corp-zq81/billing/@v/v1.0.0.info
# get https://proxy.golang.org/github.com/acme-corp-zq81/billing/@v/v1.0.0.info: 404 Not Found (3.834s)
... дальше git ls-remote https://github.com/acme-corp-zq81/billing

### GOPRIVATE=github.com/acme-corp-zq81
$ go mod download -x github.com/acme-corp-zq81/billing@v1.0.0
... сразу git ls-remote, запроса к proxy.golang.org нет
```

(Вывод сокращён: убраны служебные строки `mkdir`/`lock`.)

Без `GOPRIVATE` полный путь модуля (в нём часто имя компании, проекта, команды) отправлен на `proxy.golang.org`, а затем был бы запрошен у `sum.golang.org`. Спецификация прямо предупреждает: «it does transmit the full module path being requested». Плюс 3,8 с ожидания 404 на каждую загрузку. Как исправить: `GOPRIVATE=*.corp.example.com,github.com/acme-corp` у разработчиков (через `go env -w`) и в CI.

Типовая корпоративная схема из `go help private`:

```text
GOPRIVATE=*.corp.example.com
GOPROXY=proxy.example.com
GONOPROXY=none
```

Здесь приватные модули тоже идут через свой прокси (`GONOPROXY=none` отменяет «не через прокси» из `GOPRIVATE`), но по-прежнему не проверяются в публичной sumdb.

### 7.4. Аутентификация: .netrc, git credential, GOAUTH (Go 1.24)

`GOAUTH` появился в Go 1.24 (release notes: «provides a flexible way to authenticate private module fetches»). Из `go help goauth` (go1.27.0):

> GOAUTH is a semicolon-separated list of authentication commands for go-import and HTTPS module mirror interactions. The default is netrc.

| Значение | Что делает |
|---|---|
| `netrc` (по умолчанию) | Логин/пароль из `$NETRC` или `~/.netrc`. |
| `git dir` | Запускает `git credential fill` в папке `dir`. |
| `command args…` | Ваша команда печатает URL-префиксы и HTTP-заголовки (например, `Authorization: Bearer ...`). |
| `off` | Без аутентификации. |

Для `direct`-загрузки из git работает обычная настройка git: `git config --global url."https://oauth2:${TOKEN}@gitlab.corp.example.com/".insteadOf "https://gitlab.corp.example.com/"` или SSH. **⚠️ не проверено запуском:** аутентификацию к настоящему приватному серверу в этой сессии не проверял — нет приватного репозитория.

### 7.5. GOVCS и GOINSECURE

- `GOVCS` (Go 1.16) ограничивает, какие VCS можно запускать; по умолчанию `public:git|hg,private:all` (комментарий в `src/cmd/go/internal/vcs/vcs_test.go`, значение `defaultGOVCS` в `vcs.go`). С Go 1.27 поддержка `bzr` удалена совсем.
- `GOINSECURE` — разрешить HTTP без TLS для указанных путей. В production не нужен.

### Простыми словами

По умолчанию Go качает модули с публичного склада Google и сверяет отпечатки с публичным журналом. Для своих закрытых модулей нужно сказать Go «это наше, никому не показывай» — `GOPRIVATE`. Иначе Go сначала спросит про ваш модуль у публичного склада.

### Что важно для Senior

- `GOPRIVATE` — это два решения сразу: «не через публичный прокси» (`GONOPROXY`) и «не в публичную sumdb» (`GONOSUMDB`).
- Без него имя приватного модуля утекает в `proxy.golang.org`/`sum.golang.org` и каждая загрузка ждёт 404.
- Через запятую в `GOPROXY` следующий источник пробуется только при 404/410 — «упавший» корпоративный прокси не откатится на `direct`.
- Аутентификация: `.netrc` по умолчанию, `GOAUTH` (1.24) для токенов и credential helper'ов.

---

## 8. Кэш модулей

### 8.1. Устройство

```text
$GOMODCACHE (по умолчанию $GOPATH/pkg/mod = ~/go/pkg/mod)
├── cache/download/example.com/logx/@v/   ← list, .info, .mod, .zip, .ziphash — как у прокси
├── cache/download/sumdb/                 ← закэшированные данные журнала sum.golang.org (modfetch/sumdb.go)
├── cache/vcs/                            ← bare-клоны git для GOPROXY=direct
├── example.com/logx@v1.2.0/              ← распакованный код, только для чтения
└── golang.org/toolchain@v0.0.1-go1.27.0.darwin-arm64/   ← скачанные тулчейны (§11)
```

Папка `cache/download` устроена в формате прокси, поэтому её можно отдавать как `GOPROXY=file://$GOMODCACHE/cache/download` — удобно для офлайн-сборок.

### 8.2. Только для чтения

```text
$ ls -ld $GOMODCACHE/example.com/logx@v1.2.0 $GOMODCACHE/example.com/logx@v1.2.0/logx.go
dr-xr-xr-x@ 4 r  wheel  128 Oct 23 01:17 .../example.com/logx@v1.2.0
-r--r--r--@ 1 r  wheel   56 Oct 23 01:17 .../example.com/logx@v1.2.0/logx.go
$ rm -rf $GOMODCACHE
rm: .../example.com/logx@v1.2.0/go.mod: Permission denied
rm: .../example.com/logx@v1.2.0/logx.go: Permission denied
rm: .../example.com/logx@v1.2.0: Directory not empty
$ go clean -modcache
$ ls $GOMODCACHE
ls: .../gopath/pkg/mod: No such file or directory
```

Спецификация: «The go command creates module source files and directories in the cache with read-only permissions to prevent accidental changes to modules after they're downloaded». Флаг `-modcacherw` (или `GOFLAGS=-modcacherw`) создаёт папки с правами на запись — ценой риска, что IDE или тест испортят кэш.

**Грабли: правка в кэше незаметна для сборки**

Отладочная правка «прямо в зависимости» через `chmod u+w`:

```text
$ chmod u+w $GOMODCACHE/example.com/logx@v1.2.0/logx.go
$ echo '// debug' >> $GOMODCACHE/example.com/logx@v1.2.0/logx.go
$ go mod verify
example.com/logx v1.2.0: dir has been modified (.../example.com/logx@v1.2.0)
exit=1
$ go build -o /dev/null . && echo "build ok"
build ok
```

`go build` не перепроверяет хеши распакованных папок (иначе каждая сборка хешировала бы весь кэш), а `go mod verify` — проверяет. Как заметить: `go mod verify` в CI. Как правильно отлаживать зависимость: `go work use` с локальным клоном (§10) или временный `replace` на папку.

> **Совет.** В CI запускайте `go mod verify` после восстановления кэша модулей из артефакта: кэш мог быть испорчен прошлым шагом. Стоимость — хеширование всех зависимостей один раз. Не нужно, если кэш каждый раз скачивается с нуля.

### Простыми словами

Скачанные модули складываются в общую папку на компьютере и запираются «только для чтения», чтобы их никто случайно не испортил. Удалять папку — командой `go clean -modcache`, а не `rm`.

### Что важно для Senior

- Один кэш на машину для всех проектов; ключ — `модуль@версия`, поэтому дублирования нет.
- `cache/download` — готовый файловый прокси для офлайн-сборок.
- `go build` доверяет кэшу, `go mod verify` — нет. Испорченный кэш в CI ловится только `verify`.
- Удаление — `go clean -modcache` (или `chmod -R u+w` перед `rm`).

---

## 9. Vendoring

### 9.1. Как это выглядит

```text
$ go mod vendor
$ find vendor -type f | sort
vendor/example.com/kafka/kafka.go
vendor/example.com/logx/logx.go
vendor/example.com/pg/pg.go
vendor/modules.txt
$ cat vendor/modules.txt
# example.com/kafka v1.0.0
## explicit; go 1.22
example.com/kafka
# example.com/logx v1.2.0
## explicit; go 1.22
example.com/logx
# example.com/pg v1.1.0
## explicit; go 1.22
example.com/pg
```

В `vendor/` копируются только **нужные пакеты** (не модули целиком, без тестов и без `go.mod` зависимостей). `modules.txt` фиксирует, из какой версии модуля пришёл каждый пакет; `## explicit` значит «требуется в `go.mod` явно».

### 9.2. Когда vendor включается сам

Из `src/cmd/go/internal/modload/init.go`:

```go
// The Go version is at least 1.14, a vendor directory exists, and
// the modules.txt was generated in the same mode the command is running in.
// Set -mod=vendor by default.
cfg.BuildMod = "vendor"
cfg.BuildModReason = "Go version in " + versionSource + " is at least 1.14 and vendor directory exists."
```

Иначе — `cfg.BuildMod = "readonly"`. Сборка с vendor **не обращается к сети и кэшу модулей**:

```text
$ go clean -modcache
$ GOPROXY=off go build (кэш модулей пуст)
ok
```

**Грабли: правка go.mod без go mod vendor**

```text
$ sed -i '' 's/example.com\/pg v1.1.0/example.com\/pg v1.0.0/' go.mod
$ go build
go: inconsistent vendoring in .../work/vend:
	example.com/pg@v1.0.0: is explicitly required in go.mod, but not marked as explicit in vendor/modules.txt
	example.com/pg@v1.1.0: is marked as explicit in vendor/modules.txt, but not explicitly required in go.mod

	To ignore the vendor directory, use -mod=readonly or -mod=mod.
	To sync the vendor directory, run:
		go mod vendor
```

Как заметить: сборка падает сразу. Как исправить: `go mod vendor` после каждого `go get`/`go mod tidy`; в CI — `go mod vendor && git diff --exit-code vendor/`.

### 9.3. Vendor или нет

| | С `vendor/` | Без (кэш + прокси) |
|---|---|---|
| Сборка без сети | да, из коробки | нужен прогретый кэш или свой прокси |
| Размер репозитория | растёт (иногда на сотни МБ) | только `go.mod`/`go.sum` |
| Ревью зависимостей | diff кода зависимостей виден в PR | только смена версии |
| Защита от исчезновения модуля | полная | `proxy.golang.org` хранит версии, но для `direct`/приватных — нет |
| Скорость Docker-сборки | не нужен слой `go mod download` | нужен кэш (§18.1) |

> **Совет.** Vendor оправдан, когда сборка обязана идти без сети (изолированный контур, требования аудита) или зависимости из `direct`-источников, которые могут исчезнуть. Не нужен, если есть корпоративный прокси с хранением версий: он даёт ту же надёжность без раздувания репозитория.

### Простыми словами

`go mod vendor` кладёт копию кода всех нужных пакетов прямо в проект. Тогда сборке не нужен интернет. Но копию нужно обновлять вместе с `go.mod`, иначе Go откажется собирать.

### Что важно для Senior

- Vendor включается автоматически при `go ≥ 1.14` и наличии `vendor/`; выключить — `-mod=mod` или `-mod=readonly`.
- `vendor/modules.txt` — источник истины для режима vendor; рассинхрон с `go.mod` — фатальная ошибка.
- С Go 1.22 есть `go work vendor` для workspaces; содержимое `vendor/` для workspace и для одного модуля различается.
- С go 1.23 (`ExplicitModulesTxtImportVersion` в `gover/version.go`) пакеты из vendor можно импортировать, только если они перечислены в `modules.txt`.

---

## 10. Workspaces: go.work

### 10.1. Зачем

Вы чините баг в своей библиотеке `logx` и хотите сразу проверить фикс в сервисе `orders`, не публикуя версию. Раньше — временный `replace` в `go.mod`, который потом забывали удалить (§2.3). С Go 1.18 — `go.work`.

```text
$ go work init ./orders ./libs/logx
$ cat go.work
go 1.27.0

use (
	./libs/logx
	./orders
)
$ go run ./orders
kafka uses logx local-fix (не опубликовано)
pg uses logx local-fix (не опубликовано)
$ cd orders && GOWORK=off go run .
kafka uses logx v1.2.0
pg uses logx v1.2.0
$ go env GOWORK
.../work/ws/go.work
$ go list -m
example.com/logx
example.com/orders
```

Что видно: в workspace оба модуля — **главные** (`go list -m` выводит оба), и локальная `logx` заменяет опубликованную для всех, кто её импортирует — и для `kafka`, и для `pg`. `GOWORK=off` выключает workspace для одной команды.

### 10.2. Правила

- `go.work` ищется в текущей папке и выше, как `go.mod`; или задаётся `GOWORK=/path/go.work`.
- В `go.work` есть `go`, `toolchain`, `godebug`, `use`, `replace` (список глаголов `go.work` — в том же `modfile/rule.go`). `replace` в `go.work` перекрывает `replace` модулей.
- `go work sync` поднимает требования в `go.mod` модулей workspace до выбранных MVS версий.
- `go.work.sum` хранит хеши, которых нет в `go.sum` модулей.

**Грабли: закоммиченный go.work**

Если `go.work` попал в репозиторий, CI тоже соберёт «локальную» версию библиотеки (или упадёт, если папки нет), а `go.mod` сервиса может быть несобираемым отдельно — никто этого не заметит до релиза. Как заметить: `git ls-files | grep go.work`. Как исправить: добавить `go.work` и `go.work.sum` в `.gitignore`; в CI собирать с `GOWORK=off`. Исключение — монорепозиторий, где workspace описывает весь репозиторий осознанно (§16.3).

### Простыми словами

`go.work` — это «рабочий стол», на котором лежат несколько ваших модулей сразу. Go собирает их вместе, как будто это один проект, и берёт локальный код вместо опубликованного. На чужой компьютер этот рабочий стол обычно не переносят.

### Что важно для Senior

- Workspace делает несколько модулей главными: их `replace`/`exclude` и локальный код действуют одновременно.
- `go.work` — инструмент разработчика, по умолчанию его не коммитят; CI — с `GOWORK=off`.
- `go work sync` переносит итоговые версии обратно в `go.mod` модулей.
- Шаблон пакетов `work` (Go 1.25) — «все пакеты всех главных модулей».

---
## 11. Версия Go: директивы go и toolchain, GOTOOLCHAIN, godebug

### 11.1. Строка go

С Go 1.21 (release notes): «Go 1.21 now reads the go line in a go.work or go.mod file as a strict minimum requirement: go 1.21.0 means that the workspace or module cannot be used with Go 1.20 or with Go 1.21rc1.» До 1.21 строка `go` была «подсказкой»: старый Go пытался собрать код и падал с непонятными ошибками синтаксиса.

Строка `go` управляет одновременно:
- версией **языка** (например, семантика переменной цикла с 1.22, `range` по функциям с 1.23);
- минимальной версией тулчейна;
- pruning (§4), vendor по умолчанию (§9), значениями GODEBUG (§11.5), набором проверок `go vet` (с Go 1.27 `go test` запускает проверку `stdversion` — использование слишком новых функций стандартной библиотеки для указанной `go`).

**Расхождение release notes и кода: что пишет go mod init.** Release notes Go 1.26: «Running go mod init using a toolchain of version 1.N.X will create a go.mod file specifying the Go version go 1.(N-1).0». Проверка на go1.27.0:

```text
$ go mod init example.com/init1
$ cat go.mod
module example.com/init1

go 1.27.0
```

История в git: коммит `337f7b1f5d` («update default go directive in mod or work init») вошёл в go1.26rc1, а коммит `44ec057a3e` от 2026-02-27 («revert update default go directive in mod or work init… restores the previous behavior of setting go directive to the toolchain's version as per #77653») — в go1.27rc1; бэкпорт `e28ac674af` — в **go1.26.1**. Итог: N-1 писал только go1.26.0. **Верить коду**: в go1.26.1+ и go1.27 `go mod init` пишет версию текущего тулчейна. Чтобы поддержать старые версии Go, понизьте строку сами: `go get go@1.25.0`.

### 11.2. Директива toolchain и GOTOOLCHAIN

`toolchain go1.27.2` — «предпочтительный тулчейн, не ниже». В отличие от `go`, на потребителей не влияет (§2.2). С Go 1.25 `go` больше не дописывает строку `toolchain` автоматически при поднятии `go` (release notes 1.25).

Значение по умолчанию `GOTOOLCHAIN=auto` лежит в `go.env`. Возможные значения (`src/cmd/go/internal/toolchain/select.go`):

| GOTOOLCHAIN | Поведение |
|---|---|
| `local` | Только встроенный тулчейн; если `go.mod` требует новее — ошибка. |
| `auto` (= `local+auto`) | Встроенный, но если `go`/`toolchain` в `go.mod` требуют новее — найти в `PATH` или **скачать**. |
| `path` (= `local+path`) | Как `auto`, но только из `PATH`, без скачивания. |
| `go1.27.2` | Всегда этот тулчейн (скачать, если нужно). |
| `go1.27.2+auto` | Минимум go1.27.2, выше — по `go.mod`. |

Скачанный тулчейн — это **обычный модуль** `golang.org/toolchain` с версией вида `v0.0.1-go1.27.0.darwin-arm64` (константы `gotoolchainModule`, `gotoolchainVersion` в `select.go`), он идёт через `GOPROXY` и проверяется по `GOSUMDB`, а лежит в кэше модулей.

Трассировка выбора (`GODEBUG=toolchaintrace=1`, Go 1.24). Системный `go` — 1.26.1, в `go.mod` — `go 1.27.0`:

```text
$ go version   # это go1.26.1 из Homebrew
go: upgrading toolchain to go1.27.0 (required by go line in go.mod; upgrade allowed by GOTOOLCHAIN=auto)
go: using go1.27.0 toolchain from cache located at /Users/r/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.27.0.darwin-arm64/bin/go
go version go1.27.0 darwin/arm64
```

Именно так этот документ и собирался: системный Go 1.26.1 молча передавал управление 1.27.0.

**Грабли: GOTOOLCHAIN=local и новый go.mod**

```text
$ GOTOOLCHAIN=local go build
go: go.mod requires go >= 1.99.0 (running go 1.27.0; GOTOOLCHAIN=local)
```

**Грабли: версия языка вместо версии тулчейна**

```text
$ GOTOOLCHAIN=go1.21 go version
go: downloading go1.21 (darwin/arm64)
go: invalid toolchain: go1.21 is a language version but not a toolchain version (go1.21.x)
```

Почему: с 1.21 первый релиз называется `1.N.0` («Starting in Go 1.21, the first release is now Go 1.N.0»), а `1.21` — это версия языка. В `go.mod` строка `go 1.21` допустима (язык), а `GOTOOLCHAIN=go1.21` и `toolchain go1.21` — нет. Как исправить: `go1.21.0`.

> **Заблуждение:** «Версия Go в CI — это то, что установлено в образе». С Go 1.21 при `GOTOOLCHAIN=auto` (по умолчанию) реальная версия определяется `go.mod`: образ с Go 1.26 скачает 1.27, если так требует `go.mod`. Проверяйте `go version` **в папке модуля**, а не в `/`.

> **Совет.** В герметичных сборках (без интернета, с жёстко заданным образом) ставьте `GOTOOLCHAIN=local`: сборка упадёт сразу с понятной ошибкой, вместо попытки скачать тулчейн. Не нужно, если вы как раз хотите, чтобы `go.mod` управлял версией Go.

### 11.3. Как это устроено в MVS

В `go mod graph` (§3.2) видны узлы `go@1.27.0` и `toolchain@go1.27.0`: версия Go — это «модуль» в графе, и MVS выбирает максимум из всех `go`-строк зависимостей. Отсюда правило `GoStrictVersion = "1.21"`: строка `go` главного модуля не может быть ниже, чем у зависимостей (§4.3).

### 11.4. Пример: поднять минимум

```text
go get go@1.27.0          # поднять строку go
go get toolchain@go1.27.2 # задать предпочтительный тулчейн
go get toolchain@none     # убрать строку toolchain
```

### 11.5. Директива godebug (Go 1.23)

Release notes 1.21 формализовали правило: если исправление может сломать старые программы, новое поведение включается по строке `go` главного модуля, а старое можно вернуть через GODEBUG. С Go 1.23 эти значения можно зафиксировать в `go.mod`:

```text
### godebug asynctimerchan=0
$ go list -f '{{.DefaultGODEBUG}}' .
asynctimerchan=0

### godebug asynctimerchan=1
$ go list -f '{{.DefaultGODEBUG}}' .
go: error loading go.mod:
go.mod:5: removed GODEBUG "asynctimerchan" set to old value "1" (https://go.dev/doc/godebug#go-127)

### godebug ( default=go1.21  http2client=0 )
$ go list -f '{{.DefaultGODEBUG}}' . | tr ',' '\n' | head -8
containermaxprocs=0
cryptocustomrand=1
decoratemappings=0
gotestjsonbuildtext=1
http2client=0
httpcookiemaxnum=0
httplaxcontentlength=1
httpmuxgo121=1
```

Что видно:
- `default=go1.21` включает **все** значения GODEBUG так, как они были в Go 1.21, хотя `go 1.27.0` (например, `httpmuxgo121=1` — старый маршрутизатор `net/http`, `containermaxprocs=0` — без учёта лимита CPU контейнера в `GOMAXPROCS`);
- настройка `asynctimerchan` удалена в Go 1.27 (`Removed: 27` в `src/internal/godebugs/table.go`, «Go 1.27 removed the asynctimerchan setting» в `doc/godebug.md`). Новое правило Go 1.27: удалённая настройка в финальном значении (`=0`) **принимается**, в старом (`=1`) — ошибка сборки. Release notes 1.27: «It accepts these settings if they are set to the final default value established before the setting was removed. If they are set to an old value, the go command will fail.»

`godebug` действует только в главном модуле (и в `go.work`), значения записываются в бинарник; переменная окружения `GODEBUG` при запуске их перекрывает.

### Простыми словами

Строка `go` в `go.mod` — это «минимальная версия Go, на которой это работает». Если у вас Go старее, он сам скачает нужный (если разрешено). А `godebug` позволяет сказать «обнови Go, но оставь старое поведение вот этой функции».

### Что важно для Senior

- С 1.21 `go` — жёсткий минимум, а `GOTOOLCHAIN=auto` скачивает тулчейны как модули через `GOPROXY`.
- `go 1.21` (язык) ≠ `go1.21.0` (тулчейн). `GOTOOLCHAIN=go1.21` — ошибка.
- Поднятие `go` меняет не только синтаксис, но и значения GODEBUG — это поведенческое изменение, его надо тестировать.
- Удалённые GODEBUG-настройки в Go 1.27 принимаются только в финальном значении.

---

## 12. Инструменты: директива tool и go install

### 12.1. tool вместо tools.go (Go 1.24)

Раньше версии инструментов (генераторы, линтеры) фиксировали так:

```go
//go:build tools

package tools

import _ "github.com/sqlc-dev/sqlc/cmd/sqlc"
```

С Go 1.24 — директива `tool` (release notes 1.24: «This removes the need for the previous workaround of adding tools as blank imports to a file conventionally named "tools.go"»):

```text
$ go get -tool example.com/sqlgen@v1.0.0
go: upgraded example.com/logx v1.1.0 => v1.3.0
go: added example.com/sqlgen v1.0.0
$ cat go.mod
module example.com/tags

go 1.27.0

require example.com/kafka v1.0.0

require (
	example.com/logx v1.3.0 // indirect
	example.com/sqlgen v1.0.0 // indirect
)

tool example.com/sqlgen
$ go tool | tail -1
sqlgen (example.com/sqlgen)
$ go tool sqlgen -schema db.sql
sqlgen: generating queries, args [-schema db.sql] logx v1.3.0
```

Полезное: `go get tool` — обновить все инструменты, `go install tool` — установить их в `GOBIN`. Директива требует `go ≥ 1.24` в `go.mod` (`GoModToolVersion = "1.24"` в `gover/version.go`).

### 12.2. Зависимости инструментов смешиваются с зависимостями приложения

**Грабли: инструмент поднял версию библиотеки в приложении**

В выводе выше первая строка — `go: upgraded example.com/logx v1.1.0 => v1.3.0`. Приложению (через kafka) нужна `logx ≥ v1.1.0`, генератору `sqlgen` — `≥ v1.3.0`. Граф у главного модуля один, MVS выбрал v1.3.0 — **и в бинарник сервиса поедет v1.3.0**, хотя код сервиса об этом не просил. С крупными инструментами (golangci-lint, протогенераторы) это десятки «лишних» обновлений, в том числе `golang.org/x/*` и gRPC.

Как заметить: строки `upgraded` в выводе `go get -tool`, diff `go.mod`. Как исправить — отдельный файл модуля для инструментов:

```text
$ printf 'module example.com/tags/tools\n\ngo 1.27.0\n' > tools.mod
$ go get -modfile=tools.mod -tool example.com/sqlgen@v1.0.0
go: added example.com/logx v1.3.0
go: added example.com/sqlgen v1.0.0
$ go list -m example.com/logx          # граф приложения не тронут
example.com/logx v1.1.0
$ go tool -modfile=tools.mod sqlgen
sqlgen: generating queries, args [] logx v1.3.0
```

Рядом появляются `tools.mod` и `tools.sum`. Цена: команду нужно вызывать с `-modfile` (удобно спрятать в `Makefile` или `//go:generate go tool -modfile=tools.mod sqlgen`).

> **Совет.** Мелкие инструменты без тяжёлых зависимостей (`stringer`, `mockgen`) держите прямо в `go.mod`. Инструменты с большим графом зависимостей (линтеры, генераторы gRPC) — в отдельном `-modfile`. Проверка — `go get -tool` не должен печатать `upgraded` для зависимостей приложения.

### 12.3. go install pkg@version

С Go 1.16 `go install example.com/cmd@v1.2.3` ставит инструмент **вне** контекста текущего модуля: «ignoring the go.mod file in the current directory» (release notes 1.16). С Go 1.18 `go get` больше не собирает и не устанавливает — только меняет `go.mod`. Ограничение: в `go.mod` устанавливаемого модуля не должно быть `replace`/`exclude` (§2.3).

### Простыми словами

Инструменты разработки (генераторы кода, линтеры) теперь можно записать прямо в `go.mod` и запускать `go tool имя`. Но их зависимости попадают в общую «корзину» с зависимостями программы и могут незаметно обновить что-то в самой программе — поэтому тяжёлые инструменты лучше держать в отдельном файле.

### Что важно для Senior

- `tool` + `go tool name` воспроизводимо фиксируют версии инструментов без `tools.go`.
- Граф модулей один на главный модуль: требования инструментов влияют на версии в бинарнике сервиса.
- Изоляция — `-modfile=tools.mod` для `go get -tool` и `go tool`.
- `go install cmd@version` — для глобальной установки, вне графа проекта.

---

## 13. Информация о сборке в бинарнике

### 13.1. Что вшивает go build

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	bi, ok := debug.ReadBuildInfo()
	if !ok {
		fmt.Println("no build info")
		return
	}
	fmt.Println("main:", bi.Main.Path, bi.Main.Version, "go:", bi.GoVersion)
	for _, s := range bi.Settings {
		switch s.Key {
		case "vcs.revision", "vcs.modified", "vcs.time", "-trimpath", "GOFLAGS":
			fmt.Printf("  %s=%s\n", s.Key, s.Value)
		}
	}
}
```

В git-репозитории с тегом `v1.4.2`:

```text
$ go build -o billing . && ./billing
main: example.com/billing v1.4.2 go: go1.27.0
  vcs.revision=7887141a59d2a93e9a686d387a261186df6e8291
  vcs.time=2026-10-22T20:17:27Z
  vcs.modified=false
$ go version -m billing
billing: go1.27.0
	path	example.com/billing
	mod	example.com/billing	v1.4.2	
	build	-buildmode=exe
	build	-compiler=gc
	build	CGO_ENABLED=1
	...
	build	GOARCH=arm64
	build	GOOS=darwin
	build	GOARM64=v8.0
	build	vcs=git
	build	vcs.revision=7887141a59d2a93e9a686d387a261186df6e8291
	build	vcs.time=2026-10-22T20:17:27Z
	build	vcs.modified=false
```

| Что | С какой версии |
|---|---|
| `debug.ReadBuildInfo`, список `dep` с версиями и h1-хешами | Go 1.12 (**⚠️ не проверено** по release notes, указано по памяти) |
| `vcs.revision`, `vcs.time`, `vcs.modified`, флаг `-buildvcs` | Go 1.18 (release notes) |
| Версия главного модуля из git-тега, суффикс `+dirty` | Go 1.24 (release notes: «A +dirty suffix will be appended if there are uncommitted changes») |
| `go version -m -json` | Go 1.25 |

До 1.24 `bi.Main.Version` был `(devel)`, и версию вшивали через `-ldflags "-X main.version=..."`. Теперь это часто не нужно.

> **Устаревший совет:** «Всегда прокидывайте версию через `-ldflags -X`». С Go 1.24 Go берёт версию из git-тега сам. `-ldflags -X` остаётся нужным, если сборка идёт без `.git` (например, Docker-контекст без папки `.git`, §18.1) или версия не совпадает с тегом.

### 13.2. Prometheus: go_build_info

Коллектор `collectors.NewBuildInfoCollector()` из `github.com/prometheus/client_golang` v1.25.0 превращает build info в метрику:

```text
go_build_info{checksum="",path="example.com/billing",version="v1.3.0"} 1
```

В Grafana это даёт «какая версия какого сервиса сейчас работает» без отдельной инфраструктуры; PromQL, например: `count by (version) (go_build_info{path="example.com/billing"})` — сколько подов на каждой версии во время раскатки. **⚠️ не проверено:** запрос не выполнялся на живом Prometheus.

### 13.3. Грабли со вшиванием версии

**Грабли: тег v2.3.0 у модуля без /v2 — версия превращается в псевдоверсию**

```text
$ git tag v2.3.0 && go build -o billing . && ./billing
go_build_info{checksum="",path="example.com/billing",version="v0.0.0-20261022202009-6132b21c06c9"} 1
```

Модуль `example.com/billing` (без `/v2`) не может иметь версию v2.3.0 (§5.2), поэтому Go проигнорировал тег и вычислил псевдоверсию. Как исправить: тег, совместимый с путём модуля (`v1.3.0`), или переход на `/v2`.

**Грабли: +dirty из-за собранного бинарника**

```text
$ git tag v1.3.0 && go build -o billing . && ./billing   # в папке лежит billing от прошлой сборки
go_build_info{checksum="",path="example.com/billing",version="v1.3.0+dirty"} 1
$ rm billing && git status --short && go build -o ../billing-bin . && ../billing-bin
go_build_info{checksum="",path="example.com/billing",version="v1.3.0"} 1
```

Коммит тот же, тег тот же; разница только в неотслеживаемом файле `billing`, который остался от предыдущей сборки.

Неотслеживаемые файлы тоже делают дерево «грязным». В CI это сгенерированный код, артефакты прошлых шагов, `.env`. Как исправить: `.gitignore` для артефактов, сборка в отдельную папку.

**Грабли: git есть, но падает (Docker)**

Если в папке есть `.git`, а git **отсутствует**, при `-buildvcs=auto` (по умолчанию) VCS-информация молча пропускается — это прямо в коде `src/cmd/go/internal/load/pkg.go`:

```go
if _, err := pathcache.LookPath(vcsCmd.Cmd); err != nil {
	// We found a repository, but the required VCS tool is not present.
	// "-buildvcs=auto" means that we should silently drop the VCS metadata.
	goto omitVCS
}
```

```text
$ PATH=<только go> go build -o billing . && ./billing | head -1
main: example.com/billing (devel) go: go1.27.0
```

А если git есть, но завершается с ошибкой — сборка падает. Типичный случай в Docker — `fatal: detected dubious ownership`, когда `.git` принадлежит другому пользователю. Я эмулировал это подставным `git`, который печатает такую ошибку и выходит с кодом 128:

```text
$ go build -o billing .
error obtaining VCS status: exit status 128
	Use -buildvcs=false to disable VCS stamping.
```

Как исправить: `git config --global --add safe.directory /src` в образе, либо `-buildvcs=false` плюс `-ldflags -X`, либо не копировать `.git` в контекст сборки. **⚠️ не проверено** в настоящем Docker: демон Docker на машине не был запущен; проверена только реакция `go` на код выхода git.

### Простыми словами

Go сам записывает в программу, из какого коммита и какой версии она собрана, и с какими версиями зависимостей. Это можно прочитать командой `go version -m` или из самой программы и, например, показать в метриках.

### Что важно для Senior

- `go version -m` на production-бинарнике отвечает на вопросы «какая версия зависимости внутри» и «из какого коммита» без доступа к CI.
- С Go 1.24 версия берётся из тега; `+dirty` — любые изменения, включая неотслеживаемые файлы.
- Тег должен соответствовать major-суффиксу пути, иначе вместо версии будет псевдоверсия.
- В Docker: нет git → тихо без VCS-данных; git есть, но падает → ошибка сборки.

---

## 14. Безопасность зависимостей: govulncheck

### 14.1. Пример

Сервис использует старый `golang.org/x/text v0.3.7` и вызывает `language.ParseAcceptLanguage` с заголовком запроса:

```go
func handler(w http.ResponseWriter, r *http.Request) {
	tags, _, _ := language.ParseAcceptLanguage(r.Header.Get("Accept-Language"))
	fmt.Fprintln(w, tags)
}
```

```text
$ go run golang.org/x/vuln/cmd/govulncheck@latest ./...     # govulncheck v1.9.0
=== Symbol Results ===

Vulnerability #1: GO-2026-6617
    HTTP/2 server crash due to HPACK encoder race in net/http
  More info: https://pkg.go.dev/vuln/GO-2026-6617
  Standard library
    Found in: net/http/internal/http2@go1.27
    Fixed in: net/http/internal/http2@go1.27.2
...
Vulnerability #6: GO-2022-1059
    Denial of service via crafted Accept-Language header in
    golang.org/x/text/language
  More info: https://pkg.go.dev/vuln/GO-2022-1059
  Module: golang.org/x/text
    Found in: golang.org/x/text@v0.3.7
    Fixed in: golang.org/x/text@v0.3.8
    Example traces found:
      #1: main.go:11:44: i18n.handler calls language.ParseAcceptLanguage

Your code is affected by 6 vulnerabilities from 1 module and the Go standard library.
This scan also found 7 vulnerabilities in packages you import and 4
vulnerabilities in modules you require, but your code doesn't appear to call
these vulnerabilities.
Use '-show verbose' for more details.
exit status 3
```

Что видно:
- найдены и уязвимости **стандартной библиотеки** самого go1.27.0 (исправлены в go1.27.2) — `govulncheck` проверяет и тулчейн;
- для каждой уязвимости — **конкретная цепочка вызовов** из вашего кода (`main.go:11:44`);
- отдельно посчитаны уязвимости, которые есть в зависимостях, но **не вызываются** — они не валят проверку;
- код выхода 3 при найденных уязвимостях — удобно для CI.

**Почему так спроектировано.** Сканеры уровня «версия модуля в списке» (Dependabot-подобные) дают много шума: модуль большой, а уязвима одна функция. `govulncheck` строит граф вызовов и сообщает только достижимое. Цена — нужен полный исходный код и сборка; для бинарника есть режим `-mode binary`, но он менее точен. **⚠️ не проверено:** режим `-mode binary` не запускал.

> **Совет.** Запускайте `govulncheck ./...` в CI на каждом PR и по расписанию (уязвимости появляются в базе позже, чем ваш код). Эффект: в примере выше 6 реально достижимых уязвимостей из 17 найденных — остальные не блокируют сборку. Не заменяет обновление тулчейна: часть находок исправляется только новым патчем Go.

### Простыми словами

`govulncheck` сверяет ваши зависимости и версию Go с базой известных уязвимостей и показывает только те, до которых ваш код реально «дотягивается» вызовами.

### Что важно для Senior

- Проверяются и модули, и стандартная библиотека тулчейна.
- Отчёт — по достижимым вызовам; недостижимые уязвимости показываются отдельно и не валят проверку.
- Код выхода 3 — сигнал для CI.
- Обновление патч-версии Go — тоже «обновление зависимостей».

---

## 15. Команды go get, go mod tidy и компания

### 15.1. Шпаргалка команд

| Команда | Что делает | Меняет go.mod |
|---|---|---|
| `go get mod@v1.2.3` | Добавить/поднять/понизить требование (MVS, §3.4) | да |
| `go get -u ./...` | Поднять прямые и косвенные зависимости пакетов до последних minor/patch | да |
| `go get -u=patch ./...` | Только патч-версии | да |
| `go get mod@none` | Удалить требование (и зависящие от него) | да |
| `go get go@1.27.0` / `toolchain@go1.27.2` | Поднять версию Go / тулчейна | да |
| `go mod tidy` | Добавить недостающие и удалить лишние требования, обновить `go.sum` | да |
| `go mod tidy -diff` (1.23) | Показать diff, не меняя файлы; ненулевой код, если нужны изменения | нет |
| `go mod download [-json] [-x]` | Скачать в кэш | нет |
| `go mod verify` | Сверить кэш с хешами (§8.2) | нет |
| `go mod why -m mod` | Кратчайшая цепочка импортов до модуля | нет |
| `go mod graph` | Все рёбра графа требований | нет |
| `go mod edit -require/-replace/-exclude/-go` | Правка `go.mod` без загрузок | да |
| `go list -m all` | Build list | нет |
| `go list -m -u all` | Доступные обновления, retract, deprecated | нет |
| `go list -m -versions mod` | Список версий (без отозванных; `-retracted` — с ними) | нет |
| `go list -m -json mod@q` | Подробности: версия, время, `Origin`, `GoVersion`, с 1.23 — `Sum` и `GoModSum` | нет |
| `go version -m bin` | Build info бинарника (§13) | нет |

Пример `go mod why`:

```text
$ go mod why -m example.com/logx
# example.com/logx
example.com/orders
example.com/kafka
example.com/logx
```

### 15.2. Что делает go mod tidy

`tidy` загружает пакеты шаблона `all` **со всеми build-тегами и для всех платформ** (кроме тега `ignore`), а также тесты, и после этого:
- добавляет требования для недостающих импортов (с последней версией модуля);
- удаляет требования, которые не нужны ни одному пакету;
- переносит косвенные в блок `// indirect`;
- дописывает и чистит `go.sum` (с go 1.21 — сохраняя хеши для тестов зависимостей, `TidyGoModSumVersion`).

### 15.3. Миф о build-тегах

> **Заблуждение:** «`go mod tidy` на macOS удалит зависимость, которая импортируется только в `_linux.go`». Проверка:

```text
$ cat kafka_linux.go
//go:build linux

package main

import _ "example.com/kafka"
$ cat gen.go
//go:build ignore

package main

import _ "example.com/pg"
$ cat tools.go
//go:build tools

package main

import _ "example.com/e122"
$ go mod tidy
go: finding module for package example.com/kafka
go: finding module for package example.com/e122
go: found example.com/e122 in example.com/e122 v1.0.0
go: found example.com/kafka in example.com/kafka v1.0.0
$ cat go.mod
module example.com/tags

go 1.27.0

require (
	example.com/e122 v1.0.0
	example.com/kafka v1.0.0
)

require example.com/logx v1.1.0 // indirect
```

`kafka` (только linux) и `e122` (тег `tools`) остались; удалён только `pg` из файла с `//go:build ignore`. Отсюда старый трюк `tools.go` с тегом `tools`: он работал именно потому, что `tidy` учитывает все теги, кроме `ignore`. Реальные случаи, когда `tidy` удаляет нужное: зависимость используется только через `go run pkg@version` в `//go:generate` или в скриптах вне Go-кода — для них есть директива `tool` (§12).

Заодно видно MVS: `logx` выбрана v1.1.0 (столько требует kafka), хотя есть v1.3.0. `tidy` добавляет **последнюю** версию только для модулей, которых не было в графе; косвенные берёт по требованиям.

### 15.4. Go 1.27: tidy склеивает блоки require

После ручных правок и слияний в `go.mod` копятся блоки `require`. Один и тот же файл, разные строки `go`:

```text
### go 1.26.0 после go mod tidy

require example.com/kafka v1.0.0

require example.com/logx v1.2.0 // indirect

require example.com/pg v1.1.0

### go 1.27.0 после go mod tidy

require (
	example.com/kafka v1.0.0
	example.com/pg v1.1.0
)

require example.com/logx v1.2.0 // indirect
```

Порог — `SimplifyRequireVersion = "1.27"` в `gover/version.go`; release notes 1.27: «For modules specifying go 1.27 or later in their go.mod file, go mod tidy now automatically merges duplicate require blocks… at most two require blocks». Поведение включается строкой `go` модуля, а не версией тулчейна.

> **Совет.** В CI добавьте `go mod tidy -diff` (Go 1.23+): шаг упадёт, если `go.mod`/`go.sum` не «прибраны». Это ловит забытый `tidy` после `go get` и ручные правки. Не нужно, если `go.mod` правит только бот обновлений, который сам делает `tidy`.

### Простыми словами

`go get` — «хочу такую версию», `go mod tidy` — «наведи порядок», `go list -m` — «покажи, что выбрано», `go mod why` — «зачем это здесь». `tidy` смотрит на код сразу для всех систем, поэтому зависимость из файла «только для Linux» не потеряется.

### Что важно для Senior

- `tidy` учитывает все build-теги и платформы, кроме `ignore`; трюк `tools.go` держался на этом.
- `go mod tidy -diff` — дешёвая проверка «прибранности» в CI.
- `go get -u` поднимает и косвенные зависимости; для минимального риска — `-u=patch` или точечный `go get mod@version`.
- С go 1.27 `tidy` поддерживает ровно два блока `require`.

---
## 16. Организация кода: internal, монорепозиторий, мультимодуль

### 16.1. internal/

Пакет внутри папки `internal` можно импортировать только из кода, который лежит в **родителе** этой папки `internal` (и ниже). `github.com/acme/orders/internal/storage` доступен из `github.com/acme/orders/...` и недоступен из других модулей. Правило проверяет компилятор при импорте, модули тут ни при чём, но для модулей это главный способ сказать «это не публичный API, совместимость не обещаю».

### 16.2. Один модуль на репозиторий (по умолчанию)

Для сервиса и для большинства библиотек: один `go.mod` в корне. Все пакеты получают одну версию, один `go.sum`, один граф.

### 16.3. Несколько модулей в одном репозитории

```text
github.com/acme/platform
├── go.mod                       module github.com/acme/platform
├── logx/go.mod                  module github.com/acme/platform/logx     тег: logx/v1.3.0
└── kafka/go.mod                 module github.com/acme/platform/kafka    тег: kafka/v0.9.0
```

Тег модуля в подпапке имеет префикс папки: `logx/v1.3.0`. Вложенный `go.mod` «отрезает» папку от родительского модуля.

| | Один модуль | Несколько модулей |
|---|---|---|
| Выпуск | одна версия на всё | независимые версии частей |
| Зависимости | общий граф | свой граф у каждой части: тяжёлые зависимости одной части не попадают к пользователям другой |
| Локальная разработка | просто | нужен `go.work` или `replace` (§10) |
| CI | `go test ./...` | `./...` не переходит границы модулей — тестировать каждый модуль отдельно или через workspace и шаблон `work` (Go 1.25) |
| Ошибки | редкие | рассинхрон версий между модулями, циклы требований |

> **Совет.** Делите репозиторий на модули только если части **выпускаются и потребляются независимо** (например, лёгкий клиент-SDK и тяжёлый сервер с базой данных). Эффект — пользователи SDK не тянут драйвер PostgreSQL. Не нужно для сервиса, который никто не импортирует: там несколько модулей только добавят `go.work` и сложный CI.

**Грабли: `go test ./...` пропускает вложенные модули**

Корневой модуль `example.com/platform` и вложенный `logx/` со своим `go.mod`; во вложенном — падающий тест:

```text
$ go test ./...
ok  	example.com/platform	0.773s
$ (cd logx && go test ./...)
--- FAIL: TestFails (0.00s)
    l_test.go:5: тест во вложенном модуле упал
FAIL
FAIL	example.com/platform/logx	0.530s
FAIL
$ go work init . ./logx && go test work
ok  	example.com/platform	(cached)
--- FAIL: TestFails (0.00s)
    l_test.go:5: тест во вложенном модуле упал
FAIL
FAIL	example.com/platform/logx	0.300s
FAIL
```

Почему так: `./...` в модульном режиме означает «пакеты **текущего** модуля», а папка с `go.mod` — уже другой модуль. Как заметить: число пакетов в отчёте CI меньше ожидаемого; ошибок нет. Как исправить: цикл по модулям в CI (`for d in $(find . -name go.mod -exec dirname {} \;); do (cd $d && go test ./...); done`) или workspace и шаблон `work` (Go 1.25).

### Простыми словами

Обычно один репозиторий = один модуль. Делить на несколько стоит, только когда части живут своей жизнью и выпускаются отдельно. Папка `internal` — «служебный вход»: снаружи модуля туда не попасть.

### Что важно для Senior

- `internal/` — граница публичного API модуля, проверяемая компилятором.
- Мультимодульный репозиторий оправдан изоляцией графов зависимостей и независимыми релизами; цена — `go.work`, теги с префиксами и CI по модулям.
- Шаблон `./...` не пересекает границы модулей.

---

## 17. Настройки и параметры

Версии `GOINSECURE`, `-modcacherw` (go1.14) и `GOMODCACHE` (go1.15) определены по `git log -S` + `git tag --contains`.

| Параметр | Значения | С версии | Эффект |
|---|---|---|---|
| `GO111MODULE` | `on` (по умолчанию), `auto`, `off` | 1.11; `on` по умолчанию с 1.16 | Включение модульного режима |
| `GOPROXY` | список URL, `direct`, `off`; разделители `,` и `\|` | 1.13 (по умолчанию `proxy.golang.org,direct`) | Откуда качать модули (§7.2) |
| `GOSUMDB` | `sum.golang.org`, `off`, своё `имя+ключ+URL` | 1.13 | Checksum database (§6.2) |
| `GOPRIVATE` | glob-список путей | 1.13 | Значение по умолчанию для `GONOPROXY` и `GONOSUMDB` (§7.3) |
| `GONOPROXY` / `GONOSUMDB` | glob-список, `none` | 1.13 | Точечно: не через прокси / не в sumdb |
| `GOINSECURE` | glob-список | 1.14 | Разрешить HTTP без TLS |
| `GOVCS` | `public:git\|hg,private:all` (по умолчанию) | 1.16 | Какие VCS можно запускать |
| `GOAUTH` | `netrc` (по умолчанию), `git dir`, команда, `off` | 1.24 | Аутентификация для приватных загрузок (§7.4) |
| `GOMODCACHE` | путь | 1.15 | Кэш модулей (§8) |
| `GOFLAGS` | флаги через пробел | 1.11 | Флаги по умолчанию: `-mod=mod`, `-modcacherw`, `-buildvcs=false` |
| `GOWORK` | путь, `off` | 1.18 | Файл workspace (§10) |
| `GOTOOLCHAIN` | `auto` (по умолчанию, `go.env`), `local`, `path`, `go1.X.Y[+auto\|+path]` | 1.21 | Выбор тулчейна (§11.2) |
| `-mod` | `readonly` (по умолчанию), `vendor` (если есть `vendor/` и go ≥ 1.14), `mod` | 1.14/1.16 | Можно ли менять `go.mod` и откуда брать код |
| `-modcacherw` | флаг | 1.14 | Кэш с правами на запись |
| `-buildvcs` | `auto` (по умолчанию), `true`, `false` | 1.18 | Вшивание VCS-данных (§13) |
| `GODEBUG=toolchaintrace=1` | | 1.24 | Трассировка выбора тулчейна |

| Сценарий | Настройки | Обоснование |
|---|---|---|
| Разработчик в компании | `go env -w GOPRIVATE=*.corp.example.com` | Приватные пути не уходят в публичные сервисы |
| CI с корпоративным прокси | `GOPROXY=https://goproxy.corp.example.com`, `GONOSUMDB=*.corp.example.com`, `GOFLAGS=-mod=readonly`, `GOTOOLCHAIN=local` | Один источник, сборка не меняет `go.mod`, версия Go — из образа |
| Изолированный контур без интернета | `vendor/` или `GOPROXY=file:///mirror`, `GOSUMDB=off` + закоммиченный `go.sum`, `GOTOOLCHAIN=local` | Нет сетевых обращений; целостность — только по `go.sum` |
| Открытая библиотека | `go` в `go.mod` = самая старая поддерживаемая версия (обычно N-1), без `toolchain`, без `replace` | Не навязывать пользователям новый тулчейн |

### Простыми словами

Почти всё поведение `go` с модулями настраивается переменными окружения. Главные: откуда качать (`GOPROXY`), что считать приватным (`GOPRIVATE`) и какую версию Go использовать (`GOTOOLCHAIN`).

### Что важно для Senior

- Значения по умолчанию для `GOPROXY`, `GOSUMDB`, `GOTOOLCHAIN` лежат в `$GOROOT/go.env`; `go env -w` пишет в пользовательский файл, переменная окружения перекрывает всё.
- `go env -changed` (1.23) быстро показывает, что на машине настроено не по умолчанию.
- Для CI задавайте переменные явно, а не полагайтесь на `go env -w` в образе.

---

## 18. Production: Docker, CI, Kubernetes

### 18.1. Docker-сборка с кэшем модулей

**⚠️ не проверено запуском:** Docker-демон на машине не был запущен, Dockerfile ниже собран по документации BuildKit и правилам `go`, которые проверены в предыдущих разделах.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.27.2 AS build
WORKDIR /src
ENV GOTOOLCHAIN=local \
    GOFLAGS=-mod=readonly \
    GOPRIVATE=gitlab.corp.example.com

# 1. Сначала только go.mod/go.sum: слой с зависимостями кэшируется,
#    пока они не меняются.
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=secret,id=netrc,target=/root/.netrc \
    go mod download && go mod verify

# 2. Потом код.
COPY . .
ARG VERSION=dev
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -buildvcs=false \
      -ldflags="-s -w -X main.version=${VERSION}" \
      -o /out/orders ./cmd/orders

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/orders /orders
ENTRYPOINT ["/orders"]
```

Пояснения:
- `GOTOOLCHAIN=local` — версия Go строго из образа; если `go.mod` требует новее, сборка упадёт сразу (§11.2), а не скачает тулчейн из интернета;
- `COPY go.mod go.sum` отдельным слоем + `go mod download` — зависимости не перекачиваются при каждом изменении кода;
- `--mount=type=secret` для `.netrc` — токен приватного GitLab не остаётся в слоях образа;
- `go mod verify` — проверка, что кэш из `--mount=type=cache` не испорчен (§8.2);
- `-buildvcs=false` + `-ldflags -X` — `.git` обычно не попадает в контекст (`.dockerignore`), а если попадает, git в образе может упасть на «dubious ownership» (§13.3).

### 18.2. CI: минимальный набор проверок

```bash
go mod tidy -diff                 # go.mod/go.sum прибраны (Go 1.23+)
go mod verify                     # кэш не испорчен
test -z "$(grep -nE '=> *\.{1,2}/' go.mod)"   # нет replace на локальные папки
GOWORK=off go build ./...         # собирается без go.work
go run golang.org/x/vuln/cmd/govulncheck@v1.9.0 ./...   # код выхода 3 = есть уязвимости
go list -m -u all | grep -E 'retracted|deprecated' && exit 1 || true
```

Версия `govulncheck` зафиксирована (`@v1.9.0`, а не `@latest`), чтобы CI был воспроизводим.

### 18.3. Корпоративный прокси

Свой прокси (Athens, Artifactory, Nexus, GitLab package registry) решает три задачи: кэш (скорость, независимость от GitHub), хранение версий (модуль не исчезнет, даже если автор удалит репозиторий) и единая точка для приватных модулей. Протокол — тот же, что в §7.1. **⚠️ не проверено:** конкретные продукты не разворачивались.

### 18.4. Kubernetes

Модули влияют на Kubernetes косвенно:
- **воспроизводимость образа**: одинаковый `go.mod`/`go.sum` + `GOTOOLCHAIN=local` + фиксированный базовый образ → одинаковый бинарник;
- **версия в метриках**: `go_build_info` (§13.2) показывает, сколько подов на какой версии во время раскатки; `go version -m` на бинарнике из образа отвечает, какая версия зависимости реально работает в кластере;
- **GODEBUG** из `go.mod` вшит в бинарник; переопределить на время инцидента можно переменной окружения `GODEBUG` в манифесте пода, без пересборки.

```yaml
# фрагмент Deployment: временно вернуть старое поведение HTTP/2-клиента
env:
  - name: GODEBUG
    value: "http2client=0"
```

### Простыми словами

В Docker сначала скачиваем зависимости отдельным шагом (он кэшируется), потом собираем код. В CI проверяем, что `go.mod` в порядке, нет локальных подмен и известных уязвимостей. Версию Go в сборке фиксируем, чтобы она не «приехала» из интернета.

### Что важно для Senior

- `GOTOOLCHAIN=local` и `GOFLAGS=-mod=readonly` делают сборку герметичной и предсказуемой.
- Секреты для приватных модулей — через BuildKit secrets, не через `ARG`/`ENV`.
- `go mod tidy -diff`, `go mod verify`, `govulncheck` — три дешёвые проверки, которые ловят большинство проблем с модулями.
- GODEBUG можно переключить в поде без пересборки — полезно при регрессиях после обновления Go.

---

## 19. Наблюдаемость и диагностика

### 19.1. Симптом → инструмент

| Симптом | Команда | Что искать |
|---|---|---|
| «Откуда взялась эта версия?» | `go mod graph \| grep logx`, `go mod why -m mod` | Кто требует максимальную версию |
| Сборка тянет неожиданную версию | `go list -m all`, `go list -m -json mod` | Выбранная версия, `Replace`, `Origin` |
| Какие версии есть | `go list -m -versions [-retracted] mod` | Отозванные версии |
| Долгие загрузки, «висит» `go mod download` | `go mod download -x` | URL запросов, 404 от публичного прокси для приватных модулей (§7.3) |
| `SECURITY ERROR` / `checksum mismatch` | `go mod download -json mod@v` (поля `Sum`, `GoModSum`) | Подмена или перезаписанный тег (§6.3) |
| «Какую версию Go использует сборка?» | `GODEBUG=toolchaintrace=1 go version` в папке модуля | Переключение тулчейна (§11.2) |
| Странное поведение после обновления Go | `go list -f '{{.DefaultGODEBUG}}' .`, `go version -m bin \| grep GODEBUG` | Изменённые значения GODEBUG |
| Что в production-бинарнике | `go version -m bin`, метрика `go_build_info` | Версии зависимостей и коммит |
| `inconsistent vendoring` | `go mod vendor` | Рассинхрон `vendor/modules.txt` и `go.mod` (§9.2) |
| Подозрение на испорченный кэш | `go mod verify` | `dir has been modified` (§8.2) |
| Какие переменные отличаются от дефолта | `go env -changed` (1.23) | Забытые `go env -w` |

### 19.2. Алгоритм: «после go get сломалась сборка»

```mermaid
graph TD
  A["Сборка сломалась после go get / tidy"] --> B{"git diff go.mod: что изменилось?"}
  B -->|"upgraded X"| C["go mod graph | grep X — кто поднял"]
  C --> D{"Поднял инструмент (tool)?"}
  D -->|да| E["Вынести инструмент в -modfile (§12.2)"]
  D -->|нет| F["Откатить: go get X@старая; проверить 'removed' (§3.4)"]
  B -->|"removed Y"| G["Был downgrade общей зависимости (§3.4)"]
  B -->|"go 1.N поднялась"| H["Зависимость требует новее Go (§4.3, §11)"]
```

Шаги:
1. `git diff go.mod go.sum` — какие версии поменялись.
2. `go mod graph | grep <модуль>` — кто требует максимальную версию.
3. `go mod why -m <модуль>` — нужен ли модуль вообще.
4. Если виноват инструмент — `-modfile`; если новая версия сломана — `go get mod@prev` и `exclude` в главном модуле, плюс issue автору.
5. `go mod tidy -diff` — убедиться, что итог «прибран».

### Простыми словами

Почти на любой вопрос «почему эта версия?» отвечают две команды: `go mod graph` (кто чего требует) и `go mod why` (зачем модуль нужен). А флаг `-x` показывает, куда `go` ходит по сети.

### Что важно для Senior

- `go mod graph` + `go mod why` отвечают на 90% вопросов «откуда версия».
- `-x` показывает реальные HTTP-запросы `go`-команды — лучший способ отладки прокси и приватных модулей.
- `go version -m` и `go_build_info` — наблюдаемость зависимостей в production.

---

## 20. Грабли, заблуждения и советы: сводка

### Грабли

| Ситуация | Что происходит | Как заметить | Как исправить |
|---|---|---|---|
| `replace` в библиотеке (§2.2) | у потребителей не действует | только `go list -m all` | выпустить исправленную зависимость, поднять `require` |
| `replace` на локальную папку в git (§2.3) | `replacement directory ... does not exist` у других | CI | `go.work`, проверка в CI |
| `replace` в CLI-модуле (§2.3) | `go install cmd@v` отказывается | ошибка установки | убрать `replace` |
| `go get dep@старая` (§3.4) | зависимые модули удаляются (`removed`) | вывод `go get`, ошибка сборки | не понижать общую зависимость |
| `exclude` (§3.5) | версия ниже требуемой зависимостью | `go list -m all` | явно поднять версию выше исключённой |
| забыт `/v2` (§5.2) | `should be v0 or v1, not v2` | ошибка у потребителей | `module .../v2` |
| удалить строку `go.sum` при `SECURITY ERROR` (§6.3) | для приватных модулей подмена принимается | — | расследовать причину |
| нет `GOPRIVATE` (§7.3) | путь уходит в `proxy.golang.org`, 404-задержки | `go mod download -x` | `GOPRIVATE` |
| правка в кэше модулей (§8.2) | сборка не замечает | `go mod verify` | `go.work` / клон |
| ручная правка `go.mod` с `vendor/` (§9.2) | `inconsistent vendoring` | сборка падает | `go mod vendor` |
| закоммиченный `go.work` (§10.2) | CI собирает не то | `git ls-files` | `.gitignore`, `GOWORK=off` |
| `GOTOOLCHAIN=go1.21` (§11.2) | `language version but not a toolchain version` | ошибка | `go1.21.0` |
| удалённый GODEBUG в старом значении (§11.5) | ошибка загрузки `go.mod` в Go 1.27 | ошибка | финальное значение или убрать |
| `tool` с тяжёлым графом (§12.2) | поднимает версии в приложении | `upgraded` в `go get -tool` | `-modfile=tools.mod` |
| тег v2+ без `/v2` у главного модуля (§13.3) | в бинарнике псевдоверсия | `go version -m` | правильный тег |
| неотслеживаемые файлы (§13.3) | `+dirty` | `go version -m` | `.gitignore` |
| git падает в Docker (§13.3) | `error obtaining VCS status` | сборка падает | `safe.directory` / `-buildvcs=false` |

### Заблуждения

| Думают | На самом деле | До какой версии было верно |
|---|---|---|
| `go.sum` — это lock-файл | Версии задаёт `go.mod` + MVS; `go.sum` — хеши (§6) | никогда |
| Go берёт последнюю версию зависимости | Берёт максимальную из **требуемых** (§3) | никогда (в GOPATH — последний коммит) |
| `// indirect` можно удалить | Нужны для pruning (§4.3) | до Go 1.17 почти верно |
| `tidy` удаляет зависимости из `_linux.go` | Учитывает все теги, кроме `ignore` (§15.3) | никогда |
| Версия Go в CI = версия в образе | При `GOTOOLCHAIN=auto` определяет `go.mod` (§11.2) | до Go 1.21 |
| Версию в бинарник надо вшивать `-ldflags -X` | Go берёт её из тега (§13.1) | до Go 1.24 |
| `go mod init` пишет `go 1.(N-1).0` | Только go1.26.0; откатили в go1.26.1 (§11.1) | только в go1.26.0 |
| Есть `GONOSUMCHECK` | Есть `GONOSUMDB` (§7.3) | никогда |
| Отозванная версия ломает сборку | Только предупреждение (§5.5) | никогда |

### Советы

| Действие | Когда | Эффект (измерено) | Когда не нужно |
|---|---|---|---|
| `go ≥ 1.17` в `go.mod` библиотек (§4.3) | любая поддерживаемая библиотека | 2 строки `go.sum` вместо 4 в эксперименте; меньше загрузок `go.mod` | библиотека для Go < 1.17 |
| CI: `go list -m -u all \| grep retracted` (§5.5) | долгоживущие сервисы | ловит отозванные версии | бот обновлений показывает retract |
| `go.sum merge=union` (§6.5) | частые параллельные PR с зависимостями | меньше конфликтов (⚠️ не измерено) | — |
| `go mod verify` после восстановления кэша (§8.2) | CI с кэшем | ловит испорченный кэш | кэш каждый раз с нуля |
| vendor (§9.3) | изолированные контуры | сборка с `GOPROXY=off` и пустым кэшем | есть надёжный корпоративный прокси |
| `GOTOOLCHAIN=local` (§11.2) | герметичные сборки | падение с понятной ошибкой вместо скачивания | `go.mod` должен управлять версией Go |
| `-modfile=tools.mod` (§12.2) | тяжёлые инструменты | граф приложения не меняется | мелкие инструменты |
| `govulncheck` в CI (§14) | всегда | 6 достижимых из 17 найденных в примере | — |
| `go mod tidy -diff` в CI (§15.4) | всегда | ловит неприбранный `go.mod` | `go.mod` правит только бот |
| несколько модулей в репозитории (§16.3) | независимые релизы | изоляция графов | сервис, который никто не импортирует |

---

## 21. Сравнение с другими платформами

**⚠️ не проверено запуском:** таблица — по общеизвестному устройству этих менеджеров, без экспериментов в этой сессии.

| | Go modules | npm | Cargo (Rust) | Maven (Java) |
|---|---|---|---|---|
| Выбор версий | MVS: максимум из минимумов | диапазоны + дерево дублей | диапазоны, SAT-подобный солвер | «ближайший побеждает» (nearest wins) |
| Lock-файл | не нужен (`go.sum` — только хеши) | `package-lock.json` | `Cargo.lock` | нет (версии фиксируют в POM/BOM) |
| Несколько версий одной библиотеки | только разные major (`/v2`) | любые, вложенные `node_modules` | разные semver-несовместимые | одна на classpath |
| Центральный реестр | нет: путь = URL; прокси — кэш | npmjs.com | crates.io | Maven Central |
| Глобальная проверка целостности | `sum.golang.org` (transparency log) | `integrity` в lock-файле | хеши в `Cargo.lock` | подписи/хеши артефактов |
| Удаление опубликованной версии | невозможно; `retract` | `unpublish` с ограничениями | `yank` | невозможно в Central |

Ключевые мысли для разговора с коллегами:
- «Где lock-файл?» — его роль выполняет сам `go.mod`: MVS не зависит от того, что опубликовано сегодня.
- «Почему у нас две версии библиотеки?» — в Go это возможно только для разных major-версий, и это осознанно.
- `retract` ≈ `cargo yank`: версия остаётся доступной, но не выбирается автоматически.

---
## 22. Вопросы для собеседования

### 🟢 Middle

**1. Чем модуль отличается от пакета?**
Пакет — папка с `.go`-файлами, единица импорта. Модуль — набор пакетов с общим `go.mod` и общей версией, единица версионирования и загрузки. Путь импорта пакета = путь модуля + подпапка (§1.1).

**2. Что лежит в go.mod и go.sum?**
`go.mod` — путь модуля, версия Go, требования (нижние границы версий), `replace`/`exclude`/`retract`/`tool`/`godebug`. `go.sum` — h1-хеши zip-архива и `go.mod` каждой нужной версии (§2, §6.1). Коммитить надо оба.

**3. ⚠️ go.sum — это lock-файл?**
Нет. Версии определяет `go.mod` + MVS; `go.sum` только проверяет, что код этих версий не подменён. Удаление `go.sum` не меняет выбранные версии (§6).

**4. Как Go выбирает версию зависимости, если её требуют два модуля?**
MVS берёт максимальную из требуемых: kafka требует v1.1.0, pg — v1.2.0 → v1.2.0, даже если существует v1.3.0 (§3.2).

**5. Что значит `// indirect`?**
Требование на модуль, пакеты которого ваш код не импортирует напрямую. С go 1.17 такие строки обязательны для обрезки графа (§4.3).

**6. Как обновить зависимости?**
`go get mod@version` — точечно; `go get -u ./...` — все до последних minor/patch; `-u=patch` — только патчи; `go list -m -u all` — посмотреть, что доступно (§15.1).

**7. Что делает `go mod tidy`?**
Добавляет недостающие требования, удаляет лишние, обновляет `go.sum`; учитывает все платформы и build-теги, кроме `ignore` (§15.2, §15.3).

**8. Зачем GOPRIVATE?**
Чтобы приватные модули не запрашивались у `proxy.golang.org` и не проверялись в `sum.golang.org`: иначе путь модуля утекает, а загрузка ждёт 404 (§7.3).

**9. Как поставить CLI-инструмент нужной версии?**
Глобально — `go install example.com/cmd@v1.2.3` (Go 1.16+). Для проекта — `go get -tool` и `go tool name` (Go 1.24+) (§12).

**10. Что такое vendor и когда он включается?**
Копия нужных пакетов зависимостей в `vendor/` + `modules.txt`. Включается сам, если папка есть и `go ≥ 1.14` в `go.mod` (§9.2).

### 🟡 Senior

**11. Почему Go не использует SAT-солвер, как npm или Cargo?**
MVS линеен, детерминирован и не зависит от новых публикаций, поэтому не нужен lock-файл. Цена — нет верхних границ версий: нельзя запретить сломанную версию для всех потребителей, есть только `retract` автора и `exclude` в главном модуле (§3.3).

**12. ⚠️ Вы добавили replace в свою библиотеку, чтобы обойти баг в зависимости. Помогло ли это пользователям?**
Нет. `replace` (и `exclude`, `toolchain`, `godebug`, `tool`) читаются только из главного модуля — это зашито в парсер `go.mod`. Пользователи получат исходную версию без предупреждения (§2.2). Нужно выпустить исправленную зависимость или поднять `require`.

**13. Как выпустить несовместимую версию библиотеки?**
Новый major с суффиксом пути: `module example.com/logx/v2`, импорты `.../v2/...`, тег `v2.0.0`. v1 и v2 — разные модули и могут жить в одной сборке. Без суффикса — `invalid version: should be v0 or v1, not v2` (§5.2).

**14. Что такое псевдоверсия и почему у неё такой формат?**
Версия коммита без тега: `vX.Y.(Z+1)-0.<UTC-время>-<12 символов хеша>`. Формат гарантирует порядок «базовый тег < псевдоверсия < следующий тег», поэтому MVS сравнивает её с тегами корректно. С Go 1.13 время и хеш проверяются (§5.3).

**15. Что такое +incompatible?**
Тег v2+ у репозитория без `go.mod`. Go считает его частью v1-модуля и может автоматически поднять до несовместимой версии (§5.4).

**16. Автор выпустил сломанную версию. Что он может сделать?**
Удалить нельзя (прокси и sumdb хранят навсегда). Выпустить новую версию с `retract` сломанной: `@latest` её пропустит, `go get` предупредит, но собранные с ней проекты продолжат собираться (§5.5).

**17. Что такое module graph pruning и как оно видно на практике?**
С go 1.17 Go берёт из `go.mod` зависимостей (на go ≥ 1.17) только их прямые требования, а `go.mod` транзитивных модулей не читает и не скачивает. Видно по `go mod graph` (обрезанные узлы) и по `go.sum` (нет строк `/go.mod` транзитивных модулей); цена — полный список косвенных требований в `go.mod` (§4.2).

**18. ⚠️ Вы понизили версию общей зависимости через go get, и пропал другой модуль. Почему?**
MVS-downgrade обязан убрать все требования выше целевой версии. Если у зависимого модуля нет версии, совместимой с понижением, он понижается до `none` — `go: removed example.com/pg v1.0.0` (§3.4).

**19. Как защищён код зависимостей от подмены?**
Хеши в `go.sum` + глобальный журнал `sum.golang.org`: первая загрузка версии фиксирует хеш для всех. Подмена даёт `SECURITY ERROR`. Для приватных модулей sumdb не используется — защита только закоммиченным `go.sum` (§6.2, §6.3).

**20. Как разрешить конфликт слияния в go.sum?**
Оставить обе стороны (удалить маркеры) в `go.mod` и `go.sum`, затем `go mod tidy`. Безопасно, потому что требования — нижние границы, а `go.sum` — множество хешей (§6.5).

**21. Чем `go 1.21` в go.mod отличается от `toolchain go1.21.0`, и что такое GOTOOLCHAIN?**
`go` — версия языка и жёсткий минимум (с 1.21), влияет на потребителей. `toolchain` — предпочтительный тулчейн, только для главного модуля. `GOTOOLCHAIN=auto` (по умолчанию) скачивает нужный тулчейн как модуль `golang.org/toolchain`; `local` — запрещает (§11).

**22. Какой риск у директивы tool?**
Граф модулей один: зависимости инструмента поднимают версии зависимостей приложения, и в бинарник попадают другие версии. Изоляция — `-modfile=tools.mod` (§12.2).

### 🔴 Staff

**23. ⚠️ Почему go build не замечает правку файлов в кэше модулей, и как это ловить?**
Сборка доверяет распакованному кэшу (иначе каждая сборка хешировала бы все зависимости). `go mod verify` пересчитывает хеши папок и zip и сообщает `dir has been modified`. В CI с переиспользуемым кэшем — обязательный шаг (§8.2).

**24. Как вы организуете зависимости в изолированном контуре без интернета?**
Варианты: `vendor/` в репозитории; файловый или корпоративный прокси (`GOPROXY=file://...`, формат = `cache/download`); `GOSUMDB=off` или своя sumdb + закоммиченный `go.sum`; `GOTOOLCHAIN=local`, чтобы не пытаться скачать тулчейн (§7.1, §8.1, §17).

**25. Как GODEBUG связан с модулями и что изменилось в Go 1.27?**
Значения GODEBUG по умолчанию выбираются по строке `go` главного модуля и директиве `godebug` (1.23), вшиваются в бинарник; переменная окружения перекрывает. В Go 1.27 удалённые настройки (например, `asynctimerchan`) принимаются в финальном значении и дают ошибку в старом (§11.5).

**26. Что и как вшивается в бинарник, и как это использовать в эксплуатации?**
Версии и хеши всех зависимостей, флаги сборки, VCS-данные (1.18), версия главного модуля из тега (1.24). Читается `go version -m` и `debug.ReadBuildInfo`; экспортируется метрикой `go_build_info`. Грабли: `+dirty` от неотслеживаемых файлов, псевдоверсия при теге v2+ без `/v2`, ошибка VCS при падающем git в Docker (§13).

**27. Когда делить репозиторий на несколько модулей?**
Когда части выпускаются независимо и у них разные тяжёлые зависимости (SDK без драйвера БД). Цена — теги с префиксом папки, `go.work` для разработки, CI по модулям (`./...` не пересекает границу модуля), риск рассинхрона (§16.3).

**28. Что проверяет govulncheck и почему он точнее сканера версий?**
Сверяет модули и стандартную библиотеку с базой уязвимостей и строит граф вызовов: сообщает только достижимые уязвимости, с цепочкой вызовов; недостижимые — отдельно. Код выхода 3 — для CI (§14).

**29. ⚠️ В release notes Go 1.26 написано, что go mod init пишет go 1.(N-1).0. Это так?**
Только в go1.26.0. Изменение откатили (issue 77653) в go1.26.1 и go1.27: пишется версия текущего тулчейна. Проверка — `go mod init` и `cat go.mod` (§11.1). Хороший пример, почему факты сверяют с кодом.

**30. Как бы вы отладили «go mod download висит 30 секунд в CI»?**
`go mod download -x` покажет HTTP-запросы: типично приватные модули без `GOPRIVATE` ждут 404 от `proxy.golang.org`, затем идут в git; или корпоративный прокси через `,` не даёт откатиться на `direct`. Дальше — `GOPRIVATE`/`GONOPROXY`, `|` в `GOPROXY`, кэш модулей между запусками (§7, §19.1).

---

## 23. Чеклист и шпаргалка

### Чеклист «здоровье модулей сервиса»

**Конфигурация**
- [ ] `go.mod` и `go.sum` закоммичены; `go.work`, `go.work.sum` — в `.gitignore` (кроме осознанного монорепозитория).
- [ ] Нет `replace` на локальные папки; любой `replace` имеет комментарий «зачем и когда убрать».
- [ ] `GOPRIVATE` настроен у разработчиков и в CI.
- [ ] Строка `go` — осознанная (для библиотек — самая старая поддерживаемая версия).
- [ ] Тяжёлые инструменты — в отдельном `-modfile`.

**CI**
- [ ] `go mod tidy -diff`, `go mod verify`, `GOWORK=off go build ./...`.
- [ ] `govulncheck ./...` с зафиксированной версией.
- [ ] `GOTOOLCHAIN=local`, `GOFLAGS=-mod=readonly` в герметичных сборках.
- [ ] Тесты всех модулей репозитория, если их несколько.

**Наблюдаемость**
- [ ] Метрика `go_build_info` или эквивалент; версия в логах при старте.
- [ ] Проверка, что версия в бинарнике без `+dirty` и не псевдоверсия.

### Переменные окружения

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `GOPROXY` | `https://proxy.golang.org,direct` (`go.env`) | Источники модулей |
| `GOSUMDB` | `sum.golang.org` (`go.env`) | Checksum database |
| `GOPRIVATE` | пусто | Приватные пути (по умолчанию для `GONOPROXY`, `GONOSUMDB`) |
| `GONOPROXY`, `GONOSUMDB` | = `GOPRIVATE` | Точечная настройка |
| `GOINSECURE` | пусто | HTTP без TLS |
| `GOVCS` | `public:git\|hg,private:all` | Разрешённые VCS |
| `GOAUTH` | `netrc` | Аутентификация (1.24) |
| `GOMODCACHE` | `$GOPATH/pkg/mod` | Кэш модулей |
| `GOFLAGS` | пусто | Флаги по умолчанию |
| `GOWORK` | поиск `go.work` вверх | Workspace |
| `GOTOOLCHAIN` | `auto` (`go.env`) | Выбор тулчейна |
| `GO111MODULE` | `on` | Модульный режим |

### GODEBUG, относящиеся к модулям

| Настройка | Назначение |
|---|---|
| `toolchaintrace=1` | Трассировка выбора тулчейна (1.24) |
| `allowmultiplevcs` | Разрешить несколько VCS в одном репозитории при вшивании VCS-данных (упомянут в комментарии `src/cmd/go/internal/load/pkg.go`) |

### Ключевые константы `src/cmd/go/internal/gover/version.go`

| Константа | Значение | Смысл |
|---|---|---|
| `NarrowAllVersion` | 1.16 | `all` больше не включает зависимости тестов пакетов вне главного модуля |
| `DefaultGoModVersion` | 1.16 | Версия для `go.mod` без строки `go` |
| `DefaultGoWorkVersion` | 1.18 | То же для `go.work` |
| `ExplicitIndirectVersion` | 1.17 | Обязательные косвенные требования, pruning |
| `SeparateIndirectVersion` | 1.17 | Отдельный блок `// indirect` |
| `TidyGoModSumVersion` | 1.21 | `tidy` хранит хеши для тестов зависимостей |
| `GoStrictVersion` | 1.21 | `go` главного модуля ≥ `go` зависимостей; «слишком новый» — ошибка |
| `ExplicitModulesTxtImportVersion` | 1.23 | Импорт из vendor — только перечисленного в `modules.txt` |
| `GoModToolVersion` | 1.24 | Минимум для директивы `tool` |
| `SimplifyRequireVersion` | 1.27 | `tidy` держит не больше двух блоков `require` |

Другие константы: `gotoolchainModule = "golang.org/toolchain"`, `gotoolchainVersion = "v0.0.1"` (`toolchain/select.go`); параллелизм обхода графа MVS — `work.Do(10, ...)` (`mvs/mvs.go`); ключ `sum.golang.org` — `modfetch/key.go`.

### Команды на каждый день

```text
go mod init example.com/svc        # новый модуль
go get example.com/lib@v1.2.3      # добавить/сменить версию
go get example.com/lib@none        # удалить
go get -u=patch ./...              # патчи
go mod tidy [-diff]                # прибраться / проверить
go list -m -u all                  # обновления, retract, deprecated
go mod why -m example.com/lib      # зачем модуль
go mod graph | grep lib            # кто требует
go mod download -x                 # отладка загрузок
go mod verify                      # целостность кэша
go mod vendor                      # vendor
go work init . ../lib              # workspace
go get -tool example.com/cmd       # инструмент
go version -m ./bin/svc            # что внутри бинарника
GODEBUG=toolchaintrace=1 go version
```

### Первоисточники

- Спецификация модулей: https://go.dev/ref/mod
- Тулчейны: https://go.dev/doc/toolchain
- GODEBUG: https://go.dev/doc/godebug (в репозитории — `doc/godebug.md`)
- Release notes: https://go.dev/doc/go1.11, go1.13, go1.14, go1.16, go1.17, go1.18, go1.21, go1.22, go1.23, go1.24, go1.25, go1.26, go1.27
- Russ Cox, «Minimal Version Selection»: https://research.swtch.com/vgo-mvs (ссылка из `src/cmd/go/internal/mvs/mvs.go`)
- Политика приватности sumdb: https://sum.golang.org/privacy
- База уязвимостей: https://pkg.go.dev/vuln/
- Исходники (go1.27.0): `src/cmd/go/internal/mvs/{mvs,graph}.go`, `modload/init.go`, `modfetch/{fetch,proxy,sumdb,key}.go`, `gover/version.go`, `toolchain/select.go`, `load/pkg.go`, `src/cmd/vendor/golang.org/x/mod/modfile/rule.go`, `.../sumdb/dirhash/hash.go`, `src/internal/godebugs/table.go`, `go.env`
- Issue об откате `go mod init`: https://go.dev/issue/77653
