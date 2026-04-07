#docker
# Docker — основные команды (шпаргалка 2025–2026)

## 1. Информация и состояние

```bash 
docker --version # версия 
docker docker info # подробная информация о системе 
docker system df # сколько места занимают образы/контейнеры/volumes 
docker system prune # удалить всё неиспользуемое (осторожно!) 
docker system prune -a --volumes # удалить ВСЁ неиспользуемое (очень агрессивно)
```
## 2. Образы (images)

```bash 
docker images # список локальных образов 
docker image ls # то же самое, короче 
docker pull nginx:latest # скачать образ 
docker pull alpine # :latest подразумевается 
docker rmi nginx:latest # удалить образ 
docker rmi -f $(docker images -q) # удалить все образы (опасно!)
```

## 3. Контейнеры

```bash 
docker ps                         # запущенные контейнеры
docker ps -a                      # все контейнеры (включая остановленные)
docker ps -q                      # только ID запущенных контейнеров
docker ps -a -q                   # все ID контейнеров

docker run hello-world            # запустить тестовый контейнер
docker run -it ubuntu bash        # интерактивный терминал
docker run -d --name my-nginx -p 8080:80 nginx   # в фоне + проброс порта

docker start my-nginx             # запустить остановленный контейнер
docker stop my-nginx              # остановить (SIGTERM)
docker kill my-nginx              # убить (SIGKILL)
docker restart my-nginx           # перезапустить

docker rm my-nginx                # удалить остановленный контейнер
docker rm -f my-nginx             # force — остановить + удалить
docker rm $(docker ps -a -q)      # удалить все остановленные контейнеры
```

## 4. Exec и логи

```bash 
docker exec -it my-nginx bash          # зайти в запущенный контейнер
docker exec -it my-nginx sh            # если bash нет
docker exec my-nginx cat /etc/hosts    # выполнить одну команду

docker logs my-nginx                   # посмотреть логи
docker logs -f my-nginx                # следить за логами в реальном времени
docker logs --tail 100 my-nginx        # последние 100 строк
```

## 5. Копирование файлов

```bash 
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf         # из контейнера → хост
docker cp ./my-app.conf my-nginx:/etc/nginx/conf.d/default.conf  # хост → контейнер
```

## 6. Volumes и bind mounts (самое важное)

```bash 
# Пример запуска с volume (рекомендуемый способ)
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16

# Список volume'ов
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata
```

## 7. Docker Compose (самые частые команды)

```bash 
docker compose up                   # запустить в foreground
docker compose up -d                # в фоне (detached)
docker compose down                 # остановить и удалить контейнеры
docker compose down -v              # + удалить volume'ы
docker compose ps                   # статус сервисов
docker compose logs -f              # логи всех сервисов
docker compose exec web bash        # зайти в сервис web
docker compose pull                 # обновить образы
docker compose build                # пересобрать образы
```

## 8. Быстрые one-liners (очень часто используются)

```bash 
# Убить и удалить все контейнеры
docker rm -f $(docker ps -aq)

# Удалить все неиспользуемые образы
docker image prune -a

# Очистить всё (контейнеры + образы + volume + сеть)
docker system prune --all --volumes --force

# Запустить временный контейнер для теста
docker run --rm -it alpine sh
```

## Полезные флаги, которые стоит запомнить

- -d → detached (в фоне)
- -it → интерактивный терминал + tty
- --rm → удалить контейнер после остановки
- -p 8080:80 → проброс порта хост:контейнер
- -v /host/path:/container/path → монтирование
- --name myapp → задать имя контейнеру
- --restart unless-stopped → перезапускать автоматически
