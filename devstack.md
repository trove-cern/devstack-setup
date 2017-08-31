* Some things will need to be changed according to your specific environment, IP addresses, etc.
* Create an access key for your virtual machine
```
$ ssh-keygen -t rsa -f openstack.key
$ cat openstack.pub.key
```
* Copy the public key contents to CERN openstack access and api keys page
   * Compute / Access & Security / Key Pairs / Import Key Pair
* Create a virtual machine in the CERN cloud
   * image = CC7 - x86_64 [2017-04-06] 
   * size = m2.xlarge, 8 VCPU, 14.6GB RAM (or higher).
   * keypair = openstack.key
* For my VM, the IP address is 188.185.76.47.
* The name is th-devstack
```
$ ssh -i openstack.key root@th-devstack.cern.ch
```
* Now add a user for working with devstack
```
$ sudo useradd -s /bin/bash -d /opt/stack -m stack
$ echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
```
* And switch to that user
```
$ sudo su - stack
```
* Clone devstack with git
```
$ sudo yum install git
$ git clone https://git.openstack.org/openstack-dev/devstack
$ cd devstack/
```
* Devstack is configured by the file local.conf
```
$ nano local.conf


---------------~/devstack/local.conf---------------
[[local|localrc]]
ADMIN_PASSWORD=verysecret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD


HOST_IP=188.185.76.47


# disable nova networking, enable neutron
disable_service n-net
enable_service q-svc q-agt q-dhcp q-l3 q-meta neutron
# enable necessary components for trove
enable_service rabbit mysql key


## Trove
enable_service trove tr-api tr-tmgr tr-cond
enable_plugin trove https://github.com/openstack/trove
enable_plugin python-troveclient https://github.com/openstack/python-troveclient
enable_plugin trove-dashboard http://git.openstack.org/openstack/trove-dashboard


## Swift
enable_service s-proxy s-object s-container s-account
SWIFT_HASH=da0868472f4d4308aad810342f94e3a7
-------------------------------------------
```


* A swift hash can be generated using python
   * import uuid
   * print(uuid.uuid4().hex)
* Or the swift hash can be commented out entirely and there is a prompt in the installation script to enter a key or leave blank to randomly generate one. After randomly generating one many times, I have now decided to include it in the script after I left the installation waiting for me to press enter for the entire duration of a two hour lecture.
* Then start devstack by running the included shell script.
```
$ ./stack.sh
```
* Take a well earned break. This script will require 30-40 minutes the first time it is run. It is undone by running ./unstack.sh and subsequent runs of stack.sh will take around 20 minutes.
* Configure firewall to open http/https ports and other ports required for devstack
```
$ sudo yum install system-config-firewall-tui
$ sudo system-config-firewall-tui
```
* Enable firewall
* Trust WWW and Secure WWW services
* Open ports for the following endpoints
* 6080:tcp        Horizon web consoles
* 5672:tcp        RabbitMQ
* 9696:tcp        Openstack network
* 8776:tcp        Openstack volume
* 8080:tcp        Object store (used for trove logs storage)
* 15555:tcp        Will be used by us
* 15556:tcp        Will be used by us
* Select eth+ in the trust and masquerade pages

  

* Horizon interface is now running and accessible, either at the IP address or at th-devstack.cern.ch, in the case of my VM name. Log in with the admin/verysecret username/password.
* The demo project will be used as it has networks/subnets preconfigured.
* If Trove is correctly installed, there will be a “Database” tab in the sidebar at the left, under “Project”
* Creating a database instance requires a datastore. A datastore is an image used to start up a virtual machine, that is preconfigured to connect to Trove and run the database server.
* There are a number of premade images available for different databases at http://tarballs.openstack.org/trove/images/ubuntu/
* I will be using the mysql image.
```
$ mkdir ~/images
$ cd ~/images
$ wget http://tarballs.openstack.org/trove/images/ubuntu/mysql.qcow2
```
* All images will need to be modified to work with our system. To modify a qcow2 image, it can be mounted to the filesystem.
```
$ sudo yum install libguestfs-tools
$ sudo mkdir -p /mnt/mysql
$ sudo guestmount -a ./mysql.qcow2 -m /dev/sda1 /mnt/mysql/
```
* We will make several modifications. The first will be to add an init script that creates a user with a password that we can log in to. This is incredibly useful for debugging because the Trove instance launch process does not allow you to specify a key pair for SSHing with.
```
$ perl -e 'print crypt("trove", "xy"),"\n"'
xykk0MUYC4PJE
$ sudo nano /mnt/mysql/etc/init/add-user.conf
----------------add-user.conf----------------
description "Add trove user"
author "TH"


start on runlevel [2345]


script
    sudo useradd -p "xykk0MUYC4PJE" -G sudo -m trove
end script
----------------------------------------------
```
* We now have a user “trove” with password “trove” that we can log in as through the Horizon instance console.
* During the startup of the DB instance, it downloads its configuration file and the trove code from the devstack host. That is, it takes a copy of the file /etc/trove/trove-guestagent.conf and the entire directory ~/trove.
* This takes place in another init script.
* Because it grabs the latest trove version from the devstack host machine, the python requirements for trove are newer than are available on the premade image. We will need to include a pip --upgrade in our init script.
* In the premade init script, it uses scp to copy the necessary files from the devstack host. Rather than mess around with passwords or keys, as a temporary workaround we will host those files on a simple HTTP server using python.
```
$ cd ~/trove
$ python -m SimpleHTTPServer 15555 > /dev/null 2>&1  &
$ cd /etc/trove
$ python -m SimpleHTTPServer 15556 > /dev/null 2>&1  &
```
* This runs the HTTP servers in the background and silences the output
* The instance can then use wget to download the necessary files without worrying about authentication. There’s probably a better way to use in production. The files could be moved into the image itself, as long as they don’t need to be changed. For now, changing the configuration files will probably be inevitable, so we will keep it like this.
* The init script for the guestagent is /mnt/mysql/etc/init/trove-guest.conf
* The modifications needed are:
   * Update python library passlib to latest version.
   * Replace file retrieval method in two places
