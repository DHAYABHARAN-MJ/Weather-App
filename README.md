# Django Weather App – AWS EC2 Deployment Guide

This guide explains how to deploy a **Django Weather Application** on an **AWS EC2 (Ubuntu 22.04)** instance using **Gunicorn** and **Nginx**.

---
# Web App running at server
![Web App Screenshot](Screenshot(49).png)
## Architecture Overview

```
Browser → Nginx (Port 80) → Gunicorn (127.0.0.1:8000) → Django App
```

### Component Responsibilities

* **Nginx** – Reverse proxy, handles HTTP traffic
* **Gunicorn** – WSGI server running Django
* **Django** – Application logic
* **SQLite** – Database (for demo / learning purposes only)

> ⚠️ SQLite is **not recommended** for production-scale applications.

---

## Prerequisites

* AWS account
* GitHub account
* Basic Linux and Django knowledge
* A Django project containing:

  * `manage.py`
  * `requirements.txt`

---

## 1. Launch EC2 Instance

* **AMI**: Ubuntu 22.04 LTS
* **Instance Type**: t2.micro / t3.micro (Free Tier eligible)
* **Storage**: 8 GB (default)
* **Auto-assign Public IP**: Enabled

### Security Group Rules

| Type | Port | Source    |
| ---- | ---- | --------- |
| SSH  | 22   | My IP     |
| HTTP | 80   | 0.0.0.0/0 |

> Port **8000** may be opened temporarily for testing but **must be closed later**.

---

## 2. Connect to EC2 (Windows CMD)

```cmd
ssh -i weatherapp.pem ubuntu@<EC2_PUBLIC_IP>
```

---

## 3. Server Setup (Ubuntu)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3 python3-pip python3-venv git nginx -y
```

---

## 4. Clone the Project

```bash
git clone https://github.com/<your-username>/Weather-App.git
cd Weather-App/weather_app
```

### Project Structure

```
Weather-App/
├── weather_app/
│   ├── manage.py
│   ├── requirements.txt
│   ├── env/
│   └── weather_app/
```

---

## 5. Create & Activate Virtual Environment

```bash
python3 -m venv env
source env/bin/activate
```

### Install Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

---

## 6. Django Configuration

Edit `settings.py`:

```python
ALLOWED_HOSTS = ['<EC2_PUBLIC_IP>', 'localhost', '127.0.0.1']
DEBUG = False
```

### Run Migrations & Collect Static Files

```bash
python manage.py migrate
python manage.py collectstatic
```

---

## 7. Test Django (Temporary)

```bash
python manage.py runserver 0.0.0.0:8000
```

Open in browser:

```
http://<EC2_PUBLIC_IP>:8000
```

Stop the server:

```
CTRL + C
```

---

## 8. Install & Test Gunicorn

```bash
pip install gunicorn
gunicorn weather_app.wsgi:application --bind 127.0.0.1:8000
```

### Verify Internally

```bash
curl http://127.0.0.1:8000
```

Stop Gunicorn:

```
CTRL + C
```

---

## 9. Configure Nginx

Create a new Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/weatherapp
```

Paste the following:

```nginx
server {
    listen 80;
    server_name <EC2_PUBLIC_IP>;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Enable Site Configuration

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/weatherapp /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
```

---

## 10. Run Gunicorn in Background

```bash
gunicorn weather_app.wsgi:application --bind 127.0.0.1:8000 --daemon
```

> ⚠️ This is **not production-grade**.
> For real production, use **systemd + socket activation**.

---

## 11. Final Access

Open in browser:

```
http://<EC2_PUBLIC_IP>/
```

Your Django Weather Application is now live on **AWS EC2**.

---
