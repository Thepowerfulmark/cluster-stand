# Стенд в Kubernetes

Шаблон стенда: Postgres, Redis, RabbitMQ, одно приложение и фронт. Свой сервис подключается заменой образа `app-placeholder` в `kustomization.yaml`. Фронт заменяется содержимым ConfigMap `frontend`. Остальные манифесты остаются.

Паролей в репозитории нет. Их создают секретами на кластере.

Сервисы доступны только внутри namespace `stand`. Наружу порты не публикуются.

## CI

Push и pull request в `main` прогоняют `kustomize build`. Кластер для этой проверки не нужен.

## Состав

| Имя | Зачем |
|---|---|
| `postgres` | база, данные на диске 1 ГБ |
| `redis` | кэш, данные на диске 1 ГБ |
| `rabbitmq` | очередь, порт 5672 |
| `app` | ваш сервис, сейчас заглушка на порту 8080 |
| `frontend` | статика, порт 8080 |
| `migrate` | один раз создаёт базу и таблицу `app_meta` |

Приложение видит такие переменные: `DATABASE_HOST`, `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD`, `REDIS_HOST`, `RABBITMQ_HOST`.

## Запуск

Подставьте свои значения вместо `change-me`.

```bash
kubectl apply -f namespace.yaml
kubectl -n stand create secret generic db-auth \
  --from-literal=username=app \
  --from-literal=password=change-me \
  --from-literal=database=app
kubectl -n stand create secret generic broker-auth \
  --from-literal=username=app \
  --from-literal=password=change-me
kubectl apply -k .
kubectl -n stand get pods
```

Повторный `kubectl apply -k .` не меняет уже созданные секреты.

Проверка миграции:

```bash
kubectl -n stand exec deploy/postgres -- \
  sh -c 'PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT note FROM app_meta"'
```

Команда берёт пароль из переменных уже запущенного пода. В репозиторий он не записывается.

## Свой проект

В `kustomization.yaml` замените образ:

```yaml
images:
  - name: app-placeholder
    newName: registry.example.com/your-app
    newTag: "1.0.0"
```

Если процесс слушает не 8080 или ему не нужен конфиг nginx из ConfigMap `app-placeholder`, поправьте контейнер `app` в `app.yaml`: уберите тома nginx и укажите свой `command`.

Фронт: замените ключ `index.html` в ConfigMap `frontend` либо смените образ в `frontend.yaml` на свой.