* Additions are in blue, deletions are in red.
```
$ sudo nano /mnt/mysql/etc/init/trove-guest.conf
---------------/mnt/mysql/etc/init/trove-guest.conf---------------
description "Trove Guest"
author "Auto-Gen"


start on (filesystem and net-device-up IFACE!=lo)
stop on runlevel [016]
chdir /var/run
pre-start script
    mkdir -p /var/run/trove
    chown ubuntu:root /var/run/trove/


    mkdir -p /var/lock/trove
    chown ubuntu:root /var/lock/trove/


    mkdir -p /var/log/trove/
    chown ubuntu:root /var/log/trove/


    # Copy the trove source from the user's development environment
    if [ ! -d /home/ubuntu/trove ]; then
        sudo -u ubuntu rsync -e 'ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no' -avz -$
        sudo wget -q -r -P /home/ubuntu/trove --no-parent --reject "index.html" -nH http://188.185.76.47:15555/
    fi


    # Ensure conf dir exists and is readable
    mkdir -p /etc/trove/conf.d
    chmod -R +r /etc/trove
    exec su -c "sudo -H pip install --upgrade passlib" ubuntu
end script


script
    # For backwards compatibility until https://review.openstack.org/#/c/100381 merges
    TROVE_CONFIG="--config-dir=/etc/trove/conf.d"
    if [ ! -f /etc/trove/conf.d/guest_info ] && [ ! -f /etc/trove/conf.d/trove-guestagent.conf ]; then


        chmod +r /etc/guest_info
        sudo -u ubuntu rsync -e 'ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no' -avz -$
        sudo wget -q -P /home/ubuntu --no-parent --reject "index.html" -nH http://188.185.76.47:15556/trove-guestagent.conf
        mv ~ubuntu/trove-guestagent.conf /etc/trove/trove-guestagent.conf
        TROVE_CONFIG="--config-file=/etc/guest_info --config-file=/etc/trove/trove-guestagent.conf"


    fi


    exec su -c "/home/ubuntu/trove/contrib/trove-guestagent $TROVE_CONFIG" 
ubuntu
    exec su -c "sudo python /home/ubuntu/trove/contrib/trove-guestagent $TROVE_CONFIG" 
ubuntu
end script
------------------------------------------------------------------
```
* Be sure to change the IP address in the wget lines to your own.
* The image can now be unmounted.
```
$ sudo guestunmount /mnt/mysql
```
* The image is now ready to be uploaded to glance and set up as a datastore for trove. If the image is changed again after this, the image on glance must be deleted and the new version must be uploaded using these same steps again.
```
$ cd ~/devstack
$ source openrc admin demo
$ cd ~/images
```
* Upload the image to glance
```
$ glance image-create --name mysql --disk-format qcow2 --container-format bare --visibility public < ./mysql.qcow2
```
* Copy its id to a variable
```
$ export IMAGE_ID=..................................
```
* Add the datastore, the empty string is the name or id of the default datastore for this database type. Since this is a new database type, it is left empty.
```
$ trove-manage datastore_update mysql ""
```
* Update mysql datastore to add new image, which is mysql version 5.6. Args are datastore, version_name, manager, image_id, packages, active
```
$ trove-manage datastore_version_update mysql 5.6 mysql $IMAGE_ID "mysql-server-5.6" 1
```
* Set default mysql version to use the 5.6 datastore
```
$ trove-manage datastore_update mysql 5.6
```
* Load config parameters for datastore
```
$ trove-manage db_load_datastore_config_parameters mysql 5.6 ~/trove/trove/templates/mysql/validation-rules.json
```


