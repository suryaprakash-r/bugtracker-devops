
# syntax=docker/dockerfile:1

FROM python:3.14-slim

# Python runtime configuration
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Build dependencies required by mysqlclient
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        pkg-config \
        default-libmysqlclient-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies first for better build caching
COPY requirements.txt /app/requirements.txt

RUN python -m pip install --upgrade pip \
    && python -m pip install -r /app/requirements.txt

# Create an unprivileged application user
RUN useradd --create-home --shell /usr/sbin/nologin appuser

# Copy application source from the build context
COPY --chown=appuser:appuser . /app

# Prepare writable application directories
RUN mkdir -p /app/staticfiles /app/media /data \
    && SECRET_KEY=build-only-static-key DEBUG=False \
       python manage.py collectstatic --noinput \
    && chown -R appuser:appuser /app /data

USER appuser

EXPOSE 8000

# Development command for the initial container test.
# We will replace this with a production WSGI server before deployment.
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
