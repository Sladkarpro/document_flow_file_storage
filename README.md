# Файловое хранилище (MinIO + imgproxy + Nginx)

Инструкция по развёртыванию S3-хранилища файлов и сервиса обработки изображений для `go_kanban_service`.

Стек состоит из:
- `minio` — S3-хранилище файлов;
- `imgproxy` — ресайз изображений на лету;
- `nginx_imgproxy` — nginx-кэш перед imgproxy.

Обычный поток для превью:
```text
браузер -> основной nginx (/media/preview/) -> nginx_imgproxy -> imgproxy -> MinIO
```

`nginx_imgproxy` наружу не публикуется. Он доступен внутри общей Docker-сети `project-net`, а внешний маршрут `/media/preview/` проксирует основной nginx проекта.

---

## 1. Подготовка конфигурации

Скопируйте файл с примером переменных окружения:
```bash
cp .env.example .env
```

В созданном файле `.env` **обязательно** измените стандартные доступы:
- `MINIO_ROOT_USER` — ваш логин администратора
- `MINIO_ROOT_PASSWORD` — ваш надёжный пароль

**(Только для продакшена)** Сгенерируйте секретные ключи для безопасного ресайза картинок:
```bash
openssl rand -hex 32
```
Впишите два сгенерированных значения в `.env` в параметры `IMGPROXY_KEY` и `IMGPROXY_SALT`. Установите параметр `IMGPROXY_ALLOW_UNSAFE_URL=0`.

## 2. Создание Docker-сети

Хранилище общается по внешней (для него) сети `project-net`. Если вы разворачиваете хранилище автономно, или основной проект (Symfony) ещё не был запущен и не успел создать эту сеть, вам нужно создать её вручную, иначе при запуске Docker выдаст ошибку:
```bash
docker network create project-net
```

## 3. Запуск хранилища

Запустите все сервисы в фоновом режиме:
```bash
docker compose up -d
```

После запуска `go_kanban_service` должен быть в той же Docker-сети `project-net` и использовать:
```env
MINIO_ENDPOINT=minio:9000
MINIO_BUCKET_NAME=kanban
IMGPROXY_BASE_URL=/media/preview
```

## 4. Создание бакета (Обязательный шаг)

При первом запуске сервер MinIO абсолютно пустой. Вам нужно создать стартовый бакет (папку для файлов), например, `kanban`.

Выполните команду ниже (не забудьте подставить логин и пароль, которые вы придумали в шаге 1, а также имя бакета вместо `kanban`, если оно отличается):

```bash
docker run --rm --network project-net --entrypoint sh \
  minio/mc:latest -c \
  "mc alias set local http://minio:9000 ВАШ_ЛОГИН ВАШ_ПАРОЛЬ && mc mb -p local/kanban"
```

## 5. Проверка работы

Проверьте, что все контейнеры запущены и работают без ошибок:
```bash
docker compose ps
```

Проверьте логи или статус кэширующего сервера:
```bash
docker compose logs nginx_imgproxy
```

Если запущен основной nginx проекта, healthcheck кэша доступен через него:
```bash
curl -I http://localhost/media/preview/healthz
```

Для управления хранилищем через интерфейс используйте веб-админку MinIO, которая доступна на порту **9001** (доступна только локально, для удалённого доступа потребуется SSH-туннель).

---

## Где хранятся данные?

По умолчанию данные хранятся на уровень выше директории проекта (чтобы избежать случайного удаления вместе с проектом):
- `../minio_data/` — физические файлы бакетов
- `../imgproxy_cache/` — кэшированные миниатюры изображений

Например, при расположении проекта в `/home/dev/don_stroy_mash/file_storage` фактические пути будут:
- `/home/dev/don_stroy_mash/minio_data`
- `/home/dev/don_stroy_mash/imgproxy_cache`

## Автономная отладка без основного nginx

По умолчанию порт `nginx_imgproxy` наружу закрыт, потому что превью должны идти через основной nginx проекта. Если нужно проверить imgproxy-кэш без основного nginx, временно раскомментируйте проброс порта в `docker-compose.yml`:

```yaml
ports:
  - "${IMGPROXY_CACHE_PORT:-8082}:80"
```

После этого healthcheck будет доступен напрямую:
```bash
curl -I http://localhost:8082/healthz
```

Для обычного запуска с `go_kanban_service` это не требуется.

---

## 6. Миграция старых файлов (Разовая операция)

Если вам нужно перенести старые вложения с сервера в MinIO, используйте эту команду (путь `/home/.../uploads/kanban` замените на актуальный путь к старой папке на сервере):

```bash
docker run --rm --network project-net \
  -v /полный/путь/к/старым/uploads/kanban:/legacy_kanban:ro \
  --entrypoint sh \
  minio/mc:latest -c \
  "mc alias set local http://minio:9000 ВАШ_ЛОГИН ВАШ_ПАРОЛЬ && mc cp --recursive /legacy_kanban/ local/kanban"
```
