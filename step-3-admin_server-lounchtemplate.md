## Admin server to prepare SQL and EFS for wordpress.

# create lounch template or new instance
Amazon Linux 2023 AMI 2023.9.20251014.0 x86_64 HVM kernel-6.1
Needs to have inbound SSH


## Lounch template user data
```bash
#!/bin/bash
exec > >(tee -a /var/log/user-data.log) 2>&1
set -Eeuo pipefail

EFS_ID="fs-xxxxxxxxx"
EFS_AP="fsap-xxxxxxxxxx"

dnf -y upgrade --refresh --allowerasing
dnf -y install --allowerasing \
  nginx \
  php php-fpm php-mysqlnd php-json php-mbstring php-xml php-gd php-curl \
  amazon-efs-utils nfs-utils unzip tar rsync

systemctl enable php-fpm nginx
systemctl start php-fpm nginx

mkdir -p /var/www/html
if ! grep -q " ${EFS_ID}:/ /var/www/html " /etc/fstab; then
  echo "${EFS_ID}:/ /var/www/html efs _netdev,tls,accesspoint=${EFS_AP},noresvport 0 0" >> /etc/fstab
fi
for i in {1..6}; do mount -a && break || sleep 5; done

cat >/etc/nginx/conf.d/wordpress.conf <<'NGINX'
server {
  listen 80 default_server;
  server_name _;
  root /var/www/html;
  location = /health { return 200 'ok'; add_header Content-Type text/plain; }
  index index.php index.html;
  location / { try_files $uri $uri/ /index.php?$args; }
  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php-fpm/www.sock;
  }
}
NGINX

sed -i 's/^user = .*/user = nginx/'  /etc/php-fpm.d/www.conf
sed -i 's/^group = .*/group = nginx/' /etc/php-fpm.d/www.conf
nginx -t && systemctl restart php-fpm nginx
echo ok >/var/www/html/health || true
```




## After startup. Folowwed config is needed to be run
SSH into admin instance


```bash
DB_ENDPOINT="<rds-endpoint>"
DB_NAME="wordpress"
DB_USER="wpuser"
DB_PASS="your-strong-pass"
SITE_URL="http://placeholder.local"
SITE_TITLE="Mead"

# WordPress files 
if [ ! -f /var/www/html/wp-settings.php ]; then
  cd /tmp && curl -fsSL -o wp.tgz https://wordpress.org/latest.tar.gz
  tar xzf wp.tgz && rsync -a wordpress/ /var/www/html/
fi

# wp-cli + DB client and ensure DB exists
sudo test -x /usr/local/bin/wp || { sudo curl -fsSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && sudo chmod +x /usr/local/bin/wp; }
sudo dnf -y install mariadb105 || sudo dnf -y install mariadb
mysql -h "$DB_ENDPOINT" -u "$DB_USER" -p"$DB_PASS" -e "CREATE DATABASE IF NOT EXISTS \`$DB_NAME\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# wp-config och install
sudo -u nginx wp config create \
  --path=/var/www/html \
  --dbname="$DB_NAME" --dbuser="$DB_USER" --dbpass="$DB_PASS" --dbhost="$DB_ENDPOINT" \
  --skip-check --force

sudo -u nginx wp config set WP_HOME    "$SITE_URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set WP_SITEURL "$SITE_URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set FS_METHOD  "direct"    --type=constant --path=/var/www/html

sudo -u nginx wp core is-installed --path=/var/www/html || sudo -u nginx wp core install \
  --url="$SITE_URL" --title="$SITE_TITLE" \
  --admin_user="wpadmin" --admin_password="1337strongpaasword1929" --admin_email="jasonburne@example.com" \
  --skip-email --path=/var/www/html
```



## OBS important:
After step 4, when ALB has been created. Get the ALB DNS link.
Earlyer site title need to be updated
(placeholder)
From admin instance
```bash
URL="http://web-serverALBdns"

sudo -u nginx wp config set WP_HOME    "$URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set WP_SITEURL "$URL" --type=constant --path=/var/www/html
sudo -u nginx wp option update home    "$URL" --path=/var/www/html
sudo -u nginx wp option update siteurl "$URL" --path=/var/www/html
sudo -u nginx wp rewrite structure '/%postname%/' --hard --path=/var/www/html
sudo -u nginx wp rewrite flush --hard --path=/var/www/html
``` 

