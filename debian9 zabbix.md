# debian9 zabbix

##1

```
# wget http://repo.zabbix.com/zabbix/3.4/debian/pool/main/z/zabbix-release/zabbix-release_3.4-1+jessie_all.deb
# dpkg -i zabbix-release_3.4-1+stretch_all.deb
# apt-get update
```
##2
```
# apt-get install zabbix-server-mysql zabbix-frontend-php
```
##3
```
# cd /usr/share/doc/zabbix-server-mysql
# zcat create.sql.gz | mysql -uroot zabbix

# vi /etc/zabbix/zabbix_server.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
```