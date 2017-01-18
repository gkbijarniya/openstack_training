# Single node installation of Openstack

## Initial setup 
###  Do everything as a "root"

```
sudo su
```

### Update dist-upgrade and add cloud-archive liberty

```
apt-get update && apt-get -y dist-upgrade
add-apt-repository cloud-archive:liberty
apt-get update && apt-get -y dist-upgrade
```

### Install mysql rabbitmq , ntp 
```
apt-get install -y mariadb-server python-pymysql
apt-get install rabbitmq-server
apt-get install ntp -y
```
### Install keystone
```
apt-get install -y keystone apache2 libapache2-mod-wsgi memcached python-memcache python-openstackclient
```
### Install glance
```
apt-get install -y glance python-glanceclient
```
### Get the image
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```


### Install nova
```
apt-get install -y nova-compute sysfsutils nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient nova-console qemu-kvm
```


### Install neutron
```
apt-get install -y neutron-server neutron-plugin-openvswitch neutron-plugin-openvswitch-agent neutron-common neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent openvswitch-switch
```

### Install ciner
```
apt-get install -y cinder-api cinder-scheduler cinder-volume lvm2 open-iscsi-utils open-iscsi iscsitarget sysfsutils python-cinderclient
```


### Install dashboard
```
apt-get install -y openstack-dashboard
```




### create openstack user for rabbitmq and give all permission
```
rabbitmqctl add_user openstack password
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```


### Configure mysql
```
vi /etc/mysql/conf.d/mysqld_openstack.cnf
```
```
[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```

```
service mysql restart
```

# Add System specific info
```
vi  /etc/sysctl.conf
```

```
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

```
sysctl -p
```

### Connect to myql and create database
```
mysql -u root -p
```
```
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
```
```
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
```
```
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
```

```
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES  ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
```

```
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'password';
```

```
flush privileges;
quit
```
### Check tables of keystone database

```
mysql -u keystone -p
show databases;
use keystone;
show tables;
```



### Change keystone config
```
vi  /etc/keystone/keystone.conf
```

```
[DEFAULT]
...
admin_token = password

[database]
...
connection = mysql+pymysql://keystone:password@172.31.22.152/keystone
[memcache]
...
servers = localhost:11211
[token]
...
provider = uuid
driver = memcache
[revoke]
...
driver = sql
```


```
echo "manual" > /etc/init/keystone.override
```

### Update apache config
```
vi /etc/apache2/sites-available/wsgi-keystone.conf
```

```
Listen 5000
Listen 35357
<VirtualHost *:5000>
WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
WSGIProcessGroup keystone-public
WSGIScriptAlias / /usr/bin/keystone-wsgi-public
WSGIApplicationGroup %{GLOBAL}
WSGIPassAuthorization On
<IfVersion >= 2.4>
ErrorLogFormat "%{cu}t %M"
</IfVersion>
ErrorLog /var/log/apache2/keystone.log
CustomLog /var/log/apache2/keystone_access.log combined
<Directory /usr/bin>
<IfVersion >= 2.4>
Require all granted
</IfVersion>
<IfVersion < 2.4>
Order allow,deny
Allow from all
</IfVersion>
</Directory>
</VirtualHost>
<VirtualHost *:35357>
WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
WSGIProcessGroup keystone-admin
WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
WSGIApplicationGroup %{GLOBAL}
WSGIPassAuthorization On
<IfVersion >= 2.4>
ErrorLogFormat "%{cu}t %M"
</IfVersion>
ErrorLog /var/log/apache2/keystone.log
CustomLog /var/log/apache2/keystone_access.log combined
<Directory /usr/bin>
<IfVersion >= 2.4>
Require all granted
</IfVersion>
<IfVersion < 2.4>
Order allow,deny
Allow from all
</IfVersion>
</Directory>
</VirtualHost>
```


#### Create a symbolic link
```
ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```

### Populate the keystone db
```
keystone-manage db_sync
```

### Set the variables
```
export OS_TOKEN=password
export OS_URL=http://192.168.1.19:35357/v3
export OS_IDENTITY_API_VERSION=3
```

## Start creating the services 
### Create keystone service of identity type
```
openstack service create --name keystone --description "OpenStack Identity" identity

openstack endpoint create --region RegionOne identity public http://172.31.22.152:5000/v2.0

openstack endpoint create --region RegionOne identity internal http://192.168.1.19:5000/v2.0

openstack endpoint create --region RegionOne identity admin http://192.168.1.19:35357/v2.0
```


### Create project
```
openstack project create --domain default --description "Admin Project" admin
openstack user create --domain default --password-prompt admin
openstack role create admin
openstack role add --project admin --user admin admin
```

### Create project service
```
openstack project create --domain default --description "Service Project" service
```

```
unset OS_TOKEN OS_URL
```

### Put all the variables in creds file 
```
vi creds
```

```
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.1.19:35357/v3
export OS_IDENTITY_API_VERSION=3
```

```
source creds
```

### Issue a token and confirm it look good
```
openstack token issue
```

### Create glance
```
openstack user create --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image service" image
openstack endpoint create --region RegionOne image public http://192.168.1.19:9292
openstack endpoint create --region RegionOne image internal http://192.168.1.19:9292
openstack endpoint create --region RegionOne image admin http://192.168.1.19:9292
```



### Create nova
```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://192.168.1.19:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://192.168.1.19:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://192.168.1.19:8774/v2/%\(tenant_id\)s
```
### Create neutron
```
openstack user create --domain default --password-prompt neutron

openstack role add --project service --user neutron admin

openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://192.168.1.19:9696
openstack endpoint create --region RegionOne network internal http://192.168.1.19:9696
openstack endpoint create --region RegionOne network admin http://192.168.1.19:9696
```



### Create cinder
```
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinder --description "OpenStack Block Storage" volume
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack endpoint create --region RegionOne volume public http://192.168.1.19:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume admin http://192.168.1.19:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume internal http://192.168.1.19:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 public http://192.168.1.19:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://192.168.1.19:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://192.168.1.19:8776/v2/%\(tenant_id\)s
```
