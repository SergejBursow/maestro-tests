# Maestro Test Runs

Этот репозиторий содержит автотесты `Maestro` и несколько вариантов их запуска через GitHub Actions.

Основной рабочий сценарий на текущий момент:

- `self-hosted runner` на Ubuntu-ноутбуке
- реальный Android-телефон, подключённый к этому ноутбуку по USB
- запуск workflow `Maestro Tests on USB Device`

Именно этот вариант сейчас даёт лучший результат по стабильности и времени прогона.

## Структура проекта

- `.maestro/flows` — основные тестовые сценарии
- `.maestro/common` — общие подфлоу и переиспользуемые шаги
- `.maestro/test.yaml` — отдельный smoke-flow
- `.maestro/config.yaml` — конфигурация discovery для Maestro

## Какие workflow есть

- `Maestro Tests on USB Device` — основной сценарий: реальный телефон по USB + `self-hosted runner`
- `Maestro Tests on Real Device` — прогон через STF и удалённый `adb connect`
- `Maestro Tests on Emulator` — прогон на Android-эмуляторе через `self-hosted runner`

## Рекомендуемый сценарий

Использовать `Maestro Tests on USB Device`.

Почему:

- быстрее, чем STF
- нет плавающих `host:port`
- не нужны ADB-ключи для GitHub-hosted runner
- телефон виден локально через обычный `adb`
- `Maestro` на runner уже установлен и не скачивается заново на каждом прогоне

## Что уже настроено

На Ubuntu-ноутбуке:

- установлен `self-hosted runner`
- runner работает как systemd-сервис
- установлен `Maestro 2.3.0`
- установлен `adb`
- телефон виден локально через `adb devices -l`

Также на ноутбуке продолжает работать STF, но для CI основной путь теперь не через STF, а через локальный USB-доступ к телефону.

## Как запускать основной сценарий

### Перед прогоном

На Ubuntu-ноутбуке:

```bash
adb kill-server
adb start-server
adb devices -l
```

Ожидаемый результат:

- телефон должен появиться в списке со статусом `device`

Если телефона нет:

1. подключить или переподключить USB
2. разблокировать телефон
3. подтвердить `USB debugging`, если Android показывает запрос
4. снова выполнить `adb devices -l`

### Проверить приложение

Если нужно убедиться, что приложение установлено:

```bash
adb -s SERIAL shell pm list packages | grep com.example.testmaestro
```

### Запуск workflow

В GitHub:

1. открыть `Actions`
2. выбрать `Maestro Tests on USB Device`
3. нажать `Run workflow`

Поля:

- `android_serial` — можно оставить пустым, если телефон подключён один
- `app_id` — `com.example.testmaestro`
- `test_path` — `.maestro/flows`

## Что делает USB workflow

Workflow `Maestro Tests on USB Device`:

- проверяет наличие `adb`
- находит локально подключённый телефон
- будит устройство и пытается снять блокировку
- проверяет, что приложение установлено
- запускает `Maestro` по `.maestro/flows`
- сохраняет `junit`-отчёт и артефакты в GitHub Actions

## Текущее ожидаемое поведение тестов

На текущем наборе flow ожидаемый результат такой:

- проходят `4 из 6`
- падают `2 из 6`

Падающие сценарии:

- `.maestro/flows/04_add_product_to_cart.yaml`
- `.maestro/flows/05_cart_badge_indicator.yaml`

Причина падения сейчас ожидаемая:

- `Assertion is false: id: cart_badge_count is visible`

Эти два падения в текущем состоянии проекта считаются нормальными и специально не исправляются.

## Время прогона

Практический результат после перехода с STF на `USB + self-hosted runner`:

- через STF полноценный прогон занимал примерно `11–12 минут`
- через USB-сценарий прогон занимает около `3 минут`

Это основной выигрыш новой схемы.

## STF и эмулятор

### STF

STF остаётся в проекте как вспомогательный сценарий:

- для ручной работы
- для визуального доступа к устройству
- для отдельных прогонов через `Maestro Tests on Real Device`, если это понадобится

### Эмулятор

Эмуляторный сценарий тоже настроен:

- `Maestro Tests on Emulator`

Но сейчас он не является основным.

## Что проверять после перезагрузки ноутбука

Минимальная проверка:

```bash
adb kill-server
adb start-server
adb devices -l
```

Если телефон виден как `device`, можно сразу запускать workflow.

Дополнительно при сомнениях можно проверить runner:

```bash
systemctl list-units --type=service | grep actions.runner
```

Ожидаемый статус:

- `active (running)`

## Артефакты

После прогона workflow загружает artifact с результатами:

- `maestro-usb-device-results`

Внутри:

- `maestro-report.xml`
- файлы из `~/.maestro/tests/**/*`

## Примечание

Предупреждение GitHub про `Node.js 20` у `actions/checkout@v4` и `actions/upload-artifact@v4` сейчас не блокирует прогоны и не влияет на результат тестов.
