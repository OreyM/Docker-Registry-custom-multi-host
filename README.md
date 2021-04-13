# Custom Docker registry

Необходимо:
- [Docker](https://www.docker.com/)
- [Ansible](https://www.ansible.com/)

Данный docker-реестр можно развернуть на том же сервере, что и ваш основной проект.

> Пожалуйста убедитесь, что подключение к хост серверу, на котором будет развернут docker-реестр, происходит по ssh-ключам

### Для начала

Скопируйте файл настроек:

```bash
cd .env.dist .env
```

Настройте его:

- Настройки доступа к вашему docker-реестру:
  - `DOCKER_LOGIN` - ваш собственный логин доступа
  - `DOCKER_PASSWORD` - ваш собственный пароль доступа
  - `HOST` - IP или адрес хост-сервера
  - `PORT` - порт хост-сервера
  - `HTPASSWD_FILE` - путь к файлу аутентификации

Проинициализировать docker-реестр на локальной машине:

```
make init
```

Сгенерировать свой файла аутентификации.

```
make init-access
```

Файл **htpasswd** будет сгенерирован в корне.

Скопировать файл настроек хоста docker-реестра:

```bash
cd provisioning
cp hosts.yml.dist hosts.yml
cd ..
```

Указать в файле настроек хоста **hosts.yml** IP адрес и порт своего сервера docker-реестра:

```yml
all:
    hosts:
        server:
            ansible_connection: ssh
            ansible_user: root
            ansible_host: 111.111.111.111
            ansible_port: 22
```

> Значения `ansible_host` и `ansible_port` должны совпадать со значениями `HOST` и `PORT` в `.env` файле.

Для запуска docker-реестра используется система автоматизации [Ansible](https://www.ansible.com/). Перечень команд в **Makefile**.

## Развертывание docker-реестра на хост-сервере:

### Настройка генерации SSL-сертификатов

Указать данные для настройки **certbot**:

```bash
cd provisioning/roles/registry/defaults
cp main.yml.dist main.yml
cd ../../../..
```

Выставить настройки в файле **main.yml**:

```yml
certbot_email: name@email.com # указать свой email адрес
certbot_hosts:
    - registry.host.com # указать url хост-сервера, на котором разворачивается docker-реестр
```

### Создание пользователя docker-реестром

По умолчанию, пользователь docker-реестром - **deploy**.

Подключите пользователя docker-реестра на хост сервере:

```bash
cd provisioning
make authorize
cd ..
```

Подключение к хост серверу через ssh:

```bash
ssh deploy@111.111.111.111 # server's IP
```

### Настройка Nginx

Скопируйте файл настроек Nginx:

```bash
cd docker/production/nginx/conf.d
cp registry.conf.dist registry.conf
cd ../../../..
```

Отредактируйте файл registry.conf:

```conf
server {
    listen 80;
    server_name registry.host.com; # <-- Ваш хост
    
    ...

    rewrite ^(.*) https://registry.host.com$1 permanent; # <-- Ваш хост
}
```

```conf
server {
    listen 443 ssl;
    server_name registry.host.com; # <-- Ваш хост

    ...

    ssl_certificate /etc/letsencrypt/live/registry.host.com/fullchain.pem;     # <-- Ваш хост
    ssl_certificate_key /etc/letsencrypt/live/registry.host.com/privkey.pem;   # <-- Ваш хост
    ssl_trusted_certificate /etc/letsencrypt/live/registry.host.com/chain.pem; # <-- Ваш хост
```

### Сборка и обновление docker-реестра

Запустить сборку сервера docker-реестра:

```bash
cd provisioning
make server
cd ..
```

Так же будут сгенерированы SSL-сертификаты (**Let's Encrypt**).

Для обновления docker-сервера:

```bash
cd provisioning
make upgrade
cd ..
```

### Деплой docker-реестра на хост сервер

```
make deploy
```

Если у вас другой порт или файл htpasswd хранится в другом месте, укажите это при формировании команды деплоя.

Теперь по адресу `https://registry.host.com:5443/v2/` будет расположен docker-сервер. 

Просмотр каталога образов: `https://registry.host.com:5443/v2/_catalog`.

> Не забудьте авторизовать  docker-реестр на хосте, для внешних подключений:
> ```bash
> ssh deploy@111.111.111.111
> docker login -u __LOGIN__ -p __PASSWORD__ registry.host.com:5443

Доступ к docker-реестру происходит через порт `5443`, во избежание конфликтов с другими проектами, развернутыми на том же хост сервере, и которые используют порт `443`. Вы можете изменить это поведение в **docker-compose-production.yml**:

```yml
...
        ports:
            - "5443:443"
...
```

Например:

```
...
        ports:
            - "80:80"
            - "443:443"
...
```