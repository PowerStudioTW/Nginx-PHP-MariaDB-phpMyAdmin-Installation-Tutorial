# Nginx + PHP + MariaDB + phpMyAdmin Ubuntu安裝教學



安裝教學流程（全程都需要以管理員身分執行）：
  - Nginx
  - PHP
  - MariaDB
  - phpMyAdmin
  - 優化




### 1. Nginx

#### 1.1. 更新apt
使用管理員權限，更新apt：

```bash
sudo -s
apt update
apt upgrade -y
```

#### 1.2. 安裝nginx套件
執行指令安裝nginx：

```bash
apt install nginx -y
```

查看nginx service是否執行

```bash
service nginx status
```

此時連線到這台伺服器，將會看到 *Welcome to nginx!* 字樣


### 2. PHP

#### 2.1 安裝PHP套件
安裝PHP預設版本常用套件：

```bash
apt install php-fpm php-mysql php-curl -y
```

如需指定版本的PHP，也可指定版本安裝：

```bash
apt install software-properties-common
add-apt-repository ppa:ondrej/php
apt update

apt install php7.4-fpm php7.4-mysql php7.4-curl -y
```

更改config檔案，至 `/etc/php/{版本代號，如7.4}/fpm` 下修改`php.ini`

```bash
nano /etc/php/7.4/fpm/php.ini
```

使用F6搜尋，找到 `cgi.fix_pathinfo`，將comment移除並修改為`cgi.fix_pathinfo=0`

```
cgi.fix_pathinfo=0
```

#### 2.2 測試PHP

修改nginx的預設router，至`/etc/nginx/sites-available`目錄，修改`default`檔案：

```bash
nano /etc/nginx/sites-available/default
```
在`server { } `中修改`index`：
```nginx
        #新增index.php
        index index.php index.html index.htm index.nginx-debian.html;
```
在`server { } `中新增一組`location`：
```nginx
        #新增php對應router，注意(/run/php/php7.4-fpm.sock)版本代號要與安裝的PHP版本相符
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        }
```

檢查nginx語法是否有錯誤：
```bash
nginx -t
```
若語法正確將會回傳類似如下回應：

*nginx: the configuration file /etc/nginx/nginx.conf syntax is ok*

*nginx: configuration file /etc/nginx/nginx.conf test is successful*


在`/var/www/html`中，新增一個`index.php`測試檔案：
```bash
nano /var/www/html/index.php
```

```php
<?php
    echo phpinfo();
    exit();
```

此時重新整理網站，將會看到PHP版本資訊。

#### 2.3. 安裝Composer套件

```bash
php -r "copy('https://getcomposer.org/installer', '/tmp/composer-setup.php');"
sudo php /tmp/composer-setup.php --install-dir=/usr/bin --filename=composer
rm /tmp/composer-setup.php
```

檢查Composer安裝版本：
```bash
composer -V
```

### 3. MariaDB

#### 3.1. 安裝MariaDB套件
直接安裝MariaDB：

```bash
apt install mariadb-server -y
```

或是指定版本安裝，可由MariaDB官方網站查看如何安裝：
https://downloads.mariadb.org/mariadb/repositories/#mirror=nodesdirect 

此教學以Ubuntu 18.04 LTS搭配MariaDB 10.4為例：

```bash
sudo apt-get install software-properties-common
sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://mirror.nodesdirect.com/mariadb/repo/10.4/ubuntu bionic main'
```
```bash
apt update
apt install mariadb-server
```

檢查套件是否正常開啟：
```bash
service mysqld status
```

#### 3.2. 執行MySQL安全安裝

在安裝mariaDB套件後必須執行`mysql_secure_installation`指令才能正常使用：

```bash
mysql_secure_installation
```
依照安裝步驟安裝即可。

#### 3.3. 新增MariaDB使用者供外部使用

一般來說root使用者並不會開放非localhost連線使用，若您需要使用phpMyAdmin或是MySQL Workbench等圖形化介面登入，必須新增外部使用者。

執行`mysql -u root -p`，輸入剛剛所設定的密碼，進入MariaDB操作：
```bash
mysql -u root -p
```
新增一個帳號為`tutorial`，登入位置為`%`(皆可登入)，密碼為`tutorial2020`的使用者（自行替換帳號密碼與登入位置）：
```sql
CREATE USER 'tutorial'@'%' IDENTIFIED BY 'tutorial2020';
```
若需要給予`tutorial`帳號所有權限，可以執行：
```sql
GRANT ALL PRIVILEGES ON *.* TO 'tutorial'@'%' WITH GRANT OPTION;
```

### 4. phpMyAdmin

#### 4.1. 下載phpMyAdmin

phpMyAdmin為簡易好上手的MySQL/MariaDB圖形化介面，在開始安裝之前需先安裝`unzip`套件：
```bash
apt install unzip -y
```
切換至`/usr/share/`目錄，並下載需要版本的phpMyAdmin( https://www.phpmyadmin.net/downloads/ )，解壓縮後更名為`phpmyadmin`，此以phpMyAdmin 5.0.2全語言版本為例：
```bash
cd /usr/share
wget https://files.phpmyadmin.net/phpMyAdmin/5.0.2/phpMyAdmin-5.0.2-all-languages.zip
unzip phpMyAdmin-5.0.2-all-languages.zip
mv phpMyAdmin-5.0.2-all-languages phpmyadmin
rm phpMyAdmin-5.0.2-all-languages.zip
```

#### 4.2. 切換phpMyAdmin使用者

將`phpmyadmin`資料夾的使用者群組、名稱切換為`www-data:www-data`：
```bash
chown -R www-data:www-data phpmyadmin/
```
使用`ll`或`ls -lah`指令查看使用者與群組、名稱：
```bash
ll
```

#### 4.3. 設定Nginx Router

新增一個snippet名為`phpmyadmin.conf`位於`/etc/nginx/snippets/`：
```bash
nano /etc/nginx/snippets/phpmyadmin.conf
```
內容為，記得將`unix:/run/php/php7.4-fpm.sock;`改為安裝的php版本：
```nginx
location /phpmyadmin { 
	root /usr/share/; 
	index index.php index.html index.htm; 
	location ~ ^/phpmyadmin/(.+\.php)$ { 
		try_files $uri =404; 
		root /usr/share/; 
		fastcgi_pass unix:/run/php/php7.4-fpm.sock; 
		fastcgi_index index.php; 
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; 
		include /etc/nginx/fastcgi_params; 
	} 
	location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ { 
		root /usr/share/; 
	} 
} 
```

將default router引入snippets設定檔：
```bash
nano /etc/nginx/sites-available/default
```
在`server { } `中底部新增include snippet：
```nginx
include snippets/phpmyadmin.conf;
```

驗證nginx語法是否有誤：
```bash
nginx -t
```

如果無誤，重新啟動nginx：
```sh
service nginx restart
```

重新連線至網站，並輸入`網址/phpmyadmin`，即可看到phpMyAdmin後台！

若希望只有特定IP位置可以存取此後台，可以在`/etc/nginx/snippets/phpmyadmin.conf`的`location /phpmyadmin { }`中加入：
```nginx
        allow 0.0.0.0;
        deny all;
```
將會只允許特定IP位置的電腦可以連線至phpMyAdmin後台，增加安全性。



| 相關資訊 | 連結 |
| ------ | ------ |
| Composer | https://getcomposer.org/download/ |
| MariaDB | https://downloads.mariadb.org/mariadb/repositories |
| phpMyAdmin | https://www.phpmyadmin.net/downloads/ |