```
$ glance image-list
$ trove datastore-list
$ trove datastore-show mysql
```
* Image id is still available if image needs to be deleted.
```
$ glance image-delete $IMAGE_ID
```


* Now the configuration file that is loaded onto the instance must be edited.
```
$ nano /etc/trove/trove-guestagent.conf
```
* I spent a while messing with a few of these settings, but in the end the only one that actually needs to be changed from the auto generated defaults is the rabbit host ip.
```
---------------/etc/trove/trove-guestagent.conf---------------
[DEFAULT]
logging_exception_prefix = %(color)s%(asctime)s.%(msecs)03d TRACE %(name)s ^[[01;35m%(instance)s^[[00m
logging_debug_format_suffix = ^[[00;33mfrom (pid=%(process)d) %(funcName)s %(pathname)s:%(lineno)d^[[00m
logging_default_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [^[[00;36m-%(col$
logging_context_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [^[[01;36m%(requ$
use_syslog = False
debug = True
log_file = trove-guestagent.log
log_dir = /var/log/trove/
ignore_users = os_admin
control_exchange = trove
trove_auth_url = http://188.185.76.47/identity/v3
nova_proxy_admin_pass =
nova_proxy_admin_tenant_name = trove
nova_proxy_admin_user = radmin
rpc_backend = rabbit


[oslo_messaging_rabbit]
rabbit_hosts = 188.185.76.47
rabbit_userid = stackrabbit
rabbit_password = verysecret
--------------------------------------------------------------
```


* Back in the horizon dashboard, we can now create a database server instance.
* In Project / Database / Instances -> Launch Instance
* Give your instance a name and select the following options in the details tab.
  

* ds512M is used because the image requires a reasonably large root disk, ds512M has 5GB. 512M of RAM are not needed, at least for testing purposes, so you could create another flavour with 256M if you wanted to be efficient. 
* The networking tab does not need any changes, it should select the private network by default and there are no other options.
* In the initialize databases tab, at the very least an admin username and password for the database should be entered.
* No changes are required in the advanced tab, but this is where you would restore from a backup or replicate another instance.
* Launch the instance. You will see in the databases list that the instance is building. In the ordinary instances tab of openstack, you can see that it is also shown like a regular instance. Here you can view the console or log to check the booting progress.
* If your instance has booted to the login stage in the console and it is still listed as building in the databases section list, then there was a configuration error that stopped rabbit from connecting back to the devstack host to update its status. In this case, you must force the deletion of the instance from the command line.
```
$ trove force-delete trove_test
```
* If you’re having problems with the configuration, log into the instance with the trove user.
* The logs are located at /var/log/upstart/trove-guest.log and also in /var/log/trove/ if it got as far as starting the trove guestagent. 
* When you have your instance starting correctly, the status of it will change to active in the database instances list. 
* You should now have an active database instance.
* You can click the dropdown next to “Create Backup” for some more options, or click the instance name for more database configuration.
* Within the instance details page, when you create a new database in the databases tab, you have to grant access to it in the users tab by clicking the dropdown next to edit user, and then manage access.
* Your instance should have an IP of 10.0.0.x, which you can find in the compute / instances tab. From the devstack ssh, if you ping this IP there is no response. But, you can connect to the databasing using the command
```
$ mysql -h 10.0.0.x -u admin -p
```
* And then entering the password for the user that you created when launching the instance.
* I don’t know why you can connect this way but not ping the instance, networking is some kind of magic.
* The next thing we can do is make is so you can connect to the database from your local machine. This requires some more network configuration. When you use the system-config-firewall-tui tool again it will mess up the networking for your already created instances, so start by deleting the instance you just made. I’m not sure how devstack changes the networking settings when you create a new instance, but I assume that system-config-firewall-tui must overwrite something important.
* First, in Network / Floating IPs, allocate an IP from the public pool to your project. It should look something like 172.24.4.4, which is the one I just got assigned.
* We will forward a port to this IP, and then assign that IP to our database instance.
  

* We could forward to multiple floating IP addresses by changed the source port, and then purposefully changing the port we connect with, but for now let’s just have one.
* Create a new database server the same as last time.
* Now in the Compute / Instances tab, associate a floating ip to the new instance. Select the IP you just created.
* If everything has gone according to plan, you should now be able to connect to the database from your local machine with a command like:
```
$ mysql -h th-devstack.cern.ch -u admin -p
```
* And again entering the password you chose when creating the instance.
* Now you can interact with the database


```
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| test_db            |
+--------------------+
2 rows in set (0.00 sec)
```


* To connect with a different port, use the flags -P 3307 which is equivalent to --port 3307. This would be used if you were forwarding different ports to different floating IPs.

