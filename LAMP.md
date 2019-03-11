#!/bin/bash
ip=`ip addr show ens33 | grep "ens33$" | awk '{print $2}' | awk -F/ '{print $1}'`
mysqluser="testuser"
userpwd="123456"

tar zxf apr-util-1.5.4.tar.gz
tar zxf apr-1.5.2.tar.gz
tar zxf httpd-2.4.23.tar.gz
tar zxf zlib-1.2.8.tar.gz
tar zxf openssl-1.0.1u.tar.gz
tar zxf pcre-8.39.tar.gz
cd apr-1.5.2
./configure --prefix=/usr/local/apr && make && make install
cd ../apr-util-1.5.4
./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr/ && make && make install
cd ../pcre-8.39
./configure --prefix=/usr/local/pcre && make && make install
cd ../zlib-1.2.8
./configure --prefix=/usr/local/zlib && make && make install
cd ../openssl-1.0.1u
./config -fPIC --prefix=/usr/local/openssl enable-shared && make && make install
mv /usr/bin/openssl /usr/bin/openssl.1.0.1e
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
cd ../httpd-2.4.23
./configure --prefix=/usr/local/http --enable-so --enable-cgi --enable-cgid --enable-ssl --with-ssl=/usr/local/openssl --enable-rewrite --with-pcre=/usr/local/pcre/ --with-z=/usr/local/zlib/ --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util/ --enable-modules=most --enable-mods-shared=most --enable-mpms-shared=all --with-mpm=event --enable-proxy --enable-proxy-fcgi --enable-expires --enable-deflate && make && make install
ln -s /usr/local/http/bin/* /usr/local/bin/
sed -i '200c ServerName www.benet.com:80' /usr/local/http/conf/httpd.conf
/usr/local/http/bin/apachectl start
cp /usr/local/http/bin/apachectl /etc/init.d/httpd
sed -i '1 a #chkconfig: 35 85 15\n#description: apache 2.4.23' /etc/init.d/httpd
sed -i "474c Include conf/extra/httpd-vhosts.conf" /usr/local/http/conf/httpd.conf
sed -i "116c LoadModule proxy_module modules/mod_proxy.so" /usr/local/http/conf/httpd.conf
sed -i "120c LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so" /usr/local/http/conf/httpd.conf
apachectl -M |grep proxy
sed -i "386a AddType application/x-httpd-php .php\nAddType application/x-httpd-source-php .phps" /usr/local/http/conf/httpd.conf
sed -i "258c DirectoryIndex index.php index.html" /usr/local/http/conf/httpd.conf
sed -i "23,39d" /usr/local/http/conf/extra/httpd-vhosts.conf
cat >> /usr/local/http/conf/extra/httpd-vhosts.conf << EOF
<VirtualHost *:80>
ServerAdmin webmaster@benet.com
DocumentRoot "/var/www/benet"
ServerName www.benet.com
ServerAlias benet.com
ErrorLog "logs/benet.com-error_log"
CustomLog "logs/benet.com-access_log" common
ProxyRequests Off
ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://${ip}:9000/var/www/benet/$1
<Directory "/var/www/benet">
Options FollowSymLinks
AllowOverride None
Require all granted
</Directory>
</VirtualHost>
EOF
chkconfig --add httpd
chkconfig httpd on
systemctl restart httpd
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload

echo "----------------------------Finshed--apache-------------------------------------------"

cd /root/
rpm -e --nodeps mariadb-libs
tar zxf mysql-5.7.22-linux-glibc2.12-x86_64.tar.gz 
mv mysql-5.7.22-linux-glibc2.12-x86_64 /usr/local/mysql
mkdir /usr/local/mysql/data
groupadd -r mysql && useradd -r -g mysql -s /sbin/false -M mysql
chown -R mysql:mysql /usr/local/mysql
ln -s /usr/local/mysql/bin/* /usr/local/bin
cat <<EOF>/etc/my.cnf
[mysqld]
basedir=/usr/local/mysql/
datadir=/usr/local/mysql/data/
pid-file=/usr/local/mysql/data/mysqld.pid
socket=/usr/local/mysql/mysql.sock
log-error=/usr/local/mysql/data/mysqld.err
server_id=1
[client]
socket=/usr/local/mysql/mysql.sock
EOF
mysqld --initialize --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data/
mypwd=`grep "password" /usr/local/mysql/data/mysqld.err | awk -F'root@localhost: ' '{print $2}'`
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
/etc/init.d/mysqld start
mysql -u root -p${mypwd} -e "alter user root@localhost identified by '123'" --connect-expired-password
mysql -uroot -p123 -e "grant all on *.* to ${mysqluser}@'%' identified by '${userpwd}'"
mysql -uroot -p123 -e "flush privileges"
mysql -uroot -p123 -e "create database bbsdb"
mysql -uroot -p123 -e "grant all on bbsdb.* to runbbs@'%' identified by '123'"
/etc/init.d/mysqld restart
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --reload

echo "----------------------------Finshed--MySQL-------------------------------------------"

cd /root/
yum -y install libxml2-devel libcurl-devel openssl-devel bzip2-devel
tar zxf libmcrypt-2.5.7.tar.gz
tar zxf php-5.6.27.tar.gz
cd libmcrypt-2.5.7/
./configure --prefix=/usr/local/libmcrypt && make && make install
cd ../php-5.6.27
./configure --prefix=/usr/local/php5.6 --with-mysql=mysqlnd --with-pdo-mysql=mysqlnd --with-mysqli=mysqlnd --with-openssl --enable-fpm --enable-sockets --enable-sysvshm --enable-mbstring --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir=/usr --enable-xml --with-mhash --with-mcrypt=/usr/local/libmcrypt --with-config-file-path=/etc --with-config-file-scan-dir=/etc/php.d --with-bz2 --enable-maintainer-zts && make && make install
cp php.ini-production /etc/php.ini
cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm
chkconfig --add php-fpm
chkconfig php-fpm on
cp /usr/local/php5.6/etc/php-fpm.conf.default /usr/local/php5.6/etc/php-fpm.conf
sed -i "s/;pid/pid/" /usr/local/php5.6/etc/php-fpm.conf
sed -i "s/127.0.0.1/${ip}/" /usr/local/php5.6/etc/php-fpm.conf
sed -i "235c pm.max_children = 50" /usr/local/php5.6/etc/php-fpm.conf
sed -i "240c pm.start_servers = 5" /usr/local/php5.6/etc/php-fpm.conf
sed -i "245c pm.min_spare_servers = 5" /usr/local/php5.6/etc/php-fpm.conf
sed -i "250c pm.max_spare_servers = 35" /usr/local/php5.6/etc/php-fpm.conf
/etc/init.d/php-fpm start
firewall-cmd --add-port=9000/tcp --permanent
firewall-cmd --reload
mkdir -p /var/www/benet
cat > /var/www/benet/index.php << EOF
<?php
phpinfo();
?>
EOF
cat > /var/www/benet/test.php << EOF
<?php
\$link=mysql_connect('${ip}','${mysqluser}','${userpwd}');
if (\$link) echo "connection success .......";
mysql_close();
?>
EOF


echo "------------------------Finshed--PHP------------------------------------"
