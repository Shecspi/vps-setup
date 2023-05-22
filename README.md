# Инструкция по настрйоке VPS-сервера
## Базовая настройка сервера
* Создать нового пользователя `www`
```
adduser www
```
* Добавить пользователя `www` в группу `sudo` для того, чтобы он мог выполнять инструкции с правами суперпользователя
```
usermod -aG sudo www
```
* Теперь можно подключиться по SSH от имени пользователя `www` и все дальнейшие операции производить от него. Но при этому, от `root` пока что лучше не отключаться, чтобы была возможность что-то исправить.
* Включить брандмаузер и запретить доступ с IP-адресов, с которых пирходит больше 5 попыток подключения за полминуты.
```
sudo ufw enable && sudo ufw limit ssh
```
* Скопировать SSH-ключи на сервер, для этого **на локальном компьютере** выполнить команду
```
ssh-copy-id www@<IP-address>
```
После этого можно будет подключаться без ввода пароля
```
ssh www@<IP-address>
```
* Также на **локальном компьютере** можно создать файл `.ssh/config` и прописать в него следующее (заменив <IP-address> на реальный IP-адрес сервера, а NameForYou - на удобное имя):
```
Host NameForYou
    HostName <IP-address>
    IdentityFile ~/.ssh/id_rsa
    User www
```
После этого можно будет подключаться к серверу командой
```
ssh NameForYou
```
* В файле `/etc/ssh/sshd_config` изменить значения переменных
```
AllowUsers www
PermitRootLogin no
PasswordAuthentication no
```

* Перезагрузить SSH-сервер
```
sudo service ssh restart
```
* Установка `fial2ban` для блокирования IP-адресов, пытающихся подобрать пароль к SSH.
```bash
sudo apt install fail2ban && sudo systemctl enable fail2ban && sudo systemctl start fail2ban
```
Посмотреть статус можно командой
```bash
sudo fail2ban-client status sshd
```
* Установка минимального набора необходимых программ
```
sudo apt-get update ; \
sudo apt-get install -y vim tmux htop git curl wget unzip zip gcc build-essential make ; \
sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python3-lxml libxslt-dev python3-libxml2 libffi-dev libssl-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```
* Установка часового пояса
```
sudo dpkg-reconfigure tzdata
```

### Установка oh-my-zsh и плагинов
* Установка oh-my-zsh
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
* Установка плагина для автодополнения
```
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```
В файл `~/.zshrc` добавить плагин:
```
plugins=( 
    # other plugins...
    zsh-autosuggestions
)
```
* Установка плагина с подсветкой синтаксиса
```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
echo "source ${(q-)PWD}/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc
```
* Изменить тему оформления в файле `~/.zshrc` на `gnzh`
* Перезагрузить настройки ZSH
```
source ~/.zshrc
```

## Установка Poetry
* Установка
```bash
curl -sSL https://install.python-poetry.org | python3 -
```
* Добавляем `~/.local/bin' в `PATH`
```bash
echo 'export PATH=$HOME/.local/bin:{$PATH}' >> ~/.zshrc
```
    
## Установка Django
* Создаём папку приложения (заменить `<ROOT_DIR>` на удобное название проекта)
```
mkdir <ROOT_DIR> && cd <ROOT_DIR>
```
* Скачиваем проект с Github
```
git clone <REPO>
```
* Создаём виртуальное окружение и активируем его
```
python3 -m venv venv && source venv/bin/activate
```
* Устанавливаем Gunicorn и зависимости проекта
```
pip3 install --upgrade pip && pip3 install gunicorn && pip3 install -r requirements.txt
```
* Если необходимо - настраиваем проект (настраиваем settings.py, .env и т.д.)
* Делаем миграции
```
python3 manage.py makemigrations && python3 manage.py migrate
```
* Если нужно - заагружаем дамп базы данных
* Открываем порт 8000
```
sudo ufw allow 8000
```
* Запускаем develop-сервер для проверки
```
python3 manage.py runserver 0:8000
```

## Настройка Gunicorn
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
ExecStart=<ROOT_DIR>/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          <PROJECT_NAME>.wsgi:application
Restart=on-failure
RestartSec=5s
[Install]
WantedBy=multi-user.target
```
* Запускаем Gunicorn
```
sudo systemctl start gunicorn.socket
```
* Добавляем Gunicron в автозагрузку
```
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
sudo systemctl daemon-reload
```
```
sudo systemctl restart gunicorn
```
## Настройка Nginx
* Восздать файл `/etc/nginx/sites-available/<PROJECT_NAME>` со следующим содержимым:
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
* Установить Certbot
```bash
sudo apt-get install snapd
```
```bash
sudo snap install core
```
```bash
sudo snap install --classic certbot
```
```bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
* Настройка Certbot
```bash
sudo certbot --nginx
```
