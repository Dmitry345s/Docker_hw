Все команды выполняются из корня репозитория.
Для запуска надо скопировать файл .env.example в .env командой:
cp .env.example .env

В файле .env можно поменять значение HOST_PORT - это порт на компьютере, через который будет доступно приложение.
Переменная GREETING задаёт приветственное сообщение приложения.

Далее надо собрать образ приложения и запустить контейнеры приложения и redis в фоновом режиме:
sudo docker compose up -d --build

Готовый образ redis скачается автоматически при необходимости.

Проверки, которые я делал:
После запуска проверяем, что контейнеры обоих сервисов имеют статус healthy:
Команда: sudo docker compose ps
Вывод:
NAME                IMAGE              COMMAND                  SERVICE   CREATED         STATUS                   PORTS
docker_hw-app-1     final-image        "python app.py"          app       2 minutes ago   Up 2 minutes (healthy)   0.0.0.0:8080->8000/tcp, [::]:8080->8000/tcp
docker_hw-redis-1   redis:7.4-alpine   "docker-entrypoint.s…"   redis     2 minutes ago   Up 2 minutes (healthy)   6379/tcp
Видно, что контейнеры обоих сервисов имеют статус healthy: их проверки готовности проходят успешно.

Теперь проверяем базовую работу приложения. Выполняем команды:
curl -fsS http://127.0.0.1:8080/
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS http://127.0.0.1:8080/
curl -fsS http://127.0.0.1:8080/readyz

Первая команда выводит:
{"greeting": "Hello, Dmitry's Docker!", "count": 0, "hostname": "fefd717b8f78", "storage": "redis"}
Видим, что greeting совпадает со значением GREETING в .env (настройки скопированы из .env.example).
В начале этой проверки значение count равно 0.
storage=redis показывает, что приложение использует redis как хранилище.

Предпоследняя команда выводит:
{"greeting": "Hello, Dmitry's Docker!", "count": 2, "hostname": "fefd717b8f78", "storage": "redis"}
После двух POST-запросов count стал 2. GET-запросы count не изменяют. Значит всё работает корректно.

Последняя:
{"status": "ready", "storage": "redis"}
Ответ со status - ready показывает, что приложение может обратиться к хранилищу redis.

Теперь буду проверять сохранение значения count после пересоздания контейнеров приложения и redis.
Выполнил команды:
sudo docker compose down
sudo docker compose up -d
здесь надо дождаться статуса healthy у обоих сервисов. Его можно проверять при помощи команды: sudo docker compose ps
curl -fsS http://127.0.0.1:8080/

Первая команда останавливает и удаляет контейнеры приложения и redis, сохраняя именованный том. Вторая создаёт и запускает контейнеры заново.
Последняя команда вывела:
{"greeting": "Hello, Dmitry's Docker!", "count": 2, "hostname": "44697d051e68", "storage": "redis"}
Значение count сохранилось после пересоздания контейнеров оно по-прежнему равно 2.

Теперь посмотрим на то, что будет происходить при остановке redis и последующем перезапуске.
Команды:
sudo docker compose stop redis
(остановили redis)

curl -fsS http://127.0.0.1:8080/healthz
Возвращает {"status": "alive"}

curl -fsS http://127.0.0.1:8080/readyz
Возвращает curl: (22) The requested URL returned error: 503

curl -fsS -X POST http://127.0.0.1:8080/increment
Возвращает curl: (22) The requested URL returned error: 503

Таким образом видим что при остановке redis приложение продолжает работать но команды связанные с redis начинают возвращать предполагаемый код ошибки 503.

Теперь восстановим работу redis.
Команды:
sudo docker compose start redis
(запустили redis)

curl -fsS http://127.0.0.1:8080/
{"greeting": "Hello, Dmitry's Docker!", "count": 2, "hostname": "44697d051e68", "storage": "redis"}
count остался таким каким был до остановки redis. Значит во-первых данные не были потеряны, а во-вторых increment во время выключенного redis не применился.

