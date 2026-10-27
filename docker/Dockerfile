# syntax=docker/dockerfile:1

# ============================================
# Stage 1: Build Python dependencies
# ============================================
FROM python:3.14-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Required to compile mysqlclient
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        pkg-config \
        default-libmysqlclient-dev \
    && rm -rf /var/lib/apt/lists/*

# Create an isolated environment for dependencies
RUN python -m venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt /app/requirements.txt

RUN python -m pip install --upgrade pip \
    && python -m pip install -r /app/requirements.txt


# ============================================
# Stage 2: Minimal application runtime
# ============================================
FROM python:3.14-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Runtime library required by mysqlclient
RUN apt-get update \
    && apt-get upgrade -y --no-install-recommends \
    && apt-get install -y --no-install-recommends \
        libmariadb3 \
    && rm -rf /var/lib/apt/lists/*

# Copy only the Python environment from the builder
COPY --from=builder /opt/venv /opt/venv

# Run the application as a non-root user
RUN useradd --create-home --shell /usr/sbin/nologin appuser

# Copy application source
COPY --chown=appuser:appuser . /app

# Collect static assets and prepare writable directories
RUN mkdir -p /app/staticfiles /app/media /data \
    && SECRET_KEY=build-only-static-key DEBUG=False \
       python manage.py collectstatic --noinput \
    && chown -R appuser:appuser /app /data

USER appuser

EXPOSE 8000

# Production WSGI server
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "2", "--access-logfile", "-", "--error-logfile", "-"]