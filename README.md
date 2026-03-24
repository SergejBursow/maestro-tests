## Maestro + STF

Сейчас workflow подключается к Android-устройству по `adb connect host:port`, а затем запускает `maestro test`.

### Что уже было не так

По первому логу GitHub Actions падение происходило раньше Maestro:

- `failed to connect to '80.82.40.133:5555': Connection timed out`

Это значит, что runner GitHub не смог открыть TCP-соединение к ADB endpoint. Обычно причина одна из этих:

- STF и устройство доступны только в локальной сети другого ноутбука
- был указан неверный ADB-порт
- IP `80.82.40.133` недоступен из интернета
- ADB over TCP не слушает на этом адресе
- между GitHub runner и ноутбуком есть NAT или firewall

Сейчас подтверждён рабочий endpoint устройства из STF:

- `adb connect 80.82.40.133:7405`

Именно такой адрес и нужно использовать в `device_address`.

### Что нужно для рабочего сценария

Есть 2 нормальных варианта:

1. Запускать workflow на `self-hosted` GitHub runner на том ноутбуке, где развернут STF
2. Оставить `ubuntu-latest`, но дать ему публично достижимый `host:port` для `adb connect`

Для домашней STF-фермы почти всегда проще и надёжнее первый вариант.

### Что поменяно в workflow

- Добавлена отдельная TCP-проверка доступности ADB endpoint до `adb connect`
- Ошибка теперь явно объясняет, что нужен `self-hosted runner` или публичный/tunneled ADB endpoint
- Добавлена поддержка ADB-ключа через GitHub Secrets: `ADB_PRIVATE_KEY` и опционально `ADB_PUBLIC_KEY`
- `APP_ID` теперь передаётся как input workflow и используется внутри Maestro
- Все `adb shell` команды идут через конкретный serial устройства

### Если GitHub Actions пишет `unauthorized`

Это уже не проблема сети. Это означает, что runner достучался до устройства, но STF/ADB не доверяет ключу GitHub runner.

Нужно добавить в GitHub Secrets ключ с машины, которая уже может подключаться:

- `ADB_PRIVATE_KEY` = содержимое файла `~/.android/adbkey`
- `ADB_PUBLIC_KEY` = содержимое файла `~/.android/adbkey.pub`

На твоём Mac это можно посмотреть так:

```bash
cat ~/.android/adbkey
cat ~/.android/adbkey.pub
```

После этого добавь secrets в репозиторий:

- `Settings -> Secrets and variables -> Actions -> New repository secret`

И перезапусти workflow.

### Как запускать

В `Actions -> Maestro Tests on Real Device -> Run workflow` передай:

- `device_address`: адрес в формате `host:port`
- `app_id`: например `ru.neopharm.stolichki`
- `test_path`: `.maestro/` или путь к конкретному yaml

### Рекомендуемый следующий шаг

Если STF живёт на другом ноутбуке и не торчит в интернет:

- поставь на него GitHub Actions self-hosted runner
- переведи этот workflow на запуск на self-hosted label
- запускай `adb connect` уже внутри той же сети, где доступен STF/provider

Если захочешь, следующим шагом можно отдельно переделать workflow под `self-hosted` режим для STF-ноутбука.
