FROM python:3.9-slim

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /webapp
COPY requirements.txt /webapp/

RUN apt-get update                                                            \
    && apt-get install -y --no-install-recommends build-essential libpq-dev   \
    && pip install --no-cache-dir -r requirements.txt                         \
    && apt-get purge -y build-essential                                       \
    && apt-get autoremove -y                                                  \
    && rm -rf /var/lib/apt/lists/*

COPY src /webapp/src
WORKDIR /webapp/src

# Собираем статику на этапе сборки, чтобы контейнер можно было сразу запускать
RUN SECRET_KEY=empty python manage.py collectstatic --noinput

EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
