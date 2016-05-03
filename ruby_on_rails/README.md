#  **Configure Ruby on Rails on server**
----------

Мануал по настройке проекта на Ruby on Rails на сервере. 
Для этого нам понадобятся следующие инструменты:

 - RMV
 - Nginx
 - Unicorn

# RVM

Сначала используя официальную документацию (https://rvm.io/rvm/install), устанавливаем rvm.
В первую очередь необходимо добавить mpapis public key:

    gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3

Теперь устанавливаем RVM вместе с стабильной версией Ruby

    \curl -sSL https://get.rvm.io | bash -s stable --ruby

Теперь устанавливаем Ruby on Rails

    \curl -sSL https://get.rvm.io | bash -s stable --rails

Устанавливаем необходимые нам пакеты

    apt-get install ruby-dev rubygems libruby
и `gem`:

    gem install unicorn
    gem install rack
    gem install bundler

Переходим к конфигурации, проверяем версию установленного Ruby

    ruby -v

Т.к. на сервере проекты разворачиваем не под root пользователем, ему нужен доступ к rvm, для это создаем alias для той версии Ruby, которую будем использовать, в данном случае это 2.3.0.

    rvm alias create <username> ruby-2.3.0

Чтобы проверить, что alias успешно добавился, переходим в `/usr/local/rvm/wrappers` тут должна быть папка с именем того пользователя, для которого создавался `alias`, так же проверяем под самим пользователем, к примеру есть ли возможность установить `gem`.

# Unicorn

Мы уже ранее установили `gem` для `unicorn`, теперь нам необходимо его настроить.
Первым делом в `/etc/init.d/` добавляем файл `unicorn`

В каталоге `/etc/` создаем папку `unicorn` в ней будут лежать конфиги, для наших приложений.
К примеру содержимое файла (`example.conf`) будет такое:

    RAILS_ENV=production
    RAILS_ROOT=/<project path>
    UNICORN="/usr/local/rvm/wrappers/<username>/unicorn_rails"

Где `<project path>` это `root` каталог проекта.

В сам проект, в каталог `config` добавляем конфигурацию `unicorn.rb`

# NGINX

    upstream app {
      server unix:/<project path>/tmp/sockets/unicorn.sock;
    }
    
    server {
      listen      80;
      server_name www.<domain>;
      return      301 http://<domain>$request_uri;
    }
    
    server {
      server_name <domain>;
      listen      80;
    
      access_log  <project path>/log/access.log;
      error_log   <project path>/log/error.log;
      root        <project path>/public;
    
      location / {
        try_files $uri @app;
      }
    
      location @app {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect off;
    
        proxy_pass http://app;
      }
    }


# Supervisor

Для запуска `Active Job` будем использовать `supervisor`, его устанавливаем следующим образом:

    sudo apt-get install python-setuptools
    sudo easy_install supervisor
    echo_supervisord_conf > /etc/supervisord.conf

Так же нам понадобится `redis`

    apt-get install redis-server

Теперь, чтобы запускать `supervisor `под пользователем, где развернут проект нужно сначала добавить группу` supervisor` и назначить ее пользователю

    groupadd supervisor
    usermod -a -G supervisor <username>

Под самим пользователем, прописываем путь к supervisord.conf проекта. Делается это следующим образом:

    supervisord -c /<project path>/config/init.d/supervisor.conf
    supervisorctl -c /<project path>/config/init.d/supervisor.conf

