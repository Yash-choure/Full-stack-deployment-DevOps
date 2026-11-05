FROM python:3.13-slim AS builder

WORKDIR /app

RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir \
        django \
        django-cors-headers \
        psycopg2-binary

FROM python:3.13-slim

WORKDIR /app

ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN groupadd --system django \
    && useradd --system --gid django --create-home django

COPY --from=builder /opt/venv /opt/venv
COPY . .

RUN chown -R django:django /app
USER django

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
