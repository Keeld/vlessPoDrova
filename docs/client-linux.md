# Клиент VLESS + REALITY в Linux

Проверено по документации 31.10.2026. Для рабочего стола используйте v2rayN с импортом ссылки и, при необходимости, TUN. Для терминала и серверов без GUI ниже есть отдельный вариант с Xray-core и службой пользователя.

Команды установки пакетов приведены для Debian/Ubuntu. В Fedora используйте соответствующие RPM; в других дистрибутивах — официальный переносимый архив. Серверные и клиентские команды выполняются на **разных машинах**.

## Вариант A. Рабочий стол: v2rayN

### A1. Установка

Узнайте архитектуру:

```bash
uname -m
```

`x86_64` означает x64/amd64, `aarch64` — ARM64. Откройте [официальный релиз v2rayN](https://github.com/2dust/v2rayN/releases/latest); на дату проверки Latest — `7.25.6`.

Для Debian 12+/Ubuntu 22.04+ скачайте `v2rayN-linux-64.deb` либо `v2rayN-linux-arm64.deb`. Перейдите в каталог, куда вы его сохранили, и выполните:

```bash
sudo apt update
sudo apt install ./v2rayN-linux-64.deb
```

Для ARM64 замените имя файла. Запустите v2rayN из меню приложений. Используйте обычного пользователя. Для Fedora/Red Hat официальная [таблица пакетов](https://github.com/2dust/v2rayN/wiki/Release-files-introduction) указывает RPM-варианты; установка x64:

```bash
sudo dnf install ./v2rayN-linux-rhel-x64.rpm
```

Переносимый вариант: распакуйте `v2rayN-linux-64.zip` в отдельную папку пользователя и из неё выполните:

```bash
chmod +x ./v2rayN
./v2rayN
```

Если ошибка относится к отсутствующим GUI-библиотекам, используйте пакет под ваш дистрибутив, который устанавливает зависимости. На машине без рабочего стола используйте вариант B.

### A2. Импорт и первая проверка

1. Получите ссылку из [серверной инструкции](server-debian.md) либо у администратора.
2. Скопируйте её полностью; в v2rayN выберите импорт из буфера обмена, обычно `Ctrl+V`.
3. Откройте профиль и проверьте: адрес — VPS, порт 443, протокол VLESS, flow `xtls-rprx-vision`, сеть TCP/RAW, security REALITY, SNI и ключ совпадают с сервером, fingerprint `chrome`, short ID заполнен.
4. Выберите ядро **Xray**, сделайте профиль активным. Mux для первого запуска оставьте выключенным.
5. Обновите Xray отдельным пунктом обновления ядер. Примеры здесь рассчитаны на `26.10.29`. Интерфейс и ядро имеют разные версии.
6. Узнайте в настройках фактический порт SOCKS/смешанного listener. Далее предполагается `10808`. Доступ из LAN оставьте выключенным.
7. Пока TUN выключен, проверьте в терминале:

```bash
curl --max-time 20 --proxy socks5h://127.0.0.1:10808 https://api.ipify.org
curl --max-time 30 --proxy socks5h://127.0.0.1:10808 -I https://example.com
```

Первый запрос должен вернуть IP выхода VPS, второй — HTTP-заголовки. `socks5h` отправляет домен через прокси. Если тест не проходит, сначала исправьте профиль/сервер по [диагностике](troubleshooting.md).

При ручном добавлении профиля используйте [таблицу полей Windows](client-windows.md): она одинакова для обеих платформ. Название `Public key` означает клиентский REALITY `password`, а не приватный ключ сервера.

### A3. Браузеры и приложения через системный прокси

В v2rayN включите установку системного прокси и маршрутизацию Global для первой проверки. GNOME/KDE и отдельные приложения учитывают системный прокси по-разному. Если браузер его игнорирует, задайте в браузере SOCKS v5 `127.0.0.1:10808`, включив удалённое разрешение DNS. В Firefox это **Настройки → Настройки сети → Ручная настройка прокси → SOCKS v5 → Proxy DNS when using SOCKS v5**.

Для одной терминальной команды можно продолжать использовать `curl --proxy …`. Для программ, понимающих переменные окружения:

```bash
export ALL_PROXY='socks5h://127.0.0.1:10808'
```

Если нужен HTTP-прокси, возьмите его фактический порт в настройках v2rayN. В современных сборках смешанный порт может принимать одновременно SOCKS и HTTP; не считайте, что порт HTTP обязательно равен 10809. Для смешанного 10808:

```bash
export http_proxy='http://127.0.0.1:10808'
export https_proxy='http://127.0.0.1:10808'
export no_proxy='localhost,127.0.0.1,::1'
```

`https_proxy` здесь содержит `http://`: это HTTP CONNECT к локальному прокси. Переменные действуют для текущей оболочки и запущенных из неё программ, не для всей системы. Некоторые программы их игнорируют; `sudo` обычно не переносит их автоматически.

Чтобы отменить изменения этой оболочки:

```bash
unset ALL_PROXY all_proxy http_proxy https_proxy HTTP_PROXY HTTPS_PROXY no_proxy NO_PROXY
```

Удаляйте только переменные, которые больше не нужны; если прокси был настроен до этого, сохраните прежние значения.

### A4. TUN для приложений без поддержки прокси

1. Завершите другие VPN-клиенты и убедитесь, что обычный SOCKS-тест проходит.
2. Отключите установку системного прокси для первой проверки TUN.
3. Включите TUN в v2rayN. Для создания интерфейса и маршрутов нужны расширенные сетевые права: используйте штатный запрос авторизации приложения с паролем локального пользователя с `sudo`, если он появляется.
4. Включите автоматическую маршрутизацию. Начните с настроек стека и MTU по умолчанию. Для Xray TUN используется `gvisor`; отдельный TUN-компонент может иметь другие параметры.
5. Проверьте настройки обработки DNS, исключения LAN и маршрута к IP VPS. Весь IPv6 должен туннелироваться либо явно блокироваться, если ваша цель — весь интернет через VPS.
6. Проверьте маршруты и выход без прикладного прокси:

```bash
ip -brief addr
ip rule show
ip route show table all
curl --noproxy '*' --max-time 20 https://api.ipify.org
curl --noproxy '*' --max-time 30 -I https://example.com
```

Ожидается IP VPS. `ip route show` одной основной таблицы недостаточно: TUN может применять policy routing. Для проверки не выбирайте ICMP ping — его поведение зависит от ядра.

Если TUN не создаётся:

```bash
ls -l /dev/net/tun
sudo -v
```

Если устройство отсутствует на обычном Linux-хосте, проверьте наличие модуля:

```bash
sudo modprobe tun
ls -l /dev/net/tun
```

В контейнере/ограниченной VM устройство и `NET_ADMIN` должен предоставить владелец среды. При `operation not permitted` смотрите журнал запуска TUN и механизм авторизации выбранной сборки; GUI продолжайте запускать обычным пользователем. Особенности прав Linux разбираются в [обсуждении разработчика v2rayN](https://github.com/2dust/v2rayN/discussions/7824). Произвольное правило `NOPASSWD` для изменяемого пользователем бинарника не нужно для базовой установки.

После успешной проверки настройте автозапуск через средства рабочего стола/v2rayN и испытайте после нового входа в систему. TUN и системный прокси имеют разную область действия, см. [официальное описание](https://github.com/2dust/v2rayN/wiki/Description-of-system-proxy-routing).

## Вариант B. Терминал без GUI: Xray-core

Здесь Xray слушает только `127.0.0.1:10808` (SOCKS) и `127.0.0.1:10809` (HTTP). Это надёжная отправная точка для терминала и браузера с ручным прокси. **Этот конфиг не создаёт TUN и не перехватывает автоматически весь трафик Linux.** Не запускайте одновременно вариант A на тех же портах.

### B1. Загрузка официального ядра

Установите инструменты:

```bash
sudo apt update
sudo apt install -y curl ca-certificates unzip python3 nano
mkdir -p "$HOME/.cache/xray-reality-download"
cd "$HOME/.cache/xray-reality-download"
```

Загрузите архив под архитектуру и файл контрольных сумм. Блок рассчитан на **Bash**:

```bash
(
  set -euo pipefail
  XRAY_VERSION='26.10.29'
  case "$(uname -m)" in
    x86_64) XRAY_ASSET='Xray-linux-64.zip' ;;
    aarch64|arm64) XRAY_ASSET='Xray-linux-arm64-v8a.zip' ;;
    *) echo 'Выберите архив своей архитектуры в официальном релизе'; exit 1 ;;
  esac
  XRAY_URL="https://github.com/XTLS/Xray-core/releases/download/v${XRAY_VERSION}"
  curl -fL --proto '=https' --tlsv1.2 \
    "$XRAY_URL/$XRAY_ASSET" -o "$XRAY_ASSET"
  curl -fL --proto '=https' --tlsv1.2 \
    "$XRAY_URL/$XRAY_ASSET.dgst" -o "$XRAY_ASSET.dgst"
  XRAY_SHA256="$(sha256sum "$XRAY_ASSET" | cut -d ' ' -f 1)"
  grep -Fqi "$XRAY_SHA256" "$XRAY_ASSET.dgst" || {
    echo 'Контрольная сумма не совпала: не распаковывайте архив'; exit 1;
  }
  mkdir -p "$HOME/.local/lib/xray-reality"
  unzip -o "$XRAY_ASSET" -d "$HOME/.local/lib/xray-reality"
  chmod 755 "$HOME/.local/lib/xray-reality/xray"
  "$HOME/.local/lib/xray-reality/xray" version
)
```

При ошибке остановитесь и исправьте её до следующего шага. Официальные файлы находятся в [релизе Xray 26.10.29](https://github.com/XTLS/Xray-core/releases/tag/v26.10.29). Контрольная сумма из того же релиза проверяет целостность скачивания; доверие к источнику по-прежнему основано на официальном проекте и HTTPS.

### B2. Конфигурация клиента

```bash
umask 077
mkdir -p "$HOME/.config/xray-reality"
chmod 700 "$HOME/.config/xray-reality"
nano "$HOME/.config/xray-reality/config.json"
```

Вставьте **полный** [examples/client-reality.json](../examples/client-reality.json) и замените:

| Метка | Что вписать |
|---|---|
| `REPLACE_SERVER_IP` | Публичный IPv4 VPS |
| `REPLACE_UUID` | Ваш UUID из серверной конфигурации |
| `REPLACE_TARGET_SERVER_NAME` | То же имя, что в серверном `serverNames` |
| `REPLACE_REALITY_PASSWORD` | Серверный `Password (PublicKey)` |
| `REPLACE_SHORT_ID` | Значение из серверного `shortIds` |

Остальные значения в примере уже согласованы с сервером: порт 443, RAW, REALITY, `chrome`, Vision. Основой формата служат [VLESS outbound](https://xtls.github.io/en/config/outbounds/vless.html) и [SOCKS inbound](https://xtls.github.io/en/config/inbounds/socks.html).

У клиента только один outbound — VLESS. Запрос, принятый этими локальными портами, самопроизвольно не переключается на прямой выход при недоступности VPS. Трафик приложений, которые не используют эти порты, остаётся обычным системным трафиком.

Сохраните и проверьте:

```bash
chmod 600 "$HOME/.config/xray-reality/config.json"
grep -n 'REPLACE_' "$HOME/.config/xray-reality/config.json"
python3 -m json.tool "$HOME/.config/xray-reality/config.json" >/dev/null
"$HOME/.local/lib/xray-reality/xray" run -test \
  -config "$HOME/.config/xray-reality/config.json"
```

`grep` должен быть без вывода; его код 1 означает отсутствие меток. Запустите ядро на переднем плане:

```bash
"$HOME/.local/lib/xray-reality/xray" run \
  -config "$HOME/.config/xray-reality/config.json"
```

В другом терминале:

```bash
curl --max-time 20 --proxy socks5h://127.0.0.1:10808 https://api.ipify.org
curl --max-time 30 --proxy http://127.0.0.1:10809 -I https://example.com
ss -ltnp 'sport = :10808'
ss -ltnp 'sport = :10809'
```

Ожидается IP VPS и успешные заголовки HTTPS. Listener должен быть только на `127.0.0.1`, не на `0.0.0.0`. Остановите процесс `Ctrl+C` перед запуском службы.

### B3. Служба systemd пользователя

Если в вашей Linux-системе есть systemd user session:

```bash
mkdir -p "$HOME/.config/systemd/user"
cat > "$HOME/.config/systemd/user/xray-reality-client.service" <<'EOF'
[Unit]
Description=Xray VLESS REALITY client

[Service]
ExecStart=%h/.local/lib/xray-reality/xray run -config %h/.config/xray-reality/config.json
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
UMask=0077

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now xray-reality-client
systemctl --user --no-pager status xray-reality-client
journalctl --user -u xray-reality-client -n 50 --no-pager
```

Служба работает от вашего пользователя и запускается при входе. Если требуется работа **до входа и после выхода**, отдельно включите lingering:

```bash
sudo loginctl enable-linger "$USER"
```

Если это не нужно, оставьте стандартный запуск при входе. Для отмены lingering — `sudo loginctl disable-linger "$USER"`. В минимальных контейнерах и системах без systemd используйте запуск на переднем плане либо штатный supervisor вашего дистрибутива.

Для изменения конфига: отредактируйте его, выполните `xray run -test`, затем:

```bash
systemctl --user restart xray-reality-client
```

### B4. Подключение приложений

Firefox: SOCKS v5 `127.0.0.1:10808`, DNS через SOCKS. Для одной команды:

```bash
curl --proxy socks5h://127.0.0.1:10808 https://example.com
```

Для текущей оболочки:

```bash
export ALL_PROXY='socks5h://127.0.0.1:10808'
export http_proxy='http://127.0.0.1:10809'
export https_proxy='http://127.0.0.1:10809'
export no_proxy='localhost,127.0.0.1,::1'
```

В отличие от GUI-примера, в этом JSON HTTP действительно находится на **10809**. Для APT разовый запрос можно направить так:

```bash
sudo apt -o Acquire::http::Proxy='http://127.0.0.1:10809' \
  -o Acquire::https::Proxy='http://127.0.0.1:10809' update
```

Это не постоянная настройка системы. Для программ, использующих только UDP/собственный сетевой стек, понадобится их собственная поддержка SOCKS5 UDP или вариант A с TUN. Внешний UDP 443 на VPS открывать ради этого не требуется. QUIC в обычном Vision обрабатывается отдельно, см. [документацию flow](https://xtls.github.io/en/config/outbounds/vless.html).

### B5. Обновление и остановка

Перед обновлением сохраните конфиг и номер рабочей версии. Остановите службу, повторите загрузку B1 с проверенной новой версией, выполните проверку конфига, запустите службу и повторите реальный запрос. Не распаковывайте обновление поверх исполняющегося бинарника.

```bash
systemctl --user stop xray-reality-client
# После обновления и проверки:
systemctl --user start xray-reality-client
```

Остановить и убрать автозапуск:

```bash
systemctl --user disable --now xray-reality-client
```

После этого отмените переменные прокси в оболочке и ручной прокси браузера, если они больше не нужны. Иначе приложения будут обращаться к выключенному localhost-прокси.
