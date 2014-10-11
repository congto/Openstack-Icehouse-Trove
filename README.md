I'm working with Database-As-A-Service of Openstack, I followed docs at docs.openstack.org. After getting failed 2-3 times, I have decided to try with another guide. I goodle-ed and realised that there is no working tutorial on the Internet. So now I am writing this article to provide step-by-step installation guide to get Openstack Trove working (at least It was working in my LAB :lol:). So we will walk through 3 main parts :

**1. Building images for trove**
* [Download based images] (#based-images)
* [Install and configure trove-guestagent to communicate with trove-taskmanager and message queued](#trove-guestagent)
* [Sysprep and compress images](#sysprep)
* [Upload to glance](#upload-to-glance)
	
**2. Installing and configuring Openstack Databases Service**
* [Install and configure necessary packages] (#guest-pkgs)
* [Update database store for Trove] (#trove-datastore)

**3. Verifying**
* [Create trove instance](#create-trove-instance)
* [Create mysql user/database](#create-mysql-user-db)
* [Connect to mysql server using username:password was created.](#connect-mysql)

<a name="based-images"></a>
##1. Building images for trove
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
*Using virt-manager and libvirtd to create new virtual machine using trusty-server-cloudimg-amd64-disk1.img and my-seed.img as secondary hard disk (This is something beyond this article). After instance boot successfully, login to that virtual machine with* '**ubuntu:passw0rd**' *as username and password*

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