curl -fsS -X POST http://127.0.0.1:8080/increment
{"greeting": "Hello, Dmitry's Docker!", "count": 3, "hostname": "44697d051e68", "storage": "redis"}
count стал 3. Приложение работает в штатном режиме после восстановления работы redis.

Также прикрепляю логи полученные командой: sudo docker compose logs app

app-1  | 2026-11-22 17:24:50,147 listening port=8000 storage=redis
app-1  | 2026-11-22 17:24:55,198 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:00,396 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:05,614 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:10,823 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:16,032 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:21,142 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:26,339 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:31,560 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:36,731 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:41,912 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:45,737 http 172.18.0.1 "GET / HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:47,112 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:52,338 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:25:57,470 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:02,655 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:07,867 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:13,085 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:18,298 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:23,492 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:28,681 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:33,880 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:38,988 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:44,184 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:49,313 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:54,518 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:26:59,742 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:04,939 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:10,150 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:15,344 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:20,550 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:25,766 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:30,982 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:36,203 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:41,434 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:46,636 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:51,816 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:27:57,012 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:02,217 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:07,429 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:12,624 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:17,808 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:22,896 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:28,085 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:33,268 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:38,390 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:43,581 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:48,790 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:53,945 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:28:59,078 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:04,291 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:09,515 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:14,734 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:19,948 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:25,164 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:30,381 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:35,588 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:40,689 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:45,908 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:51,137 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:29:56,353 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:01,536 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:06,746 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:11,976 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:17,184 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:22,400 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:27,614 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:32,834 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:38,062 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:43,303 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:48,509 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:53,733 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:30:58,949 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:04,167 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:09,386 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:14,607 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:19,818 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:25,043 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:30,236 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:35,418 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:40,600 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:45,783 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:51,006 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:31:56,219 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:32:02,416 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:02,416 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:08,666 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:08,666 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:14,908 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:14,908 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:21,170 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:21,170 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:27,323 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:27,323 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:31,646 http 172.18.0.1 "GET /healthz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:32:33,549 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:33,549 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:39,802 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:39,802 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:46,055 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:46,055 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:49,099 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:49,099 http 172.18.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:52,303 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:52,303 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:32:58,400 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:32:58,400 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:00,284 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:00,284 http 172.18.0.1 "POST /increment HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:04,645 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:04,645 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:10,882 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:10,882 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:17,134 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:17,134 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:23,385 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:23,385 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:29,653 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:29,653 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:35,872 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:35,872 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:42,127 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:42,127 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:48,667 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:48,667 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:33:54,808 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:33:54,808 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:01,010 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:01,010 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:07,257 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:07,257 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:13,522 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:13,522 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:19,770 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:19,770 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:25,991 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:25,991 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:32,213 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:32,213 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:38,445 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:38,445 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:44,685 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:44,685 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:51,742 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:51,742 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:34:57,978 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:34:57,978 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:04,228 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:04,228 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:10,469 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:10,469 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:16,688 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:16,688 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:22,922 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:22,922 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:29,172 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:29,172 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:35,408 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:35,408 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:41,659 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:41,659 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:47,913 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:47,913 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:35:54,808 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:35:54,808 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:36:01,048 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:36:01,048 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:36:07,303 storage_unavailable: TimeoutError
app-1  | 2026-11-22 17:36:07,303 http 127.0.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 17:36:12,545 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:17,776 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:22,968 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:28,159 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:33,356 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:38,570 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:43,786 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:49,000 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:54,190 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:36:59,374 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:04,585 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:05,248 http 172.18.0.1 "GET / HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:09,806 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:15,002 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:20,213 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:25,431 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:30,581 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:35,792 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:41,004 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:46,226 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:51,439 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:37:56,632 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:01,811 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:07,007 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:12,232 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:17,442 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:22,687 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:27,911 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:33,116 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:36,960 http 172.18.0.1 "POST /increment HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:38,328 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:43,545 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:48,763 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:54,005 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:38:59,209 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:04,431 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:09,623 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:14,793 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:20,005 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:25,211 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:30,309 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:35,504 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:40,713 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
app-1  | 2026-11-22 17:39:45,935 http 127.0.0.1 "GET /readyz HTTP/1.1" 200 -
