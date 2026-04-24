--- install_django_backend.md (原始)


+++ install_django_backend.md (修改后)
# Поднятие Django бэкенда в Docker для тестирования

Это руководство описывает шаги по запуску Django-приложения в контейнере Docker для целей тестирования.

## Предварительные требования

Убедитесь, что у вас установлены:
*   **Docker**: [Инструкция по установке](https://docs.docker.com/get-docker/)
*   **Docker Compose** (обычно входит в состав Docker Desktop): [Инструкция по установке](https://docs.docker.com/compose/install/)

## Структура проекта

Для работы вам понадобятся следующие файлы в корне вашего проекта:

1.  `Dockerfile` — инструкция по сборке образа Django.
2.  `docker-compose.yml` — описание сервисов (веб-сервер, база данных).
3.  `.env` (опционально) — файл с переменными окружения (пароли, настройки).
4.  `requirements.txt` — список зависимостей Python.

---

## Шаг 1: Создание Dockerfile

Создайте файл `Dockerfile` в корне проекта:

```dockerfile
# Используем официальный образ Python
FROM python:3.9-slim

# Устанавливаем переменные окружения
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Устанавливаем рабочую директорию внутри контейнера
WORKDIR /app

# Устанавливаем системные зависимости (если нужны, например, для psycopg2)
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Копируем файл зависимостей и устанавливаем их
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt

# Копируем весь код проекта в контейнер
COPY . .

# Открываем порт (по умолчанию 8000)
EXPOSE 8000

# Команда для запуска сервера разработки
# Для продакшена используйте gunicorn или uvicorn
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

## Шаг 2: Создание docker-compose.yml

Создайте файл `docker-compose.yml` для оркестрации контейнеров (Django + PostgreSQL):

```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    volumes:
      - postgres_/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${DB_NAME:-mydb}
      POSTGRES_USER: ${DB_USER:-user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-password}
    ports:
      - "5432:5432"

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      DB_NAME: ${DB_NAME:-mydb}
      DB_USER: ${DB_USER:-user}
      DB_PASSWORD: ${DB_PASSWORD:-password}
      DB_HOST: db
      DB_PORT: 5432

volumes:
  postgres_
```

## Шаг 3: Настройка переменных окружения

Создайте файл `.env` в корне проекта (не добавляйте его в git, если там есть секреты):

```bash
DB_NAME=mytestdb
DB_USER=testuser
DB_PASSWORD=testpassword
SECRET_KEY=your-secret-key-here
DEBUG=True
```

Убедитесь, что ваш `settings.py` в Django читает эти переменные. Пример настройки `DATABASES`:

```python
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'db'), # 'db' - имя сервиса в docker-compose
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

Также разрешите хосты в `settings.py`:
```python
ALLOWED_HOSTS = ['*'] # Только для тестирования!
```

## Шаг 4: Запуск проекта

1.  **Сборка и запуск контейнеров:**
    Выполните команду в терминале из корня проекта:
    ```bash
    docker-compose up --build
    ```
    *   Флаг `--build` заставляет пересобрать образ перед запуском.
    *   Чтобы запустить в фоновом режиме, добавьте флаг `-d`: `docker-compose up -d --build`.

2.  **Применение миграций:**
    Если это первый запуск, нужно создать таблицы в БД. Откройте новый терминал и выполните:
    ```bash
    docker-compose exec web python manage.py migrate
    ```

3.  **Создание суперпользователя (опционально):**
    ```bash
    docker-compose exec web python manage.py createsuperuser
    ```

## Шаг 5: Проверка работы

Откройте браузер и перейдите по адресу:
[http://localhost:8000](http://localhost:8000)

Вы должны увидеть стартовую страницу Django или ваше приложение.

## Полезные команды

*   **Остановить контейнеры:**
    ```bash
    docker-compose down
    ```
*   **Остановить и удалить тома (база данных будет очищена):**
    ```bash
    docker-compose down -v
    ```
*   **Посмотреть логи:**
    ```bash
    docker-compose logs -f web
    ```
*   **Выполнить команду внутри контейнера:**
    ```bash
    docker-compose exec web <команда>
    # Например: docker-compose exec web python manage.py shell
    ```

## Устранение неполадок

*   **Ошибка подключения к БД:** Убедитесь, что в `settings.py` хост указан как `db` (имя сервиса в docker-compose), а не `localhost`.
*   **Контейнер не запускается:** Проверьте логи через `docker-compose logs web`. Частая причина — отсутствие файла `requirements.txt` или ошибки в синтаксисе Python.
*   **Изменения в коде не применяются:** Убедитесь, что в `docker-compose.yml` для сервиса `web` подключен том `- .:/app`, который синхронизирует локальную папку с контейнером.
