# Jellyfin + qBittorrent на VM node1

Пошаговое описание всего, что было сделано.  
Сервер уже настроен — этот файл для понимания и повторения вручную.

---

## Что получилось в итоге

Два Docker-контейнера на VM **node1** (Ubuntu 22.04, IP `192.168.1.123`):

| Сервис      | Назначение              | Ссылка                              |
|-------------|-------------------------|-------------------------------------|
| Jellyfin    | Просмотр фильмов        | http://192.168.1.123:8096           |
| qBittorrent | Скачивание торрентов    | http://192.168.1.123:8080           |

**Логин qBittorrent:** `admin` / `admin123`

Через Tailscale (если нужен доступ с телефона):

| Сервис      | Ссылка                    |
|-------------|---------------------------|
| Jellyfin    | http://100.66.67.5:8096   |
| qBittorrent | http://100.66.67.5:8080   |

---

## Как это работает (схема)

```
Ты добавляешь торрент в qBittorrent
              ↓
Файл скачивается в /downloads/movies (внутри контейнера)
              ↓
На VM это та же папка: /home/vagrant/jellyfin/media/movies
              ↓
Jellyfin видит её как /media/movies и показывает фильм
```

Оба контейнера используют **одну общую папку** — поэтому скачанный фильм сразу появляется в Jellyfin.

---

## Шаг 1. Структура папок на VM

Созданы папки для конфигов и медиа:

```
/home/vagrant/
├── jellyfin/
│   ├── config/          ← настройки Jellyfin (сохраняются между перезапусками)
│   └── media/
│       └── movies/      ← СЮДА попадают скачанные фильмы
└── qbittorrent/
    └── config/          ← настройки qBittorrent
```

**Зачем:** конфиги отдельно от данных, фильмы в одном месте для обоих сервисов.

---

## Шаг 2. Docker Compose — описание контейнеров

Файл: `docker/docker-compose.yml`

### Jellyfin

- **Образ:** `jellyfin/jellyfin:latest`
- **Порт:** `8096` (веб-интерфейс)
- **Папки:**
  - `jellyfin/config` → `/config` (настройки)
  - `jellyfin/media` → `/media` (фильмы)
- **Автоперезапуск:** `restart: unless-stopped`

### qBittorrent

- **Образ:** `lscr.io/linuxserver/qbittorrent:latest`
- **Порт:** `8080` (веб-интерфейс), `6881` (торрент-трафик)
- **Переменные:**
  - `PUID=1000`, `PGID=1000` — права на файлы от пользователя vagrant
  - `WEBUI_PORT=8080` — порт веб-интерфейса
  - `WEBUI_PASSWORD=admin123` — пароль при первом запуске
- **Папки:**
  - `qbittorrent/config` → `/config` (настройки)
  - `jellyfin/media` → `/downloads` (общая папка с Jellyfin!)
- **Автоперезапуск:** `restart: unless-stopped`

**Ключевой момент:** строка  
`jellyfin/media:/downloads` — qBittorrent пишет в ту же папку, откуда Jellyfin читает.

---

## Шаг 3. Запуск контейнеров

На VM выполняется:

```bash
cd /home/vagrant/docker
sudo docker-compose up -d
```

- `up` — создать и запустить контейнеры
- `-d` — в фоне (detached)

Проверка:

```bash
sudo docker ps
```

Должны быть `jellyfin` и `qbittorrent` со статусом `Up`.

---

## Шаг 4. Настройка qBittorrent

**Проблема:** при первом запуске qBittorrent даёт **временный** пароль в логах Docker. После перезапуска он меняется — войти нельзя.

Пароль уже задан в `docker-compose.yml` через `WEBUI_PASSWORD=admin123`.

Если нужно настроить вручную через API:

```bash
# Войти (если пароль временный — смотри: sudo docker logs qbittorrent)
curl -c /tmp/qb.cookie "http://localhost:8080/api/v2/auth/login" \
  -d "username=admin&password=admin123"

# Указать папку для фильмов
curl -b /tmp/qb.cookie "http://localhost:8080/api/v2/app/setPreferences" \
  -d 'json={"save_path":"/downloads/movies","web_ui_address":"*"}'
```

---

## Шаг 5. Автозапуск при включении VM

Три уровня защиты, чтобы контейнеры поднимались после перезагрузки:

| Уровень | Что делает |
|---------|------------|
| 1 | `systemctl enable docker` — Docker стартует при загрузке VM |
| 2 | `restart: unless-stopped` в docker-compose — контейнеры перезапускаются сами |
| 3 | systemd-сервис `media-stack.service` — явно запускает `docker-compose up -d` после загрузки |

Создание systemd-сервиса на VM:

```bash
sudo tee /etc/systemd/system/media-stack.service > /dev/null <<'EOF'
[Unit]
Description=Jellyfin and qBittorrent Docker containers
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/vagrant/docker
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable media-stack.service
```

---

## Шаг 6. Полная установка с нуля (все команды)

```bash
# 1. Docker
sudo apt-get update -y && sudo apt-get install -y docker.io docker-compose
sudo systemctl enable --now docker

# 2. Папки
mkdir -p /home/vagrant/jellyfin/{config,media/movies}
mkdir -p /home/vagrant/qbittorrent/config
mkdir -p /home/vagrant/docker

# 3. Скопировать docker-compose.yml и запустить
cp /vagrant/docker/docker-compose.yml /home/vagrant/docker/
cd /home/vagrant/docker && sudo docker-compose up -d
```

---

## Шаг 7. Первичная настройка Jellyfin (вручную в браузере)

1. Открой http://192.168.1.123:8096
2. Создай пользователя (логин и пароль на свой выбор)
3. **Добавить медиатеку** → тип **Фильмы**
4. Укажи папку: **`/media/movies`**
5. Сохрани

После этого Jellyfin будет сканировать папку, куда qBittorrent качает фильмы.

---

## Шаг 8. Как пользоваться

1. Открой http://192.168.1.123:8080
2. Войди: `admin` / `admin123`
3. Добавь торрент (файл `.torrent` или магнет-ссылку)
4. Дождись окончания скачивания
5. Открой Jellyfin — фильм появится в библиотеке (или нажми «Сканировать библиотеку»)

---

## Полезные команды на VM

```bash
# Статус контейнеров
sudo docker ps

# Перезапуск
cd /home/vagrant/docker && sudo docker-compose restart

# Остановка
cd /home/vagrant/docker && sudo docker-compose down

# Запуск снова
cd /home/vagrant/docker && sudo docker-compose up -d

# Логи qBittorrent
sudo docker logs qbittorrent

# Логи Jellyfin
sudo docker logs jellyfin

# Проверить автозапуск
systemctl is-enabled docker media-stack
```

---

## Файлы в проекте

| Файл | Назначение |
|------|------------|
| `docker/docker-compose.yml` | Описание контейнеров Jellyfin и qBittorrent |
| `MEDIA-SETUP.md` | Эта инструкция |

---

## Проверка доступа с Windows

С хоста (PowerShell):

```powershell
curl.exe -s -o NUL -w "qb:%{http_code}\n" http://192.168.1.123:8080
curl.exe -s -o NUL -w "jf:%{http_code}\n" http://192.168.1.123:8096
```

Ожидаемый ответ: `qb:200` и `jf:302` — оба сервиса доступны.
