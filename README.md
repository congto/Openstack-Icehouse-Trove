I'm working with Database-As-A-Service of Openstack, I followed docs at docs.openstack.org. After getting failed 2-3 times, I have decided to try with another guide. I goodle-ed and realised that there is no working tutorial on the Internet. So now I am writing this article to provide step-by-step installation guide to get Openstack Trove working (at least It was working in my LAB :lol:). So we will walk through 3 main parts :

1. Building images for trove
	- Download based images
	- Install and configure trove-guestagent to communicate with trove-taskmanager and message queued
	- Sysprep and compress images
	- Upload to glance
	
2. Installing and configuring Openstack Databases Service
	- Install and configure necessary packages
	- Update database store for Trove

3. Verifying
	- Create trove instance
	- Create mysql user/database
	- Connect to mysql server using username:password was created.
