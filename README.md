# Инструкция по настройке VPS-сервера
В этой инструкции описывается процедура настройки сервера для запуска Django-приложений.  
Предполагается, что базовая настройка сервера уже была проведена. Подробная инструкция, как это сделать, находится в репозитории [basic-server-setup](https://github.com/Shecspi/basic-server-setup)
## Установка Poetry
```bash
curl -sSL https://install.python-poetry.org | python3 -; \
echo 'export PATH=$HOME/.local/bin:{$PATH}' >> ~/.zshrc; \
source ~/.zshrc
```
    
## Установка Django
* Создаём папку приложения (заменить `<ROOT_DIR>` на удобное название проекта)
```bash
mkdir <ROOT_DIR> && cd <ROOT_DIR>
```
* Скачиваем проект с Github
```bash
git clone <REPO> .
```
* Создаём виртуальное окружение устанавливаем все зависимости проекта
```bash
poetry install
```
* Устанавливаем Gunicorn
```bash
poetry add gunicorn
```
* Если необходимо - настраиваем проект (settings.py, .env и т.д.)
* Устанавливаем PostgreSQL и Python-модуль
```bash
sudo apt-get install postgresql;
poetry add psycopg2;
```
* Входим в консоль PostgreSQL под пользователем postgres
```bash
sudo -u postgres psql postgres
```
* Меняем стандартный пароль пользователя postgres
```sql
\password postgres
```
* Создаём пользователя, который будет взаимодействовать в базой данных
```sql
create user <username> with password '<password>';
alter role <username> set client_encoding to 'utf8';
alter role <username> set default_transaction_isolation to 'read committed';
alter role <username> set timezone to 'Europe/Moscow';
```
Создаём базу данных проекта
```sql
create database <project_db> owner <username>;
```
* Делаем миграции
```bash
poetry run python3 manage.py makemigrations && poetry run python3 manage.py migrate
```
* Если нужно - загружаем дамп базы данных
```bash
poetry run python3 manage.py loaddata dump.json
```
* Открываем порт 8000
```bash
sudo ufw allow 8000
```
* Запускаем develop-сервер для проверки
```bash
poetry run python3 manage.py runserver 0:8000
```

## Настройка Gunicorn
* Устанавливаем Gunicorn
```
poetry add gunicorn
```
* Создать файл `/etc/systemd/system/gunicorn.socket` и добавить в него следующий код:
```
[Unit]
Description=gunicorn socket
[Socket]
ListenStream=/run/gunicorn.sock
[Install]
WantedBy=sockets.target
```
* Создать файл `/etc/systemd/system/gunicorn.service` и добавить в него следующий код:
```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target
[Service]
User=<USER>
Group=<GROUP>
WorkingDirectory=<ROOT_DIR>
ExecStart=<POETRY_VENV_DIR>/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          <PROJECT_NAME>.wsgi:application
Restart=on-failure
RestartSec=5s
[Install]
WantedBy=multi-user.target
```
* Запускаем Gunicorn и добавляем его в автозагрузку
```
sudo systemctl start gunicorn.socket; \
sudo systemctl enable gunicorn.socket
```
* Проверяем, что Gunicorn запустился
```
sudo systemctl status gunicorn.socket
```
* Проверяем, что файл существует
```
file /run/gunicorn.sock
```
* gunicorn.service должен быть неактивен
```
sudo systemctl status gunicorn
```
* Если следующая команда возвращает HTML - то всё ок
```
curl --unix-socket /run/gunicorn.sock localhost
```
* Убедиться в этом можно выполнив
```
sudo systemctl status gunicorn
```
* Перезапускаем Gunicorn
```
sudo systemctl daemon-reload; \
sudo systemctl restart gunicorn
```
## Настройка Nginx
* Устанавливаем Nginx
```bash
sudo apt install nginx
```
* Создать файл `/etc/nginx/sites-available/<PROJECT_NAME>` со следующим содержимым:
```
server {
  listen 80;
  server_name <DOMAIN OR IP>;
  location /static/ {
    autoindex on;
    root /var/www;
  }
  location / {
    include proxy_params;
    proxy_pass http://unix:/run/gunicorn.sock;
  }
}
```
* Создать символьную ссылку
```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```
* Проверить работу Nginx
```
sudo nginx -t
```
* Перезапустить Nginx
```
sudo systemctl restart nginx
```
* Изменить настройки портов
```
sudo ufw delete allow 8000
sudo ufw allow 'Nginx Full'
```

## Установка SSL
```bash
sudo apt-get install snapd; \
sudo snap install core; \
sudo snap install --classic certbot; \
sudo ln -s /snap/bin/certbot /usr/bin/certbot; \
sudo certbot --nginx
```
