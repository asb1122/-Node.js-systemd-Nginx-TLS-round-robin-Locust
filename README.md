# Лабораторная работа: Node.js в systemd + Nginx (TLS, round-robin) + Locust

Пошаговая инструкция по развёртыванию стенда с приложением [dos-36-js](https://github.com/anestesia001/dos-36-js): два экземпляра под systemd, Nginx как reverse proxy с TLS и балансировкой round-robin, нагрузочное тестирование Locust.

---

## 0. Исходные данные и предпосылки

| Параметр | Значение в примере | Комментарий |
|----------|-------------------|-------------|
| IP сервера | `192.168.2.105` | Замените на свой |
| Пользователь-администратор | `artsiom` | Должен быть в `sudo` |
| Пользователь приложения | `webadmin:webgroup` | Владелец файлов приложения |
| Каталог приложения | `/var/www/app16/nodejs_app` | Куда клонируется репозиторий |
| Порты бэкендов | `3000`, `3001` | Два экземпляра |
| Порт Nginx | `443` (TLS), `80` (редирект) | |

Установка пакетов:

```bash
sudo apt update
sudo apt install -y git nginx openssl python3 python3-venv pipx
```

Node.js ≥ 18 (NodeSource или `nvm`). Проверка:

```bash
node -v && npm -v
```

---

## 1. Клонирование приложения и установка зависимостей

```bash
sudo mkdir -p /var/www/app16
sudo chown -R artsiom:artsiom /var/www/app16
git clone https://github.com/anestesia001/dos-36-js.git /var/www/app16/nodejs_app
cd /var/www/app16/nodejs_app
npm install
```

Проверьте, что `server.js` читает порт из переменной окружения:

```bash
grep -n "listen\|process.env.PORT" server.js
```

Должно быть что-то вроде `const PORT = process.env.PORT || 3000;`. Если порт захардкожен — исправьте на `process.env.PORT || 3000`, иначе второй экземпляр упрётся в `EADDRINUSE`.

Права на каталог (важно для systemd-юнита, который будет работать от `webadmin`):

```bash
sudo chown -R webadmin:webgroup /var/www/app16/nodejs_app
sudo chmod -R 755 /var/www/app16/nodejs_app
```

---

## 2. Два systemd-сервиса (порты 3000 и 3001)

### 2.1. Юнит для первого экземпляра

```bash
sudo nano /etc/systemd/system/app16_3000.service
```

```ini
[Unit]
Description=Node.js Diary App on port 3000
After=network.target

[Service]
Type=simple
User=webadmin
Group=webgroup
WorkingDirectory=/var/www/app16/nodejs_app
ExecStart=/usr/bin/node server.js
Environment=PORT=3000
Environment=NODE_ENV=production
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 2.2. Юнит для второго экземпляра

```bash
sudo cp /etc/systemd/system/app16_3000.service /etc/systemd/system/app16_3001.service
sudo nano /etc/systemd/system/app16_3001.service
```

Измените **только** `Description` и `Environment=PORT=3001`:

```ini
Description=Node.js Diary App on port 3001
...
Environment=PORT=3001
```

### 2.3. Запуск и проверка

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now app16_3000 app16_3001
sudo systemctl status app16_3000 app16_3001 --no-pager
```

Проверьте, что оба слушают свои порты:

```bash
sudo ss -tlnp | grep -E ':3000|:3001'
curl -s -o /dev/null -w "3000: %{http_code}\n3001: %{http_code}\n" http://127.0.0.1:3000/ http://127.0.0.1:3001/
```

Оба должны вернуть `200`.

> ⚠️ Если `3001` падает с `status=1/FAILURE` — смотрите `sudo journalctl -u app16_3001 -n 50 --no-pager`. Частые причины: в юните остался `PORT=3000`, порт занят, либо в `server.js` порт захардкожен.

---

## 3. TLS-сертификат (самоподписанный, с SAN)

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/diary.key \
  -out /etc/nginx/ssl/diary.crt \
  -subj "/CN=192.168.2.105" \
  -addext "subjectAltName=IP:192.168.2.105,DNS:localhost"
```

Проверьте SAN:

```bash
openssl x509 -in /etc/nginx/ssl/diary.crt -noout -text | grep -A1 "Subject Alternative Name"
```

Должно быть `IP Address:192.168.2.105, DNS:localhost` — это позволит обращаться и по IP, и по `localhost` без ошибок несоответствия имени.

---

## 4. Nginx: reverse proxy, TLS, round-robin, расширенный лог

### 4.1. `log_format` в `/etc/nginx/nginx.conf`

```bash
sudo nano /etc/nginx/nginx.conf
```

Внутри блока `http { ... }`, **до** `include /etc/nginx/sites-enabled/*;`, добавьте:

```nginx
log_format upstream_log '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" '
                        'to: $upstream_addr '
                        'upstream_time: $upstream_response_time '
                        'total_time: $request_time';
```

> ⚠️ `log_format` недопустим внутри `server { }`. Только в `http { }`, иначе Nginx выдаст `"log_format" directive is not allowed here`.

### 4.2. Конфиг сайта `/etc/nginx/sites-available/app16`

```bash
sudo nano /etc/nginx/sites-available/app16
```

```nginx
upstream backend {
  server 127.0.0.1:3000;
  server 127.0.0.1:3001;
}

server {
  listen 80;
  server_name 192.168.2.105 localhost;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name 192.168.2.105 localhost;

  ssl_certificate     /etc/nginx/ssl/diary.crt;
  ssl_certificate_key /etc/nginx/ssl/diary.key;

  ssl_protocols       TLSv1.2 TLSv1.3;
  ssl_ciphers         HIGH:!aNULL:!MD5;
  ssl_session_cache   shared:SSL:10m;
  ssl_session_timeout 10m;

  access_log /var/log/nginx/app16.access.log upstream_log;
  error_log  /var/log/nginx/app16.error.log;

  location / {
    proxy_pass http://backend;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

### 4.3. Активация

```bash
sudo ln -sf /etc/nginx/sites-available/app16 /etc/nginx/sites-enabled/app16
sudo rm -f /etc/nginx/sites-enabled/default   # чтобы не было конфликта server_name "_"
sudo nginx -t
sudo systemctl restart nginx
```

> Используйте `restart`, а не `reload` — новый `log_format` подхватывается надёжнее.

### 4.4. Проверка TLS и редиректа

```bash
# HTTP → HTTPS
curl -I http://192.168.2.105/
# Ожидаем: HTTP/1.1 301 Moved Permanently, Location: https://192.168.2.105/

# HTTPS
curl -k -I https://192.168.2.105/
# Ожидаем: HTTP/2 200

# API
curl -k -s https://192.168.2.105/api/notes
# Ожидаем: JSON со списком заметок

# POST
curl -k -i -X POST https://192.168.2.105/add -d "text=test" \
  -H "Content-Type: application/x-www-form-urlencoded"
# Ожидаем: 302, Location: /notes
```

### 4.5. Проверка балансировки

```bash
for i in {1..10}; do curl -k -s -o /dev/null https://192.168.2.105/; done
sudo tail -n 20 /var/log/nginx/app16.access.log | grep -o 'to: 127.0.0.1:300[01]' | sort | uniq -c
```

Ожидаемо ~5/5:

```
      5 to: 127.0.0.1:3000
      5 to: 127.0.0.1:3001
```

---

## 5. Установка Locust

Системный Python в Ubuntu защищён (`externally-managed-environment`), поэтому используем `pipx`:

```bash
pipx install locust
pipx ensurepath
source ~/.bashrc
locust -V
```

Если `locust: command not found` — добавьте вручную в `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc
```

> ⚠️ Не запускайте `sudo locust` — `sudo` использует свой `PATH`, где `~/.local/bin` нет, и root-права Locust не нужны.

---

## 6. Подготовка `locustfile.py` к самоподписанному сертификату

В Locust 2.47 **нет** флага `--skip-ssl-verify`. Отключение проверки SSL делается через код.

Откройте `locustfile.py` и добавьте:

```python
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
```

и в класс `DiaryUser`:

```python
class DiaryUser(HttpUser):
    tasks = [DiaryFlow]
    wait_time = between(1, 3)

    def on_start(self):
        self.client.verify = False
```

> ⚠️ Остальной код `locustfile.py` из репозитория менять не нужно.

---

## 7. Запуск нагрузочного теста

CSV пишем в домашний каталог, потому что `/var/www/app16/nodejs_app` принадлежит `webadmin` и недоступен на запись пользователю `artsiom`.

```bash
cd /var/www/app16/nodejs_app
locust -f locustfile.py \
  --host=https://192.168.2.105 \
  --users=100 --spawn-rate=10 --run-time=1m \
  --headless \
  --csv=$HOME/load_test_results
```

Ключевые флаги:

| Флаг | Значение |
|------|----------|
| `--host=https://192.168.2.105` | Адрес Nginx (совпадает с SAN сертификата) |
| `--users=100` | 100 виртуальных пользователей |
| `--spawn-rate=10` | По 10 пользователей в секунду (разгон 10 с) |
| `--run-time=1m` | Длительность 1 минута |
| `--headless` | Без веб-UI |
| `--csv=$HOME/load_test_results` | Куда сохранять CSV |

### 7.1. Ожидаемый результат

```
Aggregated   2795   0(0.00%)   Avg 13 ms   Med 7 ms   RPS ~47   failures/s 0.00
...
[INFO/locust.main: Shutting down (exit code 0)]
```

- `0.00%` failures — все проверки контента прошли;
- `exit code 0` — тест завершился штатно;
- медиана 7 мс, 95% ≤ 26 мс — стенд работает быстро.

### 7.2. Файлы результатов

```bash
ls -l ~/load_test_results*
column -s, -t < ~/load_test_results_stats.csv | head -20
cat ~/load_test_results_failures.csv   # должен содержать только заголовок
```

---

## 8. Анализ результатов

### 8.1. Распределение по бэкендам

```bash
sudo tail -n 3000 /var/log/nginx/app16.access.log | grep -o 'to: 127.0.0.1:300[01]' | sort | uniq -c
```

Ожидаемо ~50/50. Если перекос — проверьте, что оба сервиса живы.

### 8.2. Среднее время ответа бэкендов

```bash
sudo grep -o 'to: 127.0.0.1:300[01] upstream_time: [0-9.]*' /var/log/nginx/app16.access.log \
  | awk '{sum[$3]+=$5; cnt[$3]++} END {for (p in sum) printf "%s: avg=%.4f s (%d req)\n", p, sum[p]/cnt[p], cnt[p]}'
```

### 8.3. Максимальное время ответа

```bash
sudo grep -o 'upstream_time: [0-9.]*' /var/log/nginx/app16.access.log \
  | awk '{if ($2+0 > max) max=$2+0} END {print "max upstream_time:", max, "s"}'
```

### 8.4. Ошибки 5xx

```bash
sudo awk '$9 ~ /^5/ {print}' /var/log/nginx/app16.access.log | wc -l
```

Должно быть `0`. Если есть — смотрите `sudo tail /var/log/nginx/app16.error.log`.

---

## 9. Типичные проблемы и их решения

| Симптом | Причина | Решение |
|---------|---------|---------|
| `nginx: [emerg] open() "/etc/nginx/sites-enabled/..."` failed | Битая симлинк-ссылка | `sudo ln -sf /etc/nginx/sites-available/app16 /etc/nginx/sites-enabled/app16` |
| `unknown directive "sl_session_cache"` | Опечатка (`sl_` вместо `ssl_`) | Исправить на `ssl_session_cache` |
| `unknown directive "proxy_path"` | Опечатка | Исправить на `proxy_pass` |
| `conflicting server name "_"` | Остался `default` в `sites-enabled` | `sudo rm /etc/nginx/sites-enabled/default` |
| `externally-managed-environment` при `pip install locust` | Защита системного Python | Использовать `pipx install locust` |
| `locust: command not found` | `~/.local/bin` не в `PATH` | `pipx ensurepath && source ~/.bashrc` |
| `PermissionError: ..._stats.csv` | Нет прав записи в каталог приложения | `--csv=$HOME/load_test_results` |
| `unrecognized arguments: --skip-ssl-verify` | Такого флага нет в Locust 2.47 | `self.client.verify = False` в `on_start` |
| `Home page is invalid`, `API notes endpoint failed` | Locust ходит на `localhost`, а `server_name` другой → Nginx отдаёт дефолтную страницу | Тестировать `--host=https://192.168.2.105`, либо добавить `localhost` в `server_name`, либо убрать `default` |
| `Unexpected status: 0` в `POST /add` | SSL-ошибка на POST | `self.client.verify = False` + правильный `--host` |
| В логе нет `to:` | `access_log` без `upstream_log` или `log_format` не в `http { }` | Проверить `grep -n "log_format\|access_log" /etc/nginx/nginx.conf /etc/nginx/sites-available/app16` |
| `3001: 000` | Второй экземпляр не запущен | `sudo journalctl -u app16_3001 -n 50 --no-pager`; проверить `PORT=3001` в юните |
| `EADDRINUSE` в юните 3001 | Порт захардкожен в `server.js` или в юните `PORT=3000` | Исправить `process.env.PORT \|\| 3000` и `Environment=PORT=3001` |
| В браузере «Не защищено» | Самоподписанный сертификат | Это нормально; для теста — добавить `.crt` в доверенные или использовать `mkcert` |
| Браузер не редиректит `localhost` на HTTPS | HSTS-кэш Chrome | `chrome://net-internals/#hsts` → удалить `localhost`; `#sockets` → Flush |

---

## 10. Чек-лист сдачи лабораторной

- [ ] Оба systemd-сервиса `active (running)`, `curl` на `3000` и `3001` → `200`.
- [ ] `curl -I http://192.168.2.105/` → `301`, `Location: https://...`.
- [ ] `curl -k -I https://192.168.2.105/` → `200`.
- [ ] `curl -k -s https://192.168.2.105/api/notes` → JSON.
- [ ] `curl -k -i -X POST https://192.168.2.105/add -d "text=test"` → `302`.
- [ ] `sudo nginx -t` — `syntax is ok`.
- [ ] В логе есть `to: 127.0.0.1:3000` и `to: 127.0.0.1:3001`, распределение ~50/50.
- [ ] `locust ... --host=https://192.168.2.105 ...` → `0.00%` failures, `exit code 0`.
- [ ] `~/load_test_results_stats.csv` — `Failure Count = 0` для всех эндпоинтов.
- [ ] `~/load_test_results_failures.csv` — только заголовок.
- [ ] В логе Nginx нет 5xx (`awk '$9 ~ /^5/' | wc -l` → `0`).

---

## 11. Полезные команды «на каждый день»

```bash
# Логи сервисов
sudo journalctl -u app16_3000 -f
sudo journalctl -u app16_3001 -f

# Лог Nginx в реальном времени с портом бэкенда
sudo tail -f /var/log/nginx/app16.access.log

# Перезапуск после правок
sudo nginx -t && sudo systemctl restart nginx
sudo systemctl restart app16_3000 app16_3001

# Быстрая проверка балансировки
for i in {1..20}; do curl -k -s -o /dev/null https://192.168.2.105/; done
sudo tail -n 40 /var/log/nginx/app16.access.log | grep -o '127.0.0.1:300[01]' | sort | uniq -c

# Повторный нагрузочный тест
cd /var/www/app16/nodejs_app
locust -f locustfile.py --host=https://192.168.2.105 \
  --users=100 --spawn-rate=10 --run-time=1m --headless \
  --csv=$HOME/load_test_results
```

---

## 12. Структура репозитория (что куда положить)

```
.
├── README.md                          # этот файл
├── nginx/
│   ├── app16.conf                     # конфиг сайта (sites-available/app16)
│   └── nginx-log_format.conf          # фрагмент для http { } (log_format)
├── systemd/
│   ├── app16_3000.service
│   └── app16_3001.service
├── locust/
│   └── locustfile.py                  # с добавленным on_start + urllib3
└── scripts/
    ├── gen-cert.sh                    # генерация самоподписанного сертификата с SAN
    └── check-balance.sh               # быстрая проверка round-robin по логу
```

Эта инструкция полностью воспроизводит стенд: два systemd-сервиса → TLS-сертификат с SAN → Nginx с `upstream` и `$upstream_addr` → Locust с отключённой проверкой SSL → анализ распределения и задержек. Все «грабли», возникшие в процессе (опечатки в директивах, `externally-managed-environment`, `PermissionError`, `--skip-ssl-verify`, `localhost` vs `server_name`), вынесены в раздел 9 — при повторном прохождении их можно избежать сразу.
