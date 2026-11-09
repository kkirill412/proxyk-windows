# ProxyK Client для Windows

Версия 0.2.0, сборка этапа 2 для Windows 10/11 AMD64. Для графического клиента нужен Microsoft Edge WebView2 Runtime. Клиент работает от имени текущего пользователя, отдельная служба управляет sing-box.

Готовые файлы находятся в `bin\proxyk-client.exe` и `bin\proxyk-service.exe`. Интерфейс поддерживает управление профилями, правилами, темой и автозапуском. Сетевой режим этапа 2 открывает SOCKS5 с паролем только на `127.0.0.1`. Клиент проверяет HTTPS через прокси перед показом состояния «Прокси подключён».

Системная маршрутизация и Kill Switch пока недоступны. Правила сайтов и приложений сохраняются для следующего этапа. TUN проверяется отдельно в лаборатории на одном тестовом адресе.

## Совместимость с ProxyK Admin

Клиент импортирует ссылки ProxyK Admin Linux v0.3.0. Проверена версия исходников `e584e0b63dfde7431ce519f9fbff147982fc12e4`. Формат экспорта использует VLESS с REALITY поверх TCP, `xtls-rprx-vision` и `chrome`. Для импорта достаточно самой ссылки из админки. [Исходники ProxyK Admin](https://github.com/kkirill412/proxyk-admin-linux/tree/e584e0b63dfde7431ce519f9fbff147982fc12e4)

В текущем серверном проекте закреплён Xray 26.10.31. В изолированном испытании официальный стабильный sing-box 1.14.4 получил отказ REALITY от этой версии Xray. Официальный клиент Xray с тем же профилем успешно передал HTTPS. Xray требует гибридный обмен ключами `X25519MLKEM768`, который этот выпуск sing-box удаляет из ClientHello. Подключение к текущей версии сервера остаётся заблокированным несовместимостью ядер. [Версия Xray в ProxyK Admin](https://github.com/kkirill412/proxyk-admin-linux/blob/e584e0b63dfde7431ce519f9fbff147982fc12e4/deploy/compose.yaml#L46), [код sing-box](https://github.com/SagerNet/sing-box/blob/v1.14.4/common/tls/reality_client.go#L149-L161), [обсуждение совместимости](https://github.com/SagerNet/sing-box/issues/4520)

## Сборка

Для разработки нужны Go 1.26.10 и Node.js 22.12+ (сборка проверена с Node.js 24). Версии Go-модулей и пакетов npm закреплены в `go.mod`, `go.sum` и `frontend/package-lock.json`. Wails, Vue и Element Plus используют лицензию MIT. [Требования Wails 3](https://v3.wails.io/quick-start/installation/)

Выполните эти команды из корня проекта в PowerShell.

```powershell
npm ci --prefix frontend
npm run build --prefix frontend
$env:GOOS = 'windows'
$env:GOARCH = 'amd64'
$env:CGO_ENABLED = '0'
go test -tags production ./...
New-Item -ItemType Directory -Path bin -Force | Out-Null
go build -mod=readonly -tags production -trimpath -ldflags '-s -w -H=windowsgui' -o bin/proxyk-client.exe ./cmd/client
go build -mod=readonly -trimpath -ldflags '-s -w -H=windowsgui' -o bin/proxyk-service.exe ./cmd/service
```

Из Linux или WSL те же команды npm собирают интерфейс. Затем соберите EXE.

```sh
mkdir -p bin
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -mod=readonly -tags production -trimpath -ldflags '-s -w -H=windowsgui' -o bin/proxyk-client.exe ./cmd/client
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -mod=readonly -trimpath -ldflags '-s -w -H=windowsgui' -o bin/proxyk-service.exe ./cmd/service
```

## Ручная установка для испытаний

В PowerShell от имени пользователя, который будет запускать интерфейс, получите его SID и загрузите проверяемый официальный sing-box. Скрипт сверяет SHA-256 ZIP и извлечённого EXE. Ядро поставляется отдельно от ProxyK. [Официальный выпуск и исходники sing-box 1.14.4](https://github.com/SagerNet/sing-box/releases/tag/v1.14.4)

```powershell
[Security.Principal.WindowsIdentity]::GetCurrent().User.Value
& .\scripts\fetch-sing-box.ps1
```

Затем откройте 64-разрядный PowerShell от имени администратора. Из доверенной папки проекта выполните команды с SID пользователя из предыдущего шага.

```powershell
$ownerSid = '<SID пользователя интерфейса>'
$clientHash = (Get-FileHash .\bin\proxyk-client.exe -Algorithm SHA256).Hash
$serviceHash = (Get-FileHash .\bin\proxyk-service.exe -Algorithm SHA256).Hash
& .\scripts\install-service.ps1 -OwnerSid $ownerSid -ClientSha256 $clientHash -ServiceSha256 $serviceHash
```

Скрипт копирует файлы в `%ProgramFiles%\ProxyK Client` и запускает службу `ProxyKVPN`. Запустите установленный `proxyk-client.exe` под учётной записью владельца, выберите профиль и нажмите «Подключиться». При успешном подключении окно показывает параметры локального SOCKS5. В тестовой программе укажите их и включите разрешение доменов через прокси (SOCKS5h). При переподключении порт и пароль могут измениться.

Для удаления выключите автозапуск интерфейса, если включали, и закройте окно. Затем выполните `& .\scripts\uninstall-service.ps1` от имени администратора. Локальные профили останутся в хранилище пользователя.

## Устройство приложения

- `frontend` содержит интерфейс Vue 3 и Element Plus. Wails встраивает файлы интерфейса в EXE.
- `cmd/client` запускает окно Wails и открывает системный диалог выбора файла.
- `internal/client` обрабатывает действия интерфейса и записывает настройки. Через Wails интерфейс получает имя, адрес и порт профиля.
- `internal/profile` разбирает и проверяет VLESS-ссылки.
- `internal/storage` использует DPAPI текущего пользователя. В `%LOCALAPPDATA%\ProxyK Client\settings.dpapi` хранится зашифрованный JSON. При записи новый зашифрованный файл атомарно заменяет предыдущий. [Документация Windows DPAPI](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)
- `cmd/service`, `internal/ipc` и `internal/vpn` запускают службу, проверяют подлинность сторон именованного канала, управляют одним процессом ядра и выполняют ограниченное переподключение.
- `internal/singbox` формирует конфигурацию, проверяет хеш и версию ядра, передаёт настройки через стандартный ввод и проверяет доступность HTTPS через аутентифицированный SOCKS5. У прокси единственный выход через VLESS. [VLESS в sing-box](https://sing-box.sagernet.org/configuration/outbound/vless/), [REALITY и uTLS в sing-box](https://sing-box.sagernet.org/configuration/shared/tls/)

Список установленных программ берётся из App Paths Windows. Для программ, которых там нет, доступен выбор файла `.exe`. Правила хранят полный путь к файлу. Правило сайта хранит домен для будущей маршрутизации самого домена и его поддоменов. Для автозапуска интерфейса используется запись `ProxyK Client` в разделе `Run` текущего пользователя. [App Paths](https://learn.microsoft.com/en-us/windows/win32/shell/app-registration), [раздел Run](https://learn.microsoft.com/en-us/windows/win32/setupapi/run-and-runonce-registry-keys)

Канал принимает только локальные запросы владельца. Интерфейс сверяет владельца канала с LocalSystem и его процесс с запущенной службой до передачи профиля. Служба повторно валидирует поля профиля и хранит его в памяти до остановки соединения. [Модель доступа к именованным каналам Windows](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights)

## Приёмка на Windows

1. Запустите EXE из постоянной папки на диске Windows под обычной учётной записью.
2. Скопируйте VLESS-ссылку тестового пользователя из ProxyK Admin. Импортируйте её и сверьте имя, сервер и порт. Попробуйте импортировать ошибочную ссылку.
3. Добавьте второй профиль, переименуйте его и выберите активным. Добавьте сайт и программу, проверьте переключатели правил и выберите другую тему.
4. Закройте и снова откройте приложение. Проверьте восстановление профилей и настроек. Удалите активный профиль и проверьте выбор оставшегося.
5. Включите запуск вместе с Windows и войдите в ту же учётную запись заново. Проверьте открытие окна. Затем выключите автозапуск.

## Выполненные проверки

- На Windows 11 Pro AMD64 проверены DPAPI, обмен по именованному каналу, отказ чужому SID и подменённому серверу канала. Собраны оба EXE. Состояния, отмена и переподключение покрыты тестами менеджера. Интерфейс подключения проверен в Chromium с имитацией вызовов службы.
- Официальный Windows-бинарник sing-box запущен с SOCKS5. Проверены владение локальным сокетом, остановка и завершение процесса. Конфигурации TUN для тестового IPv4 и IPv6 прошли `sing-box check`.
- На лабораторном Xray 26.6.27 Windows-версия sing-box передала HTTPS через VLESS/REALITY. Неверный UUID и запрос без пароля SOCKS5 отклонялись. После остановки Xray запросы через прокси завершались ошибкой, менеджер переходил в «Переподключение», а команда отключения завершала процесс.
- Для приёмки остаются установка службы под LocalSystem и живой TUN в отдельной Windows-среде с правами администратора, проверка актуального рабочего сервера и проверка Windows 10. Автозапуск при входе в Windows также требует отдельной проверки.

Из WSL с Python 3, Go, OpenSSL и curl можно воспроизвести оба протокольных сценария с временным сервером и тестовыми профилями.

```sh
python3 tests/stage2-wsl.py /path/to/xray-26.10.31/xray /path/to/sing-box.exe
python3 tests/stage2-wsl.py --legacy /path/to/xray-26.6.27/xray /path/to/sing-box.exe
```

Первый прогон подтверждает ожидаемый отказ официального sing-box с положительным контролем через Xray. Второй проверяет передачу данных на совместимой лабораторной версии. [Выпуски Xray](https://github.com/XTLS/Xray-core/releases)
