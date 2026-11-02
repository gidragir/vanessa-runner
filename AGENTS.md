# vanessa-runner — инструкции для ИИ-агентов

CLI-утилита на OneScript (≥ 2.0.0) для автоматизации операций разработчика 1С: сборка/разборка cf/cfe/epf,
работа с ИБ, хранилищем, кластером (через библиотеку [irac](https://github.com/ArKuznetsov/irac)), запуск тестов
(Vanessa-ADD, xUnit, YAxUnit), проверки, MCP-сервер. Кодовая база и коммиты — на русском.

## Устройство

| Путь | Что там |
|---|---|
| `src/main.os` | точка входа CLI (`vrunner`) |
| `src/mcp.os` | точка входа MCP-сервера (`vrunner-mcp`, stdio) |
| `src/cli/Команда<Группа>.os` | группа команд (`&КомандаПриложения`) |
| `src/cli/<Группа>_Подкоманды/[<Подгруппа>_Подкоманды/]Подкоманда<...>.os` | подкоманды (`&ПодкомандаПриложения(Родитель = ...)`), регистрируются автоматически `src/cli/package-loader.os` — ручного реестра нет |
| `src/cli/НаборыОпций/` | переиспользуемые наборы опций (`&НаборОпций`) |
| `src/core/<Пакет>/Классы/` | логика: сервисы-желуди autumn (`&Желудь`, зависимости `&Пластилин`) |
| `tests/unit/<Пакет>/Тест<Класс>.os` | юнит-тесты (oneunit + moskito, каркас `Стенд`) |
| `tests/e2e/{client,server,edt}-tests` | e2e: реальный запуск `src/main.os` с платформой 1С |
| `docs/product/` | документация сайта (VitePress-агрегатор, ссылки без числовых префиксов) |
| `tools/generate-settings-schema.os` | генерация JSON Schema `autumn-properties.json` из аннотаций → `build/` |
| `epf/` | исходники служебных обработок; `.epf` не коммитятся, собираются `epf compile -R epf` |

Командный слой тонкий: опции → `СервисЗаполненияПараметров.Заполнить(ЭтотОбъект)` → вызов сервиса из `src/core`.
Логику и её тесты держи в сервисе.

## Окружение (Windows, PowerShell) — проверено

Библиотеки ставятся в `oscript_modules` (`opm install -l --dev`). Лаунчеры `oscript_modules\bin\*.bat` сами
`oscript_modules` не видят (`{Библиотека не найдена: 'collectionos'}`), а `tasks/*.os` зовут `oneunit` из PATH. Поэтому
в каждой новой сессии PowerShell:

```powershell
chcp 65001 > $null
$env:OSCRIPT_CONFIG = "lib.additional=$PWD\oscript_modules"   # из корня репозитория
$env:PATH = "$PWD\oscript_modules\bin;$env:PATH"
```

Не создавай `oscript.cfg` внутри `oscript_modules\oneunit` — переменной окружения достаточно, а CI этот файл специально удаляет.

| Задача | Команда | Время |
|---|---|---|
| Один файл юнит-тестов | `oneunit execute -f tests/unit/Сервисы/ТестСервисКластера.os` | ~5 с |
| Все юнит-тесты | `oscript tasks/test_unit.os` → `build/reports/junit-unit.xml` | ~1,5 мин, 778 тестов |
| e2e клиентские (нужна платформа 1С) | `oscript tasks/test_e2e.os` | долго |
| e2e EDT (нужен `1cedtcli`) | `oscript tasks/test_e2e_edt.os` | долго |
| e2e кластер (RAS + PostgreSQL, `VRUNNER_TEST_*`) | см. `.github/workflows/e2e.yml` | долго |
| Справка команды | `oscript src/main.os cluster session --help` | |

Запускай CLI как `oscript src/main.os ...`. Команда `vrunner` из PATH — это **установленный** пакет
(`C:\Program Files\OneScript\lib\vanessa-runner`), она правки из репозитория не видит.

## Правила

- Юнит-тесты пиши по `tests/unit/README.md` — это обязательный регламент (Стенд, моки, фикстуры, «Нельзя»), прочитай его до первого теста.
- CI гоняет ubuntu и windows: никаких литералов `C:\...`, всё, что доходит до диска, — через `ТестовыеФайлы`/`ВременныеФайлы`.
- Замечания BSL LS подавляй только точечно (`// BSLLS:<Правило>-off <почему>`), как в соседних файлах.
- `*.os` — CRLF (`.gitattributes`), UTF-8.
- Коммиты — по-русски, одна строка о результате, номер issue в скобках: `Аргументы NAME и JOB команд infobase - из файла настроек (#751)`. PR — в `develop` из веток `feature/…`, `fix/…`, `ci/…`, `refactor/…`.
- Переименование или удаление команд и опций ломает пользователей: обнови `docs/product/миграция/` и `tools/migrate26to30.os`, если затронута совместимость с 2.x.
- Чужие незакоммиченные изменения (например, ` M src/oscript.cfg`) не откатывай и не включай в свой коммит.
- `.github/instructions/` в `.gitignore` — это локальные инструкции агентов, не публикуй их.

## Известные ловушки

- **OneScript 2.2.0** ломает irac (лишний перевод строки в `ПотокВывода.Прочитать()`, rac-вывод рвётся) и `--no-wait` клиента 1С. Кластерные e2e держатся на 2.1.0 (`e2e.yml`).
- В CI oneunit 0.5.0: 0.5.2 требует OneScript 2.2.0.
- `moskito` ставится после `oneunit` из-за `decorator` 2.x/3.x (комментарий в `packagedef`); не меняй порядок зависимостей.
- MCP: имя инструмента — два последних звена команды через `_` (`session_kill`, `cf_make_dist`), выводится автоматически.
- Установленный vrunner грузит irac из **своего** `C:\Program Files\OneScript\lib\vanessa-runner\oscript_modules\irac` (`src/oscript.cfg` → `lib.additional=../oscript_modules`), а не из глобального `lib\irac`. Чтобы обновить irac для vrunner: `cd "C:\Program Files\OneScript\lib\vanessa-runner"; opm install -l -s -f irac-X.ospx`. Переустановка vanessa-runner из хаба возвращает irac из хаба (`ЗависитОт("irac", "1.4.0")`) и откатывает локальные фиксы.
- `cluster jobs lock`/`unlock` падает с `irac\...\ИнформационнаяБаза.os / Ошибка в строке: 473 / Значение не является значением объектного типа`: в `oscript_modules` vanessa-runner стоит irac без фикса `c739eff` (пустые строки в выводе rac). Подробности — в `AGENTS.md` irac.

## Скиллы

- `adding-vrunner-command` — новая команда/подкоманда или новая опция CLI/MCP.
- `using-local-irac-in-vanessa-runner` — проверить vrunner с локальной (изменённой) копией irac.
