# WordPress в Docker

## Официальный образ

WordPress имеет официальный образ на Docker Hub: [wordpress](https://hub.docker.com/_/wordpress)

Официальный образ поддерживается командой Docker и сообществом WordPress, регулярно обновляется и включает все необходимые зависимости для работы.

---

## Системные требования

Для работы WordPress требуется определенный набор программного обеспечения. Согласно [официальным требованиям](https://wordpress.org/about/requirements/), рекомендуется:

### PHP

- **Рекомендуемая версия**: PHP 8.3 или выше

### База данных (СУБД)

WordPress поддерживает следующие системы управления базами данных:

**MySQL**
:   Рекомендуемая версия 8.0 или выше

**MariaDB**
:   Рекомендуемая версия 10.6 или выше

### HTTPS

Рекомендуется использовать HTTPS для безопасности

### HTTP-сервер

**Apache**
:   Наиболее распространенный выбор, включает модуль mod_rewrite для постоянных ссылок

**Nginx**
:   Альтернативный веб-сервер с высокой производительностью

В официальном образе WordPress используется **Apache**, так как он проще в настройке и является стандартом для большинства установок.

---

## Текущая версия WordPress

На момент написания статьи последняя стабильная версия WordPress: **6.9**

Проверить актуальную версию можно на [странице загрузки](https://wordpress.org/download/).

---

## Установка с Docker Compose

### 1. Создайте файл compose.yaml

```yaml title="compose.yaml"
services:
  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5
      start_period: 30s

  wordpress:
    image: wordpress:php8.4-apache
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: ${WORDPRESS_DB_HOST}
      WORDPRESS_DB_USER: ${WORDPRESS_DB_USER}
      WORDPRESS_DB_PASSWORD: ${WORDPRESS_DB_PASSWORD}
      WORDPRESS_DB_NAME: ${WORDPRESS_DB_NAME}
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      db:
        condition: service_healthy

  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    depends_on:
      db:
        condition: service_healthy

volumes:
  db_data:
  wordpress_data:
```

### 2. Создайте файл .env

```ini title=".env"
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_DATABASE=wordpress
MYSQL_USER=wordpress_user
MYSQL_PASSWORD=your_wordpress_password

WORDPRESS_DB_HOST=db:3306
WORDPRESS_DB_USER=wordpress_user
WORDPRESS_DB_PASSWORD=your_wordpress_password
WORDPRESS_DB_NAME=wordpress
```

!!! warning "Важно"
    Замените `your_root_password` и `your_wordpress_password` на свои безопасные пароли!

!!! tip "Совет"
    Добавьте `.env` в `.gitignore`, чтобы не закоммитить пароли в репозиторий.

---

## Разбор конфигурации

### Сервис `db` (MySQL)

```yaml
db:
  image: mysql:8.0  # (1)!
  volumes:
    - db_data:/var/lib/mysql  # (2)!
  environment:  # (3)!
    MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    MYSQL_DATABASE: ${MYSQL_DATABASE}
    MYSQL_USER: ${MYSQL_USER}
    MYSQL_PASSWORD: ${MYSQL_PASSWORD}
  healthcheck:  # (4)!
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    retries: 5
    start_period: 30s
```

1. **`image: mysql:8.0`** - Использует официальный образ MySQL версии 8.0 (рекомендуемая версия для WordPress)
2. **`volumes`** - Монтирует том `db_data` в `/var/lib/mysql` для сохранения данных базы
3. **`environment`** - Переменные окружения из `.env` файла:
    - `MYSQL_ROOT_PASSWORD` - пароль root-пользователя MySQL
    - `MYSQL_DATABASE` - создаёт базу данных с именем `wordpress`
    - `MYSQL_USER` - создаёт пользователя для WordPress
    - `MYSQL_PASSWORD` - пароль для пользователя WordPress
4. **`healthcheck`** - Проверка работоспособности MySQL:
    - `test` - команда для проверки (пинг MySQL через `mysqladmin`)
    - `interval: 10s` - проверять каждые 10 секунд
    - `retries: 5` - максимум 5 попыток
    - `start_period: 30s` - период запуска (30 секунд на инициализацию БД)

### Сервис `wordpress`

```yaml
wordpress:
  image: wordpress:php8.4-apache  # (1)!
  ports:
    - "8080:80"  # (2)!
  environment:  # (3)!
    WORDPRESS_DB_HOST: ${WORDPRESS_DB_HOST}
    WORDPRESS_DB_USER: ${WORDPRESS_DB_USER}
    WORDPRESS_DB_PASSWORD: ${WORDPRESS_DB_PASSWORD}
    WORDPRESS_DB_NAME: ${WORDPRESS_DB_NAME}
  volumes:
    - wordpress_data:/var/www/html  # (4)!
  depends_on:  # (5)!
    db:
      condition: service_healthy
```

1. **`image: wordpress:php8.4-apache`** - Использует WordPress с PHP 8.4 и Apache (последняя стабильная версия PHP)
2. **`ports`** - Пробрасывает порт 80 контейнера на порт 8080 хоста (доступ через `http://localhost:8080`)
3. **`environment`** - Настройки подключения к базе данных из `.env`:
    - `WORDPRESS_DB_HOST` - адрес сервера MySQL (`db:3306`)
    - `WORDPRESS_DB_USER` - пользователь базы данных
    - `WORDPRESS_DB_PASSWORD` - пароль пользователя
    - `WORDPRESS_DB_NAME` - имя базы данных
4. **`volumes`** - Монтирует том `wordpress_data` в `/var/www/html` для сохранения файлов WordPress (темы, плагины, загрузки)
5. **`depends_on`** - Запуск WordPress только после того, как MySQL станет **healthy** (пройдёт healthcheck)

### Сервис `adminer`

```yaml
adminer:
  image: adminer:latest  # (1)!
  ports:
    - "8081:8080"  # (2)!
  depends_on:
    db:
      condition: service_healthy  # (3)!
```

1. **`image: adminer:latest`** - Легковесный инструмент для управления базами данных через веб-интерфейс
2. **`ports`** - Пробрасывает порт 8080 контейнера на порт 8081 хоста (доступ через `http://localhost:8081`)
3. **`depends_on`** - Запускается только после того, как MySQL станет healthy

**Adminer** - это альтернатива phpMyAdmin для управления MySQL. Позволяет просматривать таблицы, выполнять SQL-запросы и администрировать базу данных.

### Volumes (Тома)

```yaml
volumes:
  db_data:  # (1)!
  wordpress_data:  # (2)!
```

1. **`db_data`** - Том для хранения данных MySQL (таблицы, индексы, конфигурация)
2. **`wordpress_data`** - Том для хранения файлов WordPress (темы, плагины, загрузки, wp-config.php)

Тома обеспечивают **постоянство данных**: даже если вы удалите контейнеры, данные сохранятся.

---

## Запуск

### 1. Запустить все сервисы

В директории с `compose.yaml` выполните:

```bash
docker compose up -d
```

!!! note "Примечание"
    Используйте `docker compose` (без дефиса) - это современная команда Compose V2.

Флаг `-d` (detached) запускает контейнеры в фоновом режиме.

### 2. Проверить статус

```bash
docker compose ps
```

Все три контейнера должны быть в статусе `running`, а `db` должен быть `healthy`.

### 3. Открыть WordPress

Откройте браузер и перейдите:

```text
http://localhost:8080
```

Вы увидите стартовую страницу установки WordPress.

### 4. Открыть Adminer (опционально)

Для управления базой данных откройте:

```text
http://localhost:8081
```

Введите данные для подключения:

- **System**: MySQL
- **Server**: `db`
- **Username**: значение из `MYSQL_USER` в `.env`
- **Password**: значение из `MYSQL_PASSWORD` в `.env`
- **Database**: значение из `MYSQL_DATABASE` в `.env`

!!! success "Готово!"
    Следуйте инструкциям мастера установки WordPress для завершения настройки.

---

## Управление

### Остановить все сервисы

```bash
docker compose stop
```

### Запустить снова

```bash
docker compose start
```

### Перезапустить

```bash
docker compose restart
```

### Остановить и удалить контейнеры

```bash
docker compose down
```

!!! warning "Внимание"
    Команда `down` удаляет контейнеры, но **данные в томах сохраняются**.

### Удалить всё (включая данные)

```bash
docker compose down -v
```

!!! danger "Осторожно!"
    Флаг `-v` удалит **все данные**, включая базу данных и файлы WordPress!

---

## Просмотр логов

### Все сервисы

```bash
docker compose logs
```

### Конкретный сервис

```bash
docker compose logs wordpress
docker compose logs db
docker compose logs adminer
```

### В реальном времени

```bash
docker compose logs -f
```

Нажмите `Ctrl+C` для выхода.

---

## Почему используется healthcheck?

**Healthcheck** для MySQL гарантирует, что WordPress и Adminer запустятся **только после того**, как база данных полностью инициализируется и будет готова принимать подключения.

Без healthcheck WordPress может попытаться подключиться к MySQL слишком рано (пока БД ещё инициализируется), что приведёт к ошибкам подключения.

**Как это работает:**

1. MySQL запускается и начинает инициализацию (создание БД, пользователя)
2. Healthcheck каждые 10 секунд проверяет доступность MySQL командой `mysqladmin ping`
3. После 5 успешных проверок MySQL получает статус `healthy`
4. Только после этого запускаются WordPress и Adminer

---

## Полезные ссылки

- [Официальный образ WordPress на Docker Hub](https://hub.docker.com/_/wordpress)
- [Официальный образ MySQL на Docker Hub](https://hub.docker.com/_/mysql)
- [Официальный образ Adminer на Docker Hub](https://hub.docker.com/_/adminer)
- [Системные требования WordPress](https://wordpress.org/about/requirements/)
- [Скачать WordPress](https://wordpress.org/download/)
- [Compose Application Model](https://docs.docker.com/compose/intro/compose-application-model/)
