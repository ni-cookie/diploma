# Этап 1: Builder (Сборщик)
FROM python:3.12-slim AS builder

# Копируем готовый бинарник uv прямо из их официального образа
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Устанавливаем системные зависимости для сборки C-расширений (если нужны для БД)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Говорим uv компилировать байт-код сразу (ускоряет холодный старт)
ENV UV_COMPILE_BYTECODE=1
# Жестко задаем путь, где uv должен создать виртуальное окружение
ENV UV_PROJECT_ENVIRONMENT="/opt/venv"

# Копируем файлы конфигурации проекта
COPY pyproject.toml uv.lock* ./

# Синхронизируем зависимости
# --no-dev: не ставим пакеты для тестирования/разработки (pytest, ruff и т.д.)
# --no-install-project: ставим ТОЛЬКО библиотеки, без самого приложения (для кэширования слоя)
RUN uv sync --no-dev --no-install-project

# Этап 2: Runner (Финальный легковесный образ)
FROM python:3.12-slim

# Практика DevSecOps: Создаем системную группу и пользователя без прав root
RUN groupadd -r fastapi_group && useradd -r -g fastapi_group fastapi_user

# Устанавливаем только runtime-зависимости для PostgreSQL (без тяжелого gcc)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Копируем готовое виртуальное окружение из первого этапа
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Копируем код приложения и конфигурацию миграций
COPY app/ ./app/
COPY alembic/ ./alembic/
COPY alembic.ini .

# Передаем права на файлы нашему безопасному пользователю
RUN chown -R fastapi_user:fastapi_group /app

# Переключаемся на этого пользователя (теперь контейнер работает НЕ от root)
USER fastapi_user

# Оптимизация работы Python в контейнере
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Указываем, какой порт слушает приложение
EXPOSE 8000

# Команда запуска FastAPI через uvicorn
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]