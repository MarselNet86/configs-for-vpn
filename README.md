# configs-for-vpn

Конфиги Xray для панели Remnawave: шаблоны подписки Xray JSON с резервом и без, профиль ноды
VLESS Reality + Hysteria2.

## Что внутри

| Файл | Что это |
| --- | --- |
| `xray-json/without-reserve.json` | шаблон подписки: клиент ходит через один сервер |
| `xray-json/with-reserve.json` | шаблон с резервом: сервер не отвечает — клиент сам уходит на запасной |
| `node/vless-reality-hysteria2.json` | профиль ноды: VLESS Reality и Hysteria2 на 443 |

Оба шаблона пускают мимо VPN российские сайты, банки, госсервисы, торренты и SSH. `max.ru` и
`oneme.ru` заблокированы. DNS идёт через DoH Cloudflare и Google, российские домены — через DoH
Яндекса.

## Шаблон подписки

1. В шаблонах подписки панели создайте шаблон типа Xray JSON и вставьте `without-reserve.json`
   или `with-reserve.json`.
2. В настройках хоста выберите этот шаблон. Хост без выбранного шаблона получает шаблон Xray JSON
   по умолчанию.

Шаблон получают клиенты, которые берут подписку в формате Xray JSON, например Happ.

### Резерв

`with-reserve.json` раз в минуту проверяет основной сервер запросом к
`https://www.gstatic.com/generate_204` и при сбое переводит трафик на резервный.

1. Заведите хост резерва на другой ноде и включите у него «Скрыть хост».
2. Поставьте основному хосту и резерву одинаковый тег.
3. Основному хосту выберите шаблон `with-reserve.json`.

Панель сама подставит скрытый хост с тем же тегом в конфиг основного (`injectHosts`).

## Профиль ноды

Нужно: домен с A-записью на IP ноды, открытые 443/tcp и 443/udp, свободный 80/tcp для сертификата.

1. Сертификат на домен. Сайт nginx по умолчанию занимает 80-й порт, поэтому его убираем:

   ```bash
   apt install -y nginx certbot
   rm -f /etc/nginx/sites-enabled/default && systemctl reload nginx
   certbot certonly --standalone -d node.example.com
   ```

2. Ключи Reality на ноде:

   ```bash
   docker exec remnanode xray x25519   # PrivateKey — в профиль
   openssl rand -hex 8                 # shortId
   ```

   Публичный ключ панель вычислит из приватного сама.

3. Сайт-прикрытие. Кто приходит на 443 без ключа, получает сайт с `127.0.0.1:8445` и настоящим
   сертификатом вашего домена. Положите любой статический сайт в `/var/www/site`, добавьте в
   `/etc/nginx/conf.d/site.conf` и выполните `systemctl reload nginx`:

   ```nginx
   server {
       listen 127.0.0.1:8445 ssl http2;
       server_name node.example.com;
       ssl_certificate     /etc/letsencrypt/live/node.example.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/node.example.com/privkey.pem;
       root /var/www/site;
   }
   ```

4. Hysteria2 читает сертификат внутри контейнера. Добавьте в `docker-compose.yml` ноды и
   перезапустите её (`docker compose up -d`):

   ```yaml
       volumes:
         - /etc/letsencrypt:/etc/letsencrypt:ro
   ```

5. В панели создайте профиль конфигурации из `node/vless-reality-hysteria2.json` и замените:
   - `node.example.com` — на ваш домен (4 места);
   - `REPLACE_WITH_REALITY_PRIVATE_KEY` — на PrivateKey;
   - `REPLACE_WITH_SHORT_ID` — на shortId.

   Привяжите профиль к ноде.

6. Хосты для инбаундов `VLESS-REALITY` и `HY2`: адрес и SNI — ваш домен, порт 443. У
   `VLESS-REALITY` отпечаток `chrome`.

Проверка: `https://node.example.com` в браузере открывает ваш сайт, а в ссылке подписки VLESS есть
`flow=xtls-rprx-vision`.
