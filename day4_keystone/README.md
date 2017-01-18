# Single node installation of Openstack

## Initial setup 
###  Do everything as a "root"

```
sudo su - 
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

### Restart apache2
```
service apache2 restart
```
Note : apache2 restart will fail if keystone is aleady running. 
check keystone status as
``` 
service keystone status
```
shows keystone start/running, process 18149
Kill keystone and restart apache2
```
service keystone stop
```
### Test the setup
Use public IP to get the console
```
http://35.156.212.69/horizon/auth/login/?next=/horizon/
```
or 
```
http://35.156.212.69/
```

### Set the variables
```
export OS_TOKEN=password
export OS_URL=http://172.31.22.152:35357/v3
export OS_IDENTITY_API_VERSION=3
```
```
echo $OS_TOKEN $OS_URL $OS_IDENTITY_API_VERSION
```
output - password http://172.31.22.152:35357/v3 3

## Start creating the services 
### Create keystone service of identity type
```
openstack service create --name keystone --description "OpenStack Identity" identity

openstack endpoint create --region RegionOne identity public http://172.31.22.152:5000/v2.0

openstack endpoint create --region RegionOne identity internal http://172.31.22.152:5000/v2.0

openstack endpoint create --region RegionOne identity admin http://172.31.22.152:35357/v2.0
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
export OS_AUTH_URL=http://172.31.22.152:35357/v3
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
openstack endpoint create --region RegionOne image public http://172.31.22.152:9292
openstack endpoint create --region RegionOne image internal http://172.31.22.152:9292
openstack endpoint create --region RegionOne image admin http://172.31.22.152:9292
```



### Create nova
```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://172.31.22.152:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://172.31.22.152:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://172.31.22.152:8774/v2/%\(tenant_id\)s
```
### Create neutron
```
openstack user create --domain default --password-prompt neutron

openstack role add --project service --user neutron admin

openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://172.31.22.152:9696
openstack endpoint create --region RegionOne network internal http://172.31.22.152:9696
openstack endpoint create --region RegionOne network admin http://172.31.22.152:9696
```



### Create cinder
```
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinder --description "OpenStack Block Storage" volume
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack endpoint create --region RegionOne volume public http://172.31.22.152:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume admin http://172.31.22.152:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume internal http://172.31.22.152:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 public http://172.31.22.152:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://172.31.22.152:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://172.31.22.152:8776/v2/%\(tenant_id\)s
```

## Display project , user, role , service list
```
openstack
```
```
(openstack)  domain list
+---------+---------+---------+----------------------------------------------------------------------+
| ID      | Name    | Enabled | Description                                                          |
+---------+---------+---------+----------------------------------------------------------------------+
| default | Default | True    | Owns users and tenants (i.e. projects) available on Identity API v2. |
+---------+---------+---------+----------------------------------------------------------------------+
(openstack) project list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 8cbab92b27a2489e9e0c6d4013376b58 | service |
| a2b6dd2f6cbe4def9c5de5390fa7c931 | admin   |
+----------------------------------+---------+
(openstack) user list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 0389dd901a8b423dbe4a938a21a41b51 | glance  |
| 39148cb4a5b1495b9e0252b040e8b963 | admin   |
| 75e89036020a4b3b9af5f0fed8ab2be4 | nova    |
| b7a04ee2231d4d24ac1898e641dd769b | cinder  |
| e6fd2b8403f0404c8ee8f6942a485afe | neutron |
+----------------------------------+---------+
(openstack) role list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 6d12acbb474d410a92ade839f98eb30f | admin |
+----------------------------------+-------+
(openstack) service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 010d1a3056f04a42bee9f8934e21af83 | neutron  | network  |
| 3fbbfbf0829248388215447e192d5b74 | keystone | identity |
| 4e4e812609054274a6af8cefc7266835 | cinder   | volume   |
| 56a476de8a0144b896f77ca6aac753ee | cinderv2 | volumev2 |
| 6b1e39b39e134412b9148575d12cfce2 | nova     | compute  |
| dfca6dab9fee467696ef21e4abd752c0 | glance   | image    |
+----------------------------------+----------+----------+
(openstack) endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+--------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                        |
+----------------------------------+-----------+--------------+--------------+---------+-----------+--------------------------------------------+
| 0f7497a1aa6943a5a416d6e199d1de06 | RegionOne | cinder       | volume       | True    | internal  | http://172.31.22.152:8776/v1/%(tenant_id)s |
| 2004a375bf594edda77daefce010dd66 | RegionOne | cinderv2     | volumev2     | True    | admin     | http://172.31.22.152:8776/v2/%(tenant_id)s |
| 21242711672c422387092474057fbe3d | RegionOne | cinderv2     | volumev2     | True    | public    | http://172.31.22.152:8776/v2/%(tenant_id)s |
| 3d424dba5a4642af81e82058778ae0e0 | RegionOne | cinderv2     | volumev2     | True    | internal  | http://172.31.22.152:8776/v2/%(tenant_id)s |
| 3e2e8ebd129b42139c30211d9949158f | RegionOne | glance       | image        | True    | internal  | http://172.31.22.152:9292                  |
| 528e4c5accda40408e01aba93c056718 | RegionOne | cinder       | volume       | True    | admin     | http://172.31.22.152:8776/v1/%(tenant_id)s |
| 5fcf9e4b4c214231b9edf56d1ecbb473 | RegionOne | nova         | compute      | True    | public    | http://172.31.22.152:8774/v2/%(tenant_id)s |
| 812ff1bc0ab149049363425a7ebaf56a | RegionOne | glance       | image        | True    | admin     | http://172.31.22.152:9292                  |
| 88dc4bd30f784ac280e3197c36acefa8 | RegionOne | nova         | compute      | True    | admin     | http://172.31.22.152:8774/v2/%(tenant_id)s |
| 8bca4e333d364b0395dca1689ffecc5b | RegionOne | keystone     | identity     | True    | internal  | http://172.31.22.152:5000/v2.0             |
| 8f75e595d0bd4fed9a08abfd623196ad | RegionOne | keystone     | identity     | True    | admin     | http://172.31.22.152:35357/v2.0            |
| 9181725f69f94fec9d7c7c809343c1c0 | RegionOne | neutron      | network      | True    | public    | http://172.31.22.152:9696                  |
| 96c37a86b9874d86b410250d5646d972 | RegionOne | nova         | compute      | True    | internal  | http://172.31.22.152:8774/v2/%(tenant_id)s |
| 9b861d83de2540a3bf0e24613eae57c4 | RegionOne | glance       | image        | True    | public    | http://172.31.22.152:9292                  |
| 9cd7c30d3d53454f9029b4c96c4c3944 | RegionOne | cinder       | volume       | True    | public    | http://172.31.22.152:8776/v1/%(tenant_id)s |
| d03c0cca16ea4783a9e81764aceeca94 | RegionOne | neutron      | network      | True    | internal  | http://172.31.22.152:9696                  |
| d1ff144c335b4e2c9ff782fb20de5028 | RegionOne | neutron      | network      | True    | admin     | http://172.31.22.152:9696                  |
| ff504fe5d9bc4bd9bdba421fecc9c98d | RegionOne | keystone     | identity     | True    | public    | http://172.31.22.152:5000/v2.0             |
+----------------------------------+-----------+--------------+--------------+---------+-----------+--------------------------------------------+
(openstack) 
```

## Installing glance 
### Update the glance-api 
```
vi /etc/glance/glance-api.conf 
```

```
[database]
comment #sqlite_db = /var/lib/glance/glance.sqlite
...
connection = mysql+pymysql://glance:password@172.31.22.152/glance

[keystone_authtoken]
...
auth_uri = http://172.31.22.152:5000
auth_url = http://172.31.22.152:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = password

[paste_deploy]
...
flavor = keystone

[glance_store]
...
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
### Update the glance-registry
```
vi /etc/glance/glance-registry.conf
```
```
[database]
...
connection = mysql+pymysql://glance:password@172.31.22.152/glance

[keystone_authtoken]
...
auth_uri = http://172.31.22.152:5000
auth_url = http://172.31.22.152:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = password

[paste_deploy]
...
flavor = keystone
```
#### Restart glance service
```
service glance-api restart
service glance-registry restart
```
### db_sync for glance
```
glance-manage db_sync
```
### Create the image at glance 
Get the image if not done
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```
Create the image
```
glance image-create --name "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility public --progress
```
```
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2017-01-18T09:11:01Z                 |
| disk_format      | qcow2                                |
| id               | c6473c1b-d3de-4ec7-9286-9243fb31250e |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros                               |
| owner            | a2b6dd2f6cbe4def9c5de5390fa7c931     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2017-01-18T09:11:05Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```
#### list the glance image list
```
root@ip-172-31-22-152:~# glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| c6473c1b-d3de-4ec7-9286-9243fb31250e | cirros |
+--------------------------------------+--------+
```
#### Issue - glance image creation is failing
* Check /var/log/glance/glance-api.log and found that keystone shows 401 for request
* Deleted glance user and re-created it
```
openstack user delete glance
openstack user create --password-prompt glance
openstack role add --project service --user glance admin
```
