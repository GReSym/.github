# GReSym

Система предназначена для формирования персонализированных рекомендаций видеоигр на основе их текстовых описаний и игровой истории пользователя. В основе рекомендательной части проекта лежит сравнение **embedding-векторов видеоигр**, полученных из их текстовых описаний.

Структура проекта:

```text
GReSym
│
├── backend      - ASP.NET API + парсер Steam (GTK UI)
├── ml           - ML-сервис на Python: векторизация игр, рекомендации
├── deploy       - Docker Compose: MariaDB, RabbitMQ, Milvus, Openinary (картинки), API, ML
├── web-frontend - веб-клиент на Vue 3
└── Mobile App   - Android/Kotlin (планируется)
```

Общая схема взаимодействия компонентов проекта:

```text
                   ┌─────────────────┐
                   │   Steam / Data  │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ GReSym.Parser   │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │    Database     │
                   └────────┬────────┘
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
      ┌─────────────┐               ┌─────────────┐
      │ GReSym.API  │               │ ML Service  │
      └──────┬──────┘               └──────┬──────┘
             │                             │
             └──────────────┬──────────────┘
                            ▼
                     Recommendations
                            │
                            ▼
                   ┌─────────────────┐       ┌─────────────────────┐
                   │  web-frontend   │◀──────│ Openinary / Steam   │
                   └─────────────────┘       │ (картинки игр)      │
                                             └─────────────────────┘
```

## Развёртывание с нуля

Ниже — полный запуск на Linux через Docker Compose. Все порты публикуются только на `127.0.0.1`.

### 1. Что нужно установить

- Docker Engine с плагином Compose; пользователь должен состоять в группе `docker`;
- `git`, `jq`, `openssl`, `zstd`;
- `git-lfs` или [uv](https://docs.astral.sh/uv/), чтобы скачать модель;
- для разработки без контейнеров и для миграций: .NET 10 SDK + `dotnet-ef`, uv, клиент MariaDB;
- для веб-клиента: Node.js 22.18+ или 24.12+.

```bash
sudo usermod -aG docker $USER        # затем перелогиниться
sudo systemctl enable --now docker
docker context use default
```

На портах 3306, 5672, 15672, 19530, 9091, 3000, 8090, 5147 и 8000 не должно быть других служб, например локальной MariaDB или RabbitMQ.

### 2. Репозитории

Все репозитории должны лежать в одном каталоге: `deploy` обращается к `../backend` и `../ml`.

```bash
mkdir GReSym && cd GReSym
git clone https://github.com/GReSym/backend.git
git clone https://github.com/GReSym/ml.git
git clone https://github.com/GReSym/deploy.git
git clone https://github.com/GReSym/web-frontend.git
```

### 3. Конфигурация

`backend/config.json` хранит пользователя БД приложения, пароль RabbitMQ и JWT-ключ:

```bash
cd backend
./manage.sh setup          # первый запуск создаёт config.json из примера и останавливается
nano config.json           # заполнить user, password, rabbitmq_user, rabbitmq_password (server оставить localhost)
./manage.sh setup          # генерирует JWT-ключ и appsettings.json
cd ..
```

`ml/config.json` хранит пользователя БД только для чтения. Пароль RabbitMQ должен совпадать с `backend/config.json`:

```bash
cat > ml/config.json <<'JSON'
{
  "database": { "user": "readonly_user", "password": "придумайте-пароль" },
  "rabbitmq": { "user": "admin", "password": "пароль-из-backend/config.json" }
}
JSON
```

Пользователи MariaDB и RabbitMQ создаются из этих файлов **при первом запуске** контейнеров. Смена паролей потом на них не влияет.

### 4. Модель эмбеддингов

По умолчанию (`model.backend: onnx`) модель работает через ONNX Runtime на CPU. Её нужно скачать в каталог `ml/Qwen3-Embedding-onnx/`:

```bash
cd ml
uv run hf download Qdrant/Qwen3-Embedding-0.6B-onnx --include "onnx/model.onnx*" --include "*.json" --local-dir Qwen3-Embedding-onnx
cd ..
```

Для работы на GPU (`model.backend: torch`) нужна исходная модель в каталоге `ml/Qwen3-Embedding/`:

```bash
cd ml
uv run hf download Qwen/Qwen3-Embedding-0.6B --local-dir Qwen3-Embedding
cd ..
```

### 5. Данные (необязательно)

Если есть дамп базы (`./manage.sh database dump` в backend), положите его в `deploy/mariadb/seed/`. При первом запуске он импортируется автоматически; дамп на 150 МБ загружается около 10 минут.

```bash
cp ~/GamesRecommend-*.sql.zst deploy/mariadb/seed/
```

### 6. Запуск

```bash
cd deploy
./manage.sh setup          # .env из backend/config.json и ml/config.json (+ пароль root MariaDB)
./manage.sh up --app       # сборка образов и запуск всего стека; ждёт, пока сервисы станут healthy
./manage.sh ps
```

Без `--app` запускаются только MariaDB, RabbitMQ и Milvus. API и ML тогда можно запускать без контейнеров:
`./manage.sh run api` в backend, `uv run ml api` и `uv run ml worker` в ml.

### 7. Схема БД

Дамп уже содержит схему и таблицу `__EFMigrationsHistory`. Миграции нужны в двух случаях:

- **дампа не было**: контейнер MariaDB создаёт пустую базу `GamesRecommend` без таблиц, и API без них не работает;
- **код новее дампа**: в backend есть миграции, которых нет в `__EFMigrationsHistory`.

Миграции не встроены в Docker и применяются с хоста. Для этого нужны .NET 10 SDK и инструмент `dotnet-ef`:

```bash
dotnet tool install --global dotnet-ef   # один раз; manage.sh находит его и в ~/.dotnet/tools
cd ../backend && ./manage.sh database update
```

`database update` подключается к контейнеру через `localhost:3306`, используя `backend/GReSym.Infrastructure/appsettings.json`, который генерирует `./manage.sh setup`.
Пользователь приложения создаётся с полными правами на базу, поэтому миграциям этого достаточно. Повторный запуск безопасен: уже применённые миграции пропускаются.

### 8. Картинки (Openinary)

Картинки игр постепенно переносятся с CDN Steam в Openinary (сервис `openinary` в deploy, версия закреплена).
Пока картинка не перенесена, клиент берёт её со Steam, так что этот шаг можно отложить.

1. Откройте http://localhost:3000, создайте учётную запись администратора на странице `/setup`,
   затем в меню профиля → **API Keys** создайте ключ.
2. Впишите ключ в `backend/config.json` как `"openinary_api_key"` и обновите конфигурацию:

   ```bash
   cd backend && ./manage.sh setup && cd ../deploy && ./manage.sh setup && ./manage.sh up --app
   ./manage.sh openinary-unlimit-keys   # иначе Openinary пропускает не больше 100 запросов в минуту на ключ
   ```

3. В веб-клиенте под учётной записью администратора (страница администратора → «Картинки») задайте базовый URL медиа
   (`http://localhost:8090` или `/cdn`, см. README web-frontend) и запустите перенос: сначала обложки,
   скриншоты — выборочно (все скриншоты каталога занимают около 300 ГБ).

Наружу публикуется только шлюз `media` (порт 8090): он пропускает лишь заданные размеры картинок.

### 9. Веб-клиент

```bash
cd web-frontend
npm install --legacy-peer-deps
npm run dev                # http://localhost:5173, запросы /api и /cdn проксируются на API и шлюз картинок
```

Для демо: `npm run build && npm run preview -- --host` (порт 4173); через туннель достаточно открыть только этот порт,
если базовый URL медиа равен `/cdn`.

### 10. Векторы игр

Векторизация идёт на CPU, около 0,6 игры в секунду. Например, первые 100 игр:

```bash
cd ../deploy
docker compose exec ml-worker ml vectorize --limit 100
```

### GPU (необязательно)

По умолчанию ML-сервис работает на CPU. Для видеокарты AMD (ROCm) или NVIDIA (CUDA):

```bash
cd deploy
TORCH_BACKEND=rocm ./manage.sh setup    # или TORCH_BACKEND=cuda; вернуть CPU: TORCH_BACKEND=cpu
./manage.sh up --app                    # пересобирает образ ML с PyTorch для выбранной видеокарты
```

- ROCm: на хосте нужен драйвер amdgpu (`/dev/kfd`, `/dev/dri`). Сама библиотека ROCm приходит вместе с PyTorch.
- CUDA: нужны драйвер NVIDIA и [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

Подробности (тип данных модели, `HSA_OVERRIDE_GFX_VERSION`, запуск без Docker) — в README репозиториев `deploy` и `ml`.

### Адреса

| Сервис | Адрес |
|--------|-------|
| Веб-клиент | http://localhost:5173 (`npm run dev`) |
| API | http://localhost:5147 |
| Openinary (дашборд, только для администратора) | http://localhost:3000 |
| Шлюз картинок (публичный адрес медиа) | http://localhost:8090 |
| ML API (Swagger) | http://localhost:8000/docs |
| RabbitMQ Management | http://localhost:15672 |
| Milvus | `localhost:19530`, проверка: `curl localhost:9091/healthz` |
| MariaDB | `localhost:3306` |

Swagger UI для API включён только в окружении `Development`. При запуске без контейнера (`./manage.sh run api`) оно используется по умолчанию:
http://localhost:5147/swagger. Контейнер `api` работает в `Production`, поэтому Swagger в нём выключен.
Если Swagger нужен и в Docker, создайте `deploy/compose.override.yaml`. Docker Compose подхватывает этот файл автоматически, а в git он не попадает:

```yaml
services:
  api:
    environment:
      ASPNETCORE_ENVIRONMENT: Development
```

Затем выполните `./manage.sh up --app`, и Swagger будет доступен на http://localhost:5147/swagger. Для общедоступного развёртывания так делать не стоит:
Development показывает подробные ошибки.

### Резервные копии

```bash
cd deploy
./manage.sh backup all                             # MariaDB (.sql.zst) + векторы Milvus (.npz) + Openinary (.tar) → backups/<дата>/
./manage.sh restore mariadb backups/<дата>/mariadb-GamesRecommend.sql.zst
./manage.sh restore milvus backups/<дата>/milvus.npz
./manage.sh restore openinary backups/<дата>/openinary.tar   # вместе с дампом MariaDB: в БД хранятся пути к картинкам
./manage.sh milvus-snapshot                        # точная копия data/milvus (для той же версии Milvus)
```

Данные контейнеров хранятся в `deploy/data/`. `./manage.sh down` останавливает контейнеры, данные при этом сохраняются.
