I'm working with Database-As-A-Service project (Trove) of Openstack, I followed docs at docs.openstack.org. After getting failed 2-3 times, I have decided to try with another guide. I goodle-ed and realised that there is no working tutorial on the Internet. So now I am writing this article to provide step-by-step installation guide to get Openstack Trove working (at least It was working in my LAB :lol:). So we will walk through 3 main parts :

**1. Building images for trove**
* [Download based images] (#based-images)
* [Install and configure trove-guestagent to communicate with trove-taskmanager and message queued](#trove-guestagent)
* [Sysprep and compress images](#sysprep)
* [Upload to glance](#upload-to-glance)
	
**2. Installing and configuring Openstack Databases Service**
* [Install and configure necessary packages] (#trove-pkgs)
* [Update database store for Trove] (#trove-datastore)

**3. Verifying**
* [Create trove instance](#create-trove-instance)
* [Create mysql user/database](#create-mysql-user-db)
* [Connect to mysql server using username:password was created.](#connect-mysql)

<a name="based-images"></a>
##1. Building images for Trove
#### Download based image
```
wget http://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
```

#### Create [local datasource](https://help.ubuntu.com/community/UEC/Images#Ubuntu_Cloud_Guest_images_on_12.04_LTS_.28Precise.29_and_beyond_using_NoCloud) for instace's cloud-init
```
$ cat > my-user-data <<EOF
#cloud-config
password: 'passw0rd'
chpasswd: { expire: False }
ssh_pwauth: True
EOF

$ cloud-localds my-seed.img my-user-data
```

#### Boot to downloaded image
*Using virt-manager and libvirtd to create new virtual machine using* **trusty-server-cloudimg-amd64-disk1.img** *and* **my-seed.img** *as secondary hard disk (This is something beyond this article). After instance boot successfully, login to that virtual machine with* '**ubuntu:passw0rd**' *as username and password*

<a name="trove-guestagent"></a>
#### Install trove-guestagent 
```
(guest)$ sudo apt-get install rsync trove-guestagent -y
(guest)$ sudo vi /etc/init/trove-guestagent.conf
	--exec /usr/bin/trove-guestagent -- --config-file=/etc/guest_info --config-file=/etc/trove/trove-guestagent.conf --log-dir=/var/log/trove --logfile=guestagent.log
```

*(guest)$* **sudo vi /etc/trove/trove-guestagent.conf**

```
[DEFAULT]
rabbit_host = $RABBITMQ-SERVER
rabbit_password = $RABBITMQ-PASSWORD
rabbit_userid = guest
verbose = True
debug = False
bind_port = 8778
bind_host = 0.0.0.0
nova_proxy_admin_user = admin
nova_proxy_admin_pass = $ADMIN_PASSWORD
nova_proxy_admin_tenant_name = admin
trove_auth_url = http://$KEYSTONE_SERVER:35357/v2.0
control_exchange = trove
root_grant = ALL
root_grant_option = True
ignore_users = os_admin
ignore_dbs = lost+found, mysql, information_schema
```

**Fix package permission**
```
$(guest)$ sudo chmod 755 /var/log/trove
```

**Enable sudo permission for 'trove' user**
```
(guest)$ sudo visudo
	trove ALL = (ALL) NOPASSWD: ALL
```

**Poweroff Virtual Machine**
```
sudo init 0
```

<a name="sysprep"></a>
#### Preparing images for Openstack
**Remove 'hardware' information**
```
virt-sysprep -a trusty-server-cloudimg-amd64-disk1.img
```

**Compress QCOW image**
```
qemu-img convert -f qcow2 -O qcow2 -c trusty-server-cloudimg-amd64-disk1.img trusty-server-cloudimg-amd64-trove-mysql.qcow2
```
<a name="upload-to-glance"></a>
#### Upload image to Glance
```
glance --os-username admin --os-password $ADMIN_PASSWORD --os-tenant-name admin \
--os-auth-url http://$KEYSTONE_SERVER:35357/v2.0 \
image-create --name trusty-server-cloudimg-amd64-trove-mysql --public --disk-format qcow2 --owner admin < trusty-server-cloudimg-amd64-trove-mysql.qcow2
```

<a name="trove-pkgs"></a>
##2. Installing Database-As-A-Service
#### Install necessary pkgs (My Openstack Lab using CentOS 6.5)
```
yum install openstack-trove python-troveclient -y
```

#### Create keystone user for Trove
```
source openrc
keystone user-create --name=trove --pass=$TROVE_PASSWORD --email=trove@example.com
keystone user-role-add --user=trove --tenant=service --role=admin
keystone service-create --name=trove --type=database \
  --description="OpenStack Database Service"
keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ trove / {print $2}') \
  --publicurl=http://$TROVE_API_SERVER:8779/v1.0/%\(tenant_id\)s \
  --internalurl=http://$TROVE_API_SERVER:8779/v1.0/%\(tenant_id\)s \
  --adminurl=http://$TROVE_API_SERVER:8779/v1.0/%\(tenant_id\)s
```

#### Configuring Trove
**/etc/trove/trove.conf**
```
[DEFAULT]
trove_api_workers = 2
use_syslog = False
debug = False
verbose = True
default_datastore = mysql
sql_connection = mysql://trove:$TROVE_DBPASS@$MYSQL_SERVER/trove?charset=utf8
rabbit_password = $RABBITMQ-PASSWORD
api_extensions_path = /usr/lib/python2.6/site-packages/trove/extensions/routes
add_addresses = True
network_label_regex = ^GATEWAY	# My External network for VMs is: GATEWAY_NET
```

**/etc/trove/trove-taskmanager.conf**
```
[DEFAULT]
use_syslog = False
debug = True
trove_auth_url = http://$KEYSTONE_SERVER:35357/v2.0
nova_proxy_admin_pass = $ADMIN_PASSWORD
nova_proxy_admin_tenant_name = admin
nova_proxy_admin_user = admin
sql_connection = mysql://trove:$TROVE_DBPASS@$MYSQL_SERVER/trove?charset=utf8
taskmanager_manager = trove.taskmanager.manager.Manager
rabbit_password = $RABBITMQ-PASSWORD
```

**/etc/trove/trove-conductor.conf**
```
[DEFAULT]
use_syslog = False
debug = True
control_exchange = trove
trove_auth_url = http://$KEYSTONE_SERVER:35357/v2.0
nova_proxy_admin_pass = $ADMIN_PASSWORD
nova_proxy_admin_tenant_name = admin
nova_proxy_admin_user = admin
sql_connection = mysql://trove:$TROVE_DBPASS@$MYSQL_SERVER/trove?charset=utf8
rabbit_password = $RABBITMQ-PASSWORD
```

**/etc/trove/api-paste.ini**
```
[composite:trove]
use = call:trove.common.wsgi:versioned_urlmap
/: versions
/v1.0: troveapi

[app:versions]
paste.app_factory = trove.versions:app_factory

[pipeline:troveapi]
pipeline = faultwrapper authtoken authorization contextwrapper ratelimit extensions troveapp
#pipeline = debug extensions troveapp

[filter:extensions]
paste.filter_factory = trove.common.extensions:factory

[filter:authtoken]
signing_dir = /var/cache/trove
admin_password = $TROVE_PASSWORD
admin_user = trove
admin_tenant_name = service
admin_token = $ADMIN_TOKEN
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = $KEYSTONE_SERVER
auth_port = 35357
auth_protocol = http

[filter:authorization]
paste.filter_factory = trove.common.auth:AuthorizationMiddleware.factory

[filter:contextwrapper]
paste.filter_factory = trove.common.wsgi:ContextMiddleware.factory

[filter:faultwrapper]
paste.filter_factory = trove.common.wsgi:FaultWrapper.factory

[filter:ratelimit]
paste.filter_factory = trove.common.limits:RateLimitingMiddleware.factory

[app:troveapp]
paste.app_factory = trove.common.api:app_factory

#Add this filter to log request and response for debugging
[filter:debug]
paste.filter_factory = trove.common.wsgi:Debug
```

#### Create Database for Trove
```
#mysql -u root -p
msyql> CREATE DATABASE trove;
msyql> GRANT ALL PRIVILEGES ON trove.* TO trove@'localhost' IDENTIFIED BY "$TROVE_DBPASS";
msyql> GRANT ALL PRIVILEGES ON trove.* TO trove@'%' IDENTIFIED BY "$TROVE_DBPASS";
msyql> exit;
```
<a name="trove-datastore"></a>
#### Update Trove datastore
```
su -s /bin/sh -c "trove-manage db_sync" trove
su -s /bin/sh -c "trove-manage datastore_update mysql ''" trove

service openstack-trove-api restart
service openstack-trove-conductor restart
service openstack-trove-taskmanager restart
trove-manage datastore_version_update mysql mysql-5.5 mysql $(glance image-list | grep 'trusty-server-cloudimg-amd64-trove-mysql' | awk '{print $2}') mysql-server-5.5 1
trove-manage datastore_update mysql mysql-5.5
```

<a name="create-trove-instance"></a>
##3. Verifying
#### Create Trove instance
```
trove create trove-mysql-instance-1 2 --size 2 --datastore mysql 
trove list
+--------------------------------------+------------------------+-----------+-------------------+--------+-----------+------+
|                  id                  |          name          | datastore | datastore_version | status | flavor_id | size |
+--------------------------------------+------------------------+-----------+-------------------+--------+-----------+------+
| a45c2824-9764-4f81-ad79-1567e31f0cf0 | trove-mysql-instance-1 |   mysql   |     mysql-5.5     | BUILD  |     2     |  2   |
+--------------------------------------+------------------------+-----------+-------------------+--------+-----------+------+

# Waiting for a couple of minutes for creating and install pkgs
trove list
+--------------------------------------+------------------------+-----------+-------------------+--------+-----------+------+
|                  id                  |          name          | datastore | datastore_version | status | flavor_id | size |
+--------------------------------------+------------------------+-----------+-------------------+--------+-----------+------+
| a45c2824-9764-4f81-ad79-1567e31f0cf0 | trove-mysql-instance-1 |   mysql   |     mysql-5.5     | ACTIVE  |     2     |  2   |
+--------------------------------------+------------------------+-----------+-------------------+--------+-----------+------+

# Get Trove instance IP
trove show a45c2824-9764-4f81-ad79-1567e31f0cf0
+-------------------+--------------------------------------+
|      Property     |                Value                 |
+-------------------+--------------------------------------+
|      created      |         2014-10-11T06:08:22          |
|     datastore     |                mysql                 |
| datastore_version |              mysql-5.5               |
|       flavor      |                  2                   |
|         id        | a45c2824-9764-4f81-ad79-1567e31f0cf0 |
|         ip        |            10.10.80.87             |
|        name       |        trove-mysql-instance-1        |
|       status      |                ACTIVE                |
|      updated      |         2014-10-11T06:08:28          |
|       volume      |                  2                   |
|    volume_used    |                 0.17                 |
+-------------------+--------------------------------------+

```
<a name="create-mysql-user-db"></a>
#### Create MySQL Database/User
```
trove database-create a45c2824-9764-4f81-ad79-1567e31f0cf0 demo-db
trove user-create a45c2824-9764-4f81-ad79-1567e31f0cf0 demo-user passw0rd --databases demo-db
```

#### Verify
```
trove user-list a45c2824-9764-4f81-ad79-1567e31f0cf0
+-----------+------+-----------+
|    name   | host | databases |
+-----------+------+-----------+
| demo-user |  %   |  demo-db  |
+-----------+------+-----------+
```

<a name="connect-mysql"></a>
#### Connect to MySQL Server
```
mysql -h 10.10.80.87 -u demo-user -p'passw0rd' demo-db
```

#Conclusion
At the time I write this article, Trove support so well for mysql. You can create/delete user/database and you can grant privileges for user. Another datastore as cassandra/mongodb you have very little options to choose, you can only create Trove instance for cassandra/mongodb . I hope Trove dev team will release more and more funny options in the feature. I am thankful for all your works done for Opensource World, I hope I will be a part of Opensource developer community in some day instead of just using your softwares. 
