I'm working with Database-As-A-Service of Openstack, I followed docs at docs.openstack.org. After getting failed 2-3 times, I have decided to try with another guide. I goodle-ed and realised that there is no working tutorial on the Internet. So now I am writing this article to provide step-by-step installation guide to get Openstack Trove working (at least It was working in my LAB :lol:). So we will walk through 3 main parts :

**1. Building images for trove**
	- [Download based images] (#based-images)
	- [Install and configure trove-guestagent to communicate with trove-taskmanager and message queued](#trove-guestagent)
	- [Sysprep and compress images](#sysprep)
	- [Upload to glance](#upload-to-glance)
	
**2. Installing and configuring Openstack Databases Service**
	- [Install and configure necessary packages] (#guest-pkgs)
	- [Update database store for Trove] (#trove-datastore)

**3. Verifying**
	- [Create trove instance](#create-trove-instance)
	- [Create mysql user/database](#create-mysql-user-db)
	- [Connect to mysql server using username:password was created.](#connect-mysql)

<a name="based-images"></a>
## Building images for trove
```
wget http://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
```
