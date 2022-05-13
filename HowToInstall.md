---
> 2022/2
> https://docs.nextcloud.com/server/latest/admin_manual/contents.html
> Version : 23
---

# 安裝流程
## 先在 PVE 開好虛擬機
- OS : Ubuntu-20.04.3-server-amd64
- CPU : host
- Memory : 512MB/4GB
- BIOS : Default(SeaBIOS)
- 機器架構 : Default(i440fx)
    
## 照著手冊中的 Example installation on Ubuntu 20.04 LTS 操作
### 先安裝一些基本套件
```
sudo apt update
sudo apt install apache2 mariadb-server libapache2-mod-php7.4
sudo apt install php7.4-gd php7.4-mysql php7.4-curl php7.4-mbstring php7.4-intl
sudo apt install php7.4-gmp php7.4-bcmath php-imagick php7.4-xml php7.4-zip
```

### 啟動 MySQL，並設定 root 使用者密碼並以 root 使用者登入
```
sudo /etc/init.d/mysql start    # 啟動 MySQL 服務
sudo mysql -uroot -p            # 以 root 使用者登入
# 第一次以 root 登入時，會提示設定登入密碼
# 不輸入任何密碼直接 enter 進入 == 不設定密碼
```
這時應該會出現 `MariaDB [root]>` 在畫面中
> 雖然我實際運作看到的是 `MariaDB [none]>`

### 建立普通使用者並初始化資料庫
```
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
# username/password 可以修改成自己喜歡的，不過我直接照抄
quit;
```

### 下載 Nextcloud 壓縮檔
1. 下載本體
```
wget https://download.nextcloud.com/server/releases/nextcloud-23.0.2.tar.bz2
```

2. 下載驗證碼(可略過)
```
wget https://download.nextcloud.com/server/releases/nextcloud-23.0.2.tar.bz2.md5
```

3. 驗證壓縮檔(可略過)
```
md5sum -c nextcloud-23.0.2.tar.bz2.md5 < nextcloud-23.0.2.tar.bz2
```

4. 解壓縮本體
```
tar -xjvf nextcloud-23.0.2.tar.bz2
```

5. 將解壓出來的本體移動到 Apache HTTP server 操作的目錄，以 Ubuntu 為例
```
cp -r nextcloud /var/www
```
    
### 依照安裝指示，開始設定 apache server
1. 在 /etc/apache2/sites-available 建立 nextcloud.conf 檔案
```
sudo vim /etc/apache2/sites-available/nextcloud.conf
```

2. 將網站建議內容複製貼上
> use the virtual host installation
```
<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud/
    ServerName  cloud.bouzzing.com      # your.server.com

<Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
    Satisfy Any

    <IfModule mod_dav.c>
        Dav off
    </IfModule>
    </Directory>
</VirtualHost>
```

3. 啟動上一步寫好的設定檔
```
a2ensite nextcloud.conf
```

4. 啟動各種 apache madule
```
a2enmod rewrite
a2enmod headers
a2enmod env
a2enmod dir
a2enmod mime
service apache2 restart     # 重新啟動 apache 讓剛剛啟動的 module 生效
```
    
### 啟動與使用安裝引導
1. 保證權限設定正確
```
chown -R www-data:www-data /var/www/nextcloud/
```

2. 啟動瀏覽器，連線到 http://[IP of nextcloud]/nextcloud
3. 設定管理員帳密
4. 照著瀏覽器設定，大概就沒問題了
    
# 各種疑難雜症小技巧
## config.php 的部分
> /var/www/nextcloud/config/config.php
```
'default_phone_region' => 'TW',         # 電話預設
'default_language' => 'TW',             # 語言預設
'default_locale' => 'zh_TW',            # 地區預設(但不知道為啥不起作用)
'memcache.local' => '\\OC\\Memcache\\APCu',     # 啟用快取(有三種，居家用這個就可以了)
'filesystem_check_changes' => 1,        # 會自動檢查檔案變更，方便不透過 Nextcloud 操作檔案
'htaccess.RewriteBase' => '/',          # 網址美觀，新增後要執行 occ maintenance:update:htaccess
'defaultapp' => 'files,dashboard',      # 預設開啟頁面
```

## 系統 Crontab 的部分
1. 先用管理員到網頁的基本設定中，將背景工作設為 Cron
2. 在 Ubuntu 中設定 crontab (嘗試用過 /etc/crontab，沒有效果)
```
sudo crontab -u www-data -e
```
3. 加入以下任務設定
```
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php --define apc.enable_cli=1
```

## SSL 證書的部分
待補，TBD

## 資料部分單獨掛接
1. 在 TureNAS 設定好 www-data 的使用者(33)以及群組(33)
2. 在 TureNAS 開好 NFS 分享，並將權限設定為 www-data:www-data 770
3. 在 nextcloud 中掛上剛剛分享出來的 NFS
```
sudo mount -o bg,soft,rsize=32768,wsize=32768 192.168.0.100:/mnt/OU_House/pve_Nc_data /var/www/nextcloud/data
```

## 安裝 wiki 之後跳出錯誤
- 有兩個檔有相同設定，就乾脆通通設了
### 大概長這樣
```
The PHP OPcache module is not properly configured
The OPcache interned strings buffer is nearly full. To assure that repeating strings can be effectively cached, it is recommended to apply opcache.interned_strings_buffer to your PHP configuration with a value higher than 8
```
### 修理方式
```
sudo vim /etc/php/7.4/apache2/php.ini
sudo vim /etc/php/7.4/cli/php.ini
```
### 照著建議新增
- 原本是註解掉的
- 從 8 改成 16
```
opcache.interned_strings_buffer=16
```
### 重啟 apache2
```
systemctl restart apache2
```

## 檔案靠 scp 上傳後在網頁找不到
- 使用 occ 重新掃描檔案
```
cd /var/www/nextcloud
sudo -u www-data php --define apc.enable_cli=1 occ  # 全部操作介紹
sudo -u www-data php --define apc.enable_cli=1 occ files:scan <user_id>
```