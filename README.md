# Инструкция по настрйоке VPS-сервера
### Базовая настройка сервера
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
* Установка минимального набора необходимых программ
```
sudo apt-get update ; \
sudo apt-get install -y vim tmux htop git curl wget unzip zip gcc build-essential make ; \
sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python3-lxml libxslt-dev python3-libxml2 libffi-dev libssl-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
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
* Перезагрузить настройки ZSH
```
source ~/.zshrc
```
    
# Установка Django
* Создаём папку приложения (заменить `<root_dir>` на удобное название проекта)
```
mkdir <root_dir> && cd <root_dir>
```
* Скачиваем проект с Github
```
git clone <repo>
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

