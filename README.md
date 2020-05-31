# Nginx + PHP + MariaDB + phpMyAdmin 安裝教學



完整安裝流程（所有安裝都需要以管理員身分執行）：
  - Nginx
  - PHP
  - MariaDB
  - phpMyAdmin
  - 優化




### 1. Nginx

使用管理員權限，更新apt：

```sh
sudo -s
apt update
apt upgrade -y
```

安裝nginx：

```sh
apt install nginx -y
```

查看nginx service是否執行

```sh
service nginx status
```

此時連線到這台伺服器，將會看到`Welcome to nginx!`字樣


### 2. PHP

#### 2.1 安裝PHP套件
安裝PHP預設版本常用套件：

```sh
apt install php-fpm php-mysql php-curl -y
```

如需指定版本的PHP，也可指定版本安裝：

```sh
apt install software-properties-common
add-apt-repository ppa:ondrej/php
apt update

apt install php7.4-fpm php7.4-mysql php7.4-curl -y
```

更改config檔案，至 `/etc/php/{版本代號，如7.4}/fpm` 下修改`php.ini`

```sh
nano /etc/php/7.4/fpm/php.ini
```

使用F6搜尋，找到 `cgi.fix_pathinfo`，將comment移除並修改為`cgi.fix_pathinfo=0`

```
cgi.fix_pathinfo=0
```

#### 2.2 測試PHP

修改nginx的預設router，至`/etc/nginx/sites-available`目錄，修改`default`檔案

```sh
nano /etc/nginx/sites-available/default
```

```nginx
        #新增index.php
        index index.php index.html index.htm index.nginx-debian.html;

        #新增php對應router，注意(/run/php/php7.4-fpm.sock)版本代號要與安裝的PHP版本相符
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        }
```

在`/var/www/html`中，新增一個`index.php`測試

```php
<?php
    echo phpinfo();
```

