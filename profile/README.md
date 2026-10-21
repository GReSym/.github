# GReSym

Система предназначена для формирования персонализированных рекомендаций видеоигр на основе их текстовых описаний и игровой истории пользователя. В основе рекомендательной части проекта лежит сравнение **embedding-векторов видеоигр**, полученных из их текстовых описаний.

Структура проекта:

```text
GReSym
│
├── backend     - ASP.NET API + парсер Steam (GTK UI)
├── ml          - ML-сервис на Python: векторизация игр, рекомендации
├── deploy      - Docker Compose: MariaDB, RabbitMQ, Milvus, API, ML
├── Frontend    - Vue (планируется)
└── Mobile App  - Android/Kotlin (планируется)
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
```

## Развёртывание с нуля

Ниже — полный запуск на Linux через Docker Compose. Все порты публикуются только на `127.0.0.1`.

### 1. Что нужно установить

- Docker Engine с плагином Compose; пользователь должен состоять в группе `docker`;
- `git`, `jq`, `openssl`, `zstd`;
- `git-lfs` или [uv](https://docs.astral.sh/uv/), чтобы скачать модель;
- для разработки без контейнеров и для миграций: .NET 10 SDK + `dotnet-ef`, uv, клиент MariaDB.

```bash
sudo usermod -aG docker $USER        # затем перелогиниться
sudo systemctl enable --now docker
docker context use default
```

На портах 3306, 5672, 15672, 19530, 9091, 5147 и 8000 не должно быть других служб, например локальной MariaDB или RabbitMQ.

### 2. Репозитории

Все три репозитория должны лежать в одном каталоге: `deploy` обращается к `../backend` и `../ml`.

```bash
mkdir GReSym && cd GReSym
git clone https://github.com/GReSym/backend.git
git clone https://github.com/GReSym/ml.git
git clone https://github.com/GReSym/deploy.git
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

Модель Qwen3-Embedding-0.6B занимает около 2,3 ГБ. В образ она не входит, а монтируется в контейнер.

```bash
cd ml
uv run hf download Qwen/Qwen3-Embedding-0.6B --local-dir Qwen3-Embedding
# или: git lfs install && git clone https://huggingface.co/Qwen/Qwen3-Embedding-0.6B Qwen3-Embedding
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

Если дампа не было или код новее дампа, примените миграции. Нужны .NET SDK и `dotnet-ef`:

```bash
cd ../backend && ./manage.sh database update
```

### 8. Векторы игр

Векторизация идёт на CPU, около 0,6 игры в секунду. Например, первые 100 игр:

```bash
cd ../deploy
docker compose exec ml-worker ml vectorize --limit 100
```

### Адреса

| Сервис | Адрес |
|--------|-------|
| API | http://localhost:5147 |
| ML API (Swagger) | http://localhost:8000/docs |
| RabbitMQ Management | http://localhost:15672 |
| Milvus | `localhost:19530`, проверка: `curl localhost:9091/healthz` |
| MariaDB | `localhost:3306` |

Swagger API доступен только при запуске без контейнера (`./manage.sh run api`, окружение Development).

### Резервные копии

```bash
cd deploy
./manage.sh backup all                             # MariaDB (.sql.zst) + векторы Milvus (.npz) → backups/<дата>/
./manage.sh restore mariadb backups/<дата>/mariadb-GamesRecommend.sql.zst
./manage.sh restore milvus backups/<дата>/milvus.npz
./manage.sh milvus-snapshot                        # точная копия data/milvus (для той же версии Milvus)
```

Данные контейнеров хранятся в `deploy/data/`. `./manage.sh down` останавливает контейнеры, данные при этом сохраняются.
