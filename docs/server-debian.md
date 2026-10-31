# Сервер VLESS + REALITY на VPS с Debian

Проверено по официальной документации 31.10.2026. Используется Xray-core `26.10.29`, systemd, порт `443/tcp`, один пользователь. Команды этого раздела выполняются **на VPS**, если явно не указано другое.

## 1. Выбор и проверка VPS

Практическая отправная точка для личного использования: 1 vCPU, 1 ГБ RAM, 10 ГБ диска, Debian 13, публичный IPv4, доступ по SSH и возможность открыть `443/tcp`. Это оценка для небольшого числа устройств, а не гарантированная пропускная способность. Debian 12 тоже подходит; текущая stable — [Debian 13](https://www.debian.org/releases/).

Выбирайте локацию с нормальным маршрутом из вашей сети. Перед длительной оплатой проверьте краткосрочный VPS: страна и название хостинга сами по себе не гарантируют доступность IP у российского оператора. Уточните лимит трафика, разрешение личного прокси в правилах провайдера, наличие веб-консоли и условия смены IP.

Для базовой схемы не требуются домен, Nginx, панель 3x-ui, Docker и сертификат Let's Encrypt. Если на VPS уже занят порт 443, сначала выясните кем; инструкция далее рассчитана на отдельный сервер без работающего сайта на этом порту.

Проверьте в панели хостинга:

- входящий SSH: ваш фактический порт, обычно `22/tcp`;
- входящий Xray: `443/tcp`;
- исходящие соединения: DNS, HTTPS и другие нужные назначения разрешены;
- публичный IPv4 действительно назначен серверу или проброшен через NAT хостинга.

Внешний UDP 443 для описанного транспорта не нужен. UDP приложений при поддержке клиента передаётся внутри прокси; это не UDP-listener на VPS.

## 2. Подключение и подготовка Debian

На своём компьютере выполните, заменив `SERVER_IP`:

```bash
ssh root@SERVER_IP
```

Если хостинг выдаёт другого пользователя, подключитесь им и выполните `sudo -i`. При первом подключении сопоставьте отпечаток SSH-ключа с панелью хостинга, если она его показывает.

Далее команды рассчитаны на root:

```bash
cat /etc/os-release
uname -m
apt update
apt full-upgrade -y
apt install -y curl ca-certificates unzip openssl ufw nano python3
timedatectl status
ss -ltnp 'sport = :443'
```

Последняя команда до установки Xray обычно не показывает слушателя. Если есть Nginx, Apache или другая служба, решите конфликт до продолжения. Если обновление требует перезагрузки, выполните `reboot` и подключитесь снова.

Время должно быть корректным. Если синхронизация выключена и в образе нет NTP-службы:

```bash
apt install -y systemd-timesyncd
timedatectl set-ntp true
timedatectl status
```

Если уже работает `chrony`, оставьте его: второй NTP-клиент не нужен.

## 3. Firewall без потери SSH

Для свежего VPS используем UFW. При нестандартном SSH-порте замените `22` фактическим портом **до включения** firewall. Проверьте его в настройках хостинга и выводе `ss -ltnp`. Сохраните текущую SSH-сессию и доступ к веб-консоли.

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 443/tcp
ufw enable
ufw status verbose
```

Откройте **вторую** SSH-сессию и убедитесь, что вход сохранился. Те же порты должны быть разрешены в security group/firewall панели хостинга. На сервере с уже настроенным nftables, Docker или firewalld сначала согласуйте существующие правила; не добавляйте второй независимый менеджер вслепую. [Справочник UFW Debian](https://manpages.debian.org/trixie/ufw/ufw.8.en.html).

## 4. Установка Xray из официального проекта

Скачайте установщик в файл, просмотрите его, затем выполните:

```bash
curl -fL --proto '=https' --tlsv1.2 \
  https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh \
  -o /root/xray-install.sh
less /root/xray-install.sh
bash /root/xray-install.sh install --version 26.10.29
/usr/local/bin/xray version
systemctl stop xray
systemctl cat xray
```

Из `less` выходят клавишей `q`. При ошибке скачивания остановитесь: не запускайте неполный файл. Фиксированная версия делает пример воспроизводимым. Для последующего обновления порядок описан ниже.

Официальный установщик создаёт пользователя `xray`, службу `xray.service`, бинарный файл `/usr/local/bin/xray` и основной конфиг `/usr/local/etc/xray/config.json`. С октября 2026 года конфиг должен читаться группой `xray`: каталог `root:xray 0750`, файл `root:xray 0640`. Инструкции с `nobody` или конфигом `root:root 0600` могут не подходить этой установке. [README установщика](https://github.com/XTLS/Xray-install#readme).

`systemctl cat xray` должен показывать основной `ExecStart` с `-config /usr/local/etc/xray/config.json`. Если там `-confdir`, другой пользователь или дополнительные drop-in, это существующая изменённая установка; сначала выясните, какие конфиги действительно загружаются.

## 5. Выбор сайта прикрытия — target и SNI

Выберите реальный HTTPS-сайт, доступный **с VPS**, с TLS 1.3 и HTTP/2, с сертификатом, подходящим выбранному имени. Лучше использовать имя, которое не служит только перенаправлением на другой хост. Не указывайте ваш VLESS-listener в качестве target: получится петля.

Например, кандидатом можно проверить `www.kernel.org`. Это пример для проверки, а не универсальный рабочий SNI для России:

```bash
TARGET_SERVER_NAME='www.kernel.org'
/usr/local/bin/xray tls ping "$TARGET_SERVER_NAME:443"
openssl s_client \
  -connect "$TARGET_SERVER_NAME:443" \
  -servername "$TARGET_SERVER_NAME" \
  -tls1_3 -alpn h2 \
  -verify_hostname "$TARGET_SERVER_NAME" \
  -verify_return_error </dev/null
```

В выводе OpenSSL ищите TLS 1.3, `ALPN protocol: h2` и успешную проверку сертификата (`Verify return code: 0 (ok)` либо эквивалент). `xray tls ping` дополнительно показывает особенности TLS, в том числе поддержку современных обменов ключами. Если кандидат не проходит проверки, выберите другой и повторите их. [Требования REALITY](https://github.com/XTLS/REALITY/blob/main/README.en.md).

Используйте одно и то же выбранное имя в трёх местах:

- сервер: `target = имя:443`;
- сервер: `serverNames = ["имя"]`;
- клиент: `serverName` / `SNI` / `sni = имя`.

Доступность самого сайта в России и доступность VPS — разные вещи. SNI обычно виден оператору: REALITY не включает ECH автоматически. Не подставляйте произвольный «разрешённый» домен, которому не соответствует target. Проверяйте результат из нужной сети; [измерения фильтрации SNI и подсетей](https://github.com/net4people/bbs/issues/490) описывают разные ограничения.

Неавторизованные подключения REALITY пересылает к target. Это расходует трафик и может привлекать злоупотребления; следите за счётчиками хостинга. При публичном использовании изучите ограничители fallback из [документации REALITY](https://xtls.github.io/en/config/transports/reality.html), прежде чем усложнять конфигурацию.

## 6. Генерация реквизитов

```bash
umask 077
install -d -m 700 /root/reality-secrets
/usr/local/bin/xray uuid > /root/reality-secrets/uuid.txt
/usr/local/bin/xray x25519 > /root/reality-secrets/x25519.txt
openssl rand -hex 8 > /root/reality-secrets/short-id.txt
cat /root/reality-secrets/uuid.txt
cat /root/reality-secrets/x25519.txt
cat /root/reality-secrets/short-id.txt
```

Скопируйте значения в защищённое хранилище:

- `uuid.txt` → `UUID`;
- `PrivateKey` → `PRIVATE_KEY`, остаётся на сервере;
- `Password (PublicKey)` → `REALITY_PASSWORD`, нужен клиенту;
- `short-id.txt` → `SHORT_ID`.

`Hash32` из вывода `x25519` здесь не используется. В старых версиях клиентский ключ мог называться просто `Public key`. Не генерируйте отдельно новую пару на клиенте. Формат вывода текущей версии подтверждён [исходным кодом команды](https://github.com/XTLS/Xray-core/blob/v26.10.29/main/commands/all/curve25519.go).

## 7. Конфигурация сервера

Откройте файл:

```bash
nano /usr/local/etc/xray/config.json
```

Вставьте содержимое [examples/server-reality.json](../examples/server-reality.json). Замените **все** метки:

| Метка | Подстановка |
|---|---|
| `REPLACE_UUID` | Значение `uuid.txt` |
| `REPLACE_PRIVATE_KEY` | `PrivateKey` |
| `REPLACE_SHORT_ID` | Сгенерированные 16 hex-символов |
| `REPLACE_TARGET_SERVER_NAME` | Проверенное имя target; оба вхождения |

Публичный IP VPS не требуется вписывать в этот серверный конфиг. `listen: 0.0.0.0` открывает IPv4-listener. Клиент будет соединяться с публичным IPv4, указанным в панели VPS. Если планируете вход по IPv6, настройте его отдельно и проверьте firewall; данный пример рассчитан на IPv4.

Минимальная рабочая часть шаблона выглядит так; вместо многоточий нужны ваши значения:

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
      "users": [{ "id": "ВАШ_UUID", "flow": "xtls-rprx-vision" }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "raw",
      "security": "reality",
      "realitySettings": {
        "target": "ВАШ_TARGET:443",
        "serverNames": ["ВАШ_TARGET"],
        "privateKey": "ВАШ_PRIVATE_KEY",
        "shortIds": ["ВАШ_SHORT_ID"]
      }
    }
  }],
  "outbounds": [{ "protocol": "freedom" }]
}
```

Для установки используйте **полный шаблон**: он дополнительно ограничивает проксирование в частные/служебные IP-сети VPS и исходящий SMTP-порт 25. Имена назначений разрешаются для применения IP-правил. Правила основаны на механизме [routing Xray](https://xtls.github.io/en/config/routing.html); geodata для них не нужны. Стратегия разрешения исходящих имён задана в `streamSettings.sockopt.domainStrategy`, согласно [актуальному описанию Freedom](https://xtls.github.io/en/config/outbounds/freedom.html). Это базовое ограничение, а не полная изоляция от всех ресурсов облака.

Не добавляйте `tlsSettings`, `allowInsecure`, WebSocket path, HTTP Host или Mux: для приведённой схемы они не требуются. `decryption: none` у VLESS совместимо с шифрованием REALITY на транспортном уровне. [VLESS inbound](https://xtls.github.io/en/config/inbounds/vless.html).

Сохраните в nano: `Ctrl+O`, Enter, `Ctrl+X`. Установите права и проверьте отсутствие меток:

```bash
chown root:xray /usr/local/etc/xray
chmod 750 /usr/local/etc/xray
chown root:xray /usr/local/etc/xray/config.json
chmod 640 /usr/local/etc/xray/config.json
grep -n 'REPLACE_' /usr/local/etc/xray/config.json
python3 -m json.tool /usr/local/etc/xray/config.json >/dev/null
runuser -u xray -- /usr/local/bin/xray run -test \
  -config /usr/local/etc/xray/config.json
```

`grep` не должен ничего вывести; его код 1 здесь нормален. Проверка JSON должна завершиться без ошибок. Проверка Xray должна сообщить об успешной конфигурации. Запуск от `xray` проверяет также доступ службы к файлу. Исправьте ошибки до запуска.

## 8. Запуск и проверка сервиса

```bash
systemctl enable xray
systemctl restart xray
systemctl --no-pager --full status xray
journalctl -u xray -n 50 --no-pager
ss -ltnp 'sport = :443'
```

Ожидается `active (running)`, слушатель `0.0.0.0:443` и отсутствие ошибок загрузки конфига. Если процесс запустился, но listener отсутствует, проверьте `ExecStart`, конфиг и конфликт порта.

Теперь проверьте **со своего компьютера**, а не с самого VPS:

```powershell
Test-NetConnection SERVER_IP -Port 443
```

На Linux:

```bash
nc -vz -w 5 SERVER_IP 443
```

Для `nc` в Debian/Ubuntu нужен пакет `netcat-openbsd`. Успешный результат доказывает только доступность TCP. Дальше требуется настоящий запрос через клиент.

На VPS запишите его внешний IP для сравнения с клиентом:

```bash
curl -4 --max-time 20 https://api.ipify.org
printf '\n'
```

При NAT это именно адрес выхода; он может отличаться от локального интерфейса VPS.

## 9. Ссылка для импорта клиента

Ссылка имеет вид одной непрерывной строки:

```text
vless://UUID@SERVER_IP:443?encryption=none&security=reality&sni=TARGET_SERVER_NAME&fp=chrome&pbk=REALITY_PASSWORD&sid=SHORT_ID&type=tcp&flow=xtls-rprx-vision&spx=%2F#My-Reality
```

`type=tcp` в URI соответствует RAW/TCP в конфигурации Xray. `pbk` — клиентский `Password (PublicKey)`, а не `PrivateKey` и не `Hash32`. Параметры URI кодируются по стандартным правилам URL. [Предложение формата ссылок VLESS](https://github.com/XTLS/Xray-core/discussions/716).

Чтобы не собирать строку вручную, выполните на VPS интерактивный генератор. Он читает реквизиты из файлов шага 6 и текущего серверного конфига; у вас спросит только публичный IPv4. Ссылка сохранится с правами 600:

```bash
python3 - <<'PY'
import ipaddress
import json
import os
from pathlib import Path
from urllib.parse import urlencode

os.umask(0o077)
cfg = json.loads(Path('/usr/local/etc/xray/config.json').read_text())
inbound = next(i for i in cfg['inbounds'] if i.get('tag') == 'vless-reality-in')
reality = inbound['streamSettings']['realitySettings']
lines = Path('/root/reality-secrets/x25519.txt').read_text().splitlines()
password = next(line.split(':', 1)[1].strip() for line in lines
                if line.startswith('Password (PublicKey):'))
# /dev/tty нужен, поскольку stdin занят кодом heredoc.
with open('/dev/tty', 'r+') as tty:
    tty.write('Публичный IPv4 VPS: ')
    tty.flush()
    address = str(ipaddress.IPv4Address(tty.readline().strip()))
user = inbound['settings']['users'][0]
query = urlencode({
    'encryption': 'none', 'security': 'reality',
    'sni': reality['serverNames'][0], 'fp': 'chrome',
    'pbk': password, 'sid': reality['shortIds'][0],
    'type': 'tcp', 'flow': user['flow'], 'spx': '/'
})
uri = f"vless://{user['id']}@{address}:{inbound['port']}?{query}#My-Reality"
path = Path('/root/reality-secrets/client-uri.txt')
path.write_text(uri + '\n')
path.chmod(0o600)
print(uri)
print(f'Сохранено: {path}')
PY
```

Скопируйте ссылку в v2rayN по [инструкции Windows](client-windows.md) или [Linux](client-linux.md). Не вставляйте её в публичные онлайн-генераторы QR и не сохраняйте в открытом репозитории. Для нескольких устройств лучше выпустить отдельные UUID.

## 10. Обслуживание, резервная копия и отзыв доступа

Резервная копия конфига:

```bash
install -d -m 700 /root/xray-backups
cp -a /usr/local/etc/xray/config.json \
  "/root/xray-backups/config-$(date +%Y%m%d-%H%M%S).json"
```

Храните защищённую копию также вне VPS вместе с версией ядра и реквизитами. Каталог копий 700 закрывает доступ остальным пользователям.

Для нового пользователя выполните `xray uuid` и добавьте ещё один объект в `settings.users` с новым `id`, `flow: xtls-rprx-vision`, уникальной меткой `email`. Общие REALITY-ключи и short ID могут оставаться теми же. Сгенерируйте ссылку для нового UUID отдельно: генератор выше всегда берёт первого пользователя.

Для отзыва доступа удалите соответствующий объект пользователя, проверьте конфиг и перезапустите службу. Перезапуск прервёт существующие соединения. Short ID сам по себе не является заменой индивидуальному UUID.

Плановое обновление:

1. Сделайте копию конфига и запишите `xray version`.
2. Прочитайте примечания [официальных релизов](https://github.com/XTLS/Xray-core/releases) и [миграции установщика](https://github.com/XTLS/Xray-install#readme).
3. Скачайте установщик заново командой шага 4 и просмотрите изменения.
4. Выполните `bash /root/xray-install.sh install --version НОВАЯ_ВЕРСИЯ`.
5. Проверьте права, `xray run -test`, `systemctl status`, реальный запрос с клиента.

Для установки текущего стабильного релиза без фиксации версии используется `bash /root/xray-install.sh install`. Не включайте автоматическую смену основной версии без проверки совместимости клиентов. Переход со старого `nobody` на `xray` особенно требует проверки прав.

При неудачном обновлении можно вернуть ранее записанную версию тем же `install --version`, восстановить её конфиг из копии, проверить и перезапустить. Не выбирайте старую версию только ради непроверенного совета об обходе блокировок.

Обычный уровень логирования — `warning`, вывод службы доступен через `journalctl -u xray`. Если временно включаете `info`/`debug`, учитывайте адреса и метаданные в журнале и верните `warning` после диагностики. Прямое указание файлов логов требует прав пользователя `xray`; для ротации официальная инструкция предлагает опцию установщика `--logrotate`.

Для REALITY-прокси не нужно включать `net.ipv4.ip_forward`, делать MASQUERADE или открывать публично SOCKS-порт. Это прикладной прокси. На VPS из этой инструкции наружу слушает только VLESS на 443 и ваш SSH.
