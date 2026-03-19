# ExcelTL — Django Edition (Open Source)

Refactor từ FastAPI + Vue.js → Django thuần, không có auth/license.

## Stack

| Thành phần | Công nghệ |
|---|---|
| Web framework | Django 5.x |
| ASGI server | Daphne |
| WebSocket | Django Channels |
| Scheduler | APScheduler + django-apscheduler |
| Frontend | Django Templates + HTMX + Bootstrap 5 |
| Database | SQLite (dev) / PostgreSQL (prod) |
| ETL Engine | `exceltl/` (giữ nguyên) |

## Cài đặt

```bash
cd exceltl_django
cp .env.sample .env
# Chỉnh sửa .env nếu cần

pip install -r requirements.txt

python manage.py migrate
python run.py          # http://localhost:8200
```

## Cấu trúc

```
exceltl_django/
├── config/          # settings, urls, asgi, wsgi
├── apps/
│   ├── core/        # Dashboard, ETL bridge
│   ├── jobs/        # Job management
│   ├── executions/  # Job execution + WebSocket consumer
│   ├── schedules/   # APScheduler integration
│   ├── files/       # File upload
│   └── variables/   # Global variables
├── templates/       # Django templates (HTMX)
├── static/          # CSS, JS
└── run.py           # Entry point (Daphne ASGI)
```

## URLs

| Path | Mô tả |
|---|---|
| `/` | Dashboard |
| `/jobs/` | Job list |
| `/executions/` | Execution history |
| `/schedules/` | Schedule management |
| `/files/` | File upload |
| `/variables/` | Global variables |
| `ws/executions/` | WebSocket (live status) |
| `/api/jobs/` | JSON API |
| `/api/system/health/` | Health check |

## Những gì đã xóa bỏ

- Toàn bộ Vue.js frontend (`frontend/`)
- FastAPI (`local_agent/`)
- License system (`exceltl/license_manager/`)
- Authentication / login
- Alembic migrations (thay bằng Django migrations)
