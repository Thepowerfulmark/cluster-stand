# Установка стенда

Документ описывает установку одноузлового k3s на виртуальной машине и стенда в namespace `stand`: Postgres, Redis, RabbitMQ, приложение и фронт.

Секреты в документе не приводятся. Значения `change-me` нужно заменить значениями из защищённого хранилища.

Все команды выполняются от `root`.

Разработка и тестирование проводились на одноузловом k3s `v1.37.0+k3s1` и Headlamp `v0.45.0`. Повторный прогон установщика stable-канала поставил k3s `v1.36.4+k3s1` на Ubuntu 22.04. Другие дистрибутивы этим документом отдельно не покрываются.

Образы стенда: `postgres:16-alpine`, `redis:7-alpine`, `rabbitmq:3.13-alpine`.

## 1. Состав

| Имя | Назначение |
|-----|------------|
| `postgres` | база, том 1 ГБ |
| `redis` | кэш, том 1 ГБ |
| `rabbitmq` | очередь, порт 5672 |
| `app` | сервис, сейчас заглушка на порту 8080 |
| `frontend` | статика, порт 8080 |
| `migrate` | один раз создаёт базу и таблицу `app_meta` |

Сервисы доступны только внутри namespace `stand`. Наружу порты стенда не публикуются.

Приложение читает переменные `DATABASE_HOST`, `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD`, `REDIS_HOST`, `RABBITMQ_HOST`. Имена и ключи секретов заданы в `app.yaml`, `postgres.yaml` и `rabbitmq.yaml`.

## 2. Требования к хосту

- Одна виртуальная машина, архитектура `amd64`.
- Установлен `curl`.
- Свободное место не меньше объёма томов: Postgres и Redis запрашивают по 1 ГБ, плюс место под образы.
- Исходящий доступ в интернет на время установки k3s и загрузки образов.

k3s ставит свой Traefik. Ingress Headlamp использует этот класс.

## 3. Кластер

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

Узел в колонке `STATUS` должен быть `Ready`.

Kubeconfig на машине: `/etc/rancher/k3s/k3s.yaml`.

Дальше в этом документе команды идут через `k3s kubectl`.

## 4. Headlamp

Это учебный стенд. Файл `headlamp.yaml` привязывает ServiceAccount `headlamp` к роли `cluster-admin`. Для продуктива эту роль меняют на `view`, а срок токена делают коротким. Годовой токен с `cluster-admin` для продуктива не подходит.

Применить манифест из этого репозитория:

```bash
k3s kubectl apply -f headlamp.yaml
k3s kubectl -n kube-system rollout status deploy/headlamp
```

Токен для входа в морду. Срок в примере — учебный, один год:

```bash
k3s kubectl -n kube-system create token headlamp --duration=8760h
```

Для продуктива, после замены роли на `view`:

```bash
k3s kubectl -n kube-system create token headlamp --duration=1h
```

Команда печатает токен в терминал. В репозиторий и в этот документ его не записывают.

Открыть в браузере `http://<адрес-вм>/`. На странице `/c/main/token` вставить токен в поле идентификационного токена и нажать «Аутентификация».

## 5. Секреты

Пароли создаются на кластере. Повторный `kubectl apply` их не меняет.

```bash
k3s kubectl apply -f namespace.yaml
k3s kubectl -n stand create secret generic db-auth \
  --from-literal=username=app \
  --from-literal=password=change-me \
  --from-literal=database=app
k3s kubectl -n stand create secret generic broker-auth \
  --from-literal=username=app \
  --from-literal=password=change-me
```

`db-auth` читают Postgres, миграция и приложение. Ключи: `username`, `password`, `database`.

`broker-auth` читает RabbitMQ. Ключи: `username`, `password`.

## 6. Установка стенда

Из каталога этого репозитория:

```bash
k3s kubectl apply -k .
k3s kubectl -n stand get pods
```

Дождаться `Running` у `postgres`, `redis`, `rabbitmq`, `app`, `frontend`. Job `migrate` должен перейти в `Completed`.

Повторная установка:

```bash
k3s kubectl apply -k .
```

Уже созданные секреты эта команда не перезаписывает.

Снять стенд:

```bash
k3s kubectl delete -k .
k3s kubectl delete namespace stand
```

Вторая команда удаляет и секреты, и тома namespace `stand`.

## 7. Свой сервис

В `kustomization.yaml` заменить образ:

```yaml
images:
  - name: app-placeholder
    newName: registry.example.com/your-app
    newTag: "1.0.0"
```

Если процесс слушает не 8080, поправить контейнер `app` в `app.yaml`.

Фронт: заменить ключ `index.html` в ConfigMap `frontend` либо сменить образ в `frontend.yaml`.

## 8. Проверка

Открыть Headlamp и убедиться, что поды namespace `stand` в состоянии Running, job `migrate` — Completed.

Строка миграции:

```bash
k3s kubectl -n stand exec deploy/postgres -- \
  sh -c 'PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT note FROM app_meta"'
```

Ожидаемое значение `note` — `ready`. Пароль команда берёт из переменных уже запущенного пода.

## 9. Чеклист

- Узел `k3s kubectl get nodes` в статусе `Ready`.
- В браузере открывается Headlamp, вход по токену проходит.
- Поды `stand` в состоянии Running, `migrate` — Completed.
- `SELECT note FROM app_meta` возвращает `ready`.
