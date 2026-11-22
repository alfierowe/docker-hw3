# docker hw3

проверки выполнены 22 ноября 2026 года в отдельных тестовых проектах с пустыми volumes. команды ниже приведены для обычного запуска на портах 8080 и 8081; фактические результаты проверки указаны в соответствующих разделах. если volume уже содержит данные, начальное значение счетчика может отличаться от нуля.

## запуск

сначала собираем образ и запускаем контейнер:

```bash
docker build -t counter-hw .
docker run -d --name counter-app -p 8080:8000 -e GREETING=Hello counter-hw
```

проверяем:

```bash
curl http://localhost:8080/
curl -X POST http://localhost:8080/increment
docker logs counter-app
```

при проверке нового контейнера первый GET вернул `count = 0`, POST — `count = 1`, повторный GET — `count = 1`. передача `GREETING` тоже проверена: ответ содержал заданное приветствие `Audit level 4`. сохраненные логи первоначального запуска лежат в [docker-logs.txt](docker-logs.txt).

после проверки останавливаем одиночный контейнер, чтобы освободить порт 8080 для compose:

```bash
docker stop counter-app
```

## compose с sqlite

запускаем:

```bash
HOST_PORT=8081 docker compose -p hw3sqlite -f compose.sqlite.yaml up -d --build
```

база лежит в `/data/counter.db`, для нее подключен volume, чтобы данные не удалялись вместе с контейнером

проверяем:

```bash
curl http://localhost:8081/
for i in 1 2 3; do
  curl -X POST http://localhost:8081/increment
done

docker compose -p hw3sqlite -f compose.sqlite.yaml down
HOST_PORT=8081 docker compose -p hw3sqlite -f compose.sqlite.yaml up -d

curl http://localhost:8081/
```

в тесте с пустым volume начальное значение было `0`, три POST вернули `1`, `2`, `3`. до удаления контейнера счетчик был равен `3`, после `down` и повторного `up` новый контейнер вернул `3`. данные сохранились; `down -v` не использовался.

останавливаем:

```bash
docker compose -p hw3sqlite -f compose.sqlite.yaml down
```

## compose с redis

теперь используем redis вместо sqlite. в `compose.yaml` два сервиса: `app` и `redis`

приложение подключается по адресу `redis://redis:6379/0`

для redis тоже есть volume, данные записываются на диск через AOF. для обоих сервисов настроен healthcheck. снаружи доступен только порт приложения

запуск:

```bash
docker compose up -d --build --wait --wait-timeout 60
docker compose ps
```

### сохранение данных

в тесте с пустым volume начальное значение было `0`. после трех POST получены значения `1`, `2`, `3`:

```bash
curl http://localhost:8080/
for i in 1 2 3; do
  curl -X POST http://localhost:8080/increment
done
```

после этого контейнеры были удалены и созданы заново:

```bash
docker compose down
docker compose up -d --wait --wait-timeout 60
curl http://localhost:8080/
```

счетчик остался равен `3`, то есть данные не пропали

### отключение redis

останавливаем redis:

```bash
docker compose stop redis
```

проверяем, что происходит с приложением:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/healthz
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/readyz
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://localhost:8080/increment
```

получилось:

```text
200
503
503
```

само приложение продолжило работать, но без redis счетчик был недоступен. после пяти неудачных healthcheck статус приложения сменился с `healthy` на `unhealthy`.

### восстановление redis

запускаем redis обратно:

```bash
docker compose start redis
curl --fail --silent --show-error --retry 15 --retry-all-errors --retry-delay 1 --retry-max-time 60 --max-time 3 http://localhost:8080/readyz
docker compose ps
curl http://localhost:8080/
curl -X POST http://localhost:8080/increment
```

после запуска счетчик все еще был `3`, следующий POST увеличил его до `4`

приложение при этом не перезапускалось: идентификатор контейнера и время его запуска остались прежними. оба сервиса снова получили статус `healthy`.

## настройки

в `.env.example` лежат настройки:

```dotenv
HOST_PORT=8080
GREETING=Hello
```

`HOST_PORT` - порт на компьютере

`GREETING` - текст приветствия

чтобы поменять настройки, можно создать `.env`:

```bash
cp .env.example .env
```

запуск:

```bash
docker compose up -d --build --wait --wait-timeout 60
```

остановка:

```bash
docker compose down
```

при обычном `down` данные остаются, а `down -v` удаляет еще и volumes.
