# Installation guide for MySQL to Turris Omnia
> A step-by-step tutorial how to install MySQL database to a LXC container running on Turris Omnia.

## Warning
An installation of any database could lead to high level of I/O operations. If you install your LXC container database host to the internal Turris Omnia storage, it could and **probably it will destroy** your internal Turris Omnia storage by wearing up. Turris Omnia can't operate without this internal storage.

An easy solution is to buy SSD with mSATA socket, plug it in and switch the internal storage to this newly mounted drive. Follow [this](https://www.youtube.com/watch?v=71_M2N3ga7s) for upgrade. Additional information can be found [here](https://forum.turris.cz/t/installing-of-msata-disk/6208/6).

# Overview

The installation consists of these steps:

1. Create a LXC container
2. Container configuration
3. Deployment of MySQL to the LXC container
4. MySQL first time configuration
5. Remote access configuration for DBeaver


This tutorial is made for **Ubuntu 20.04 LTS** or **Debian** distributions (tested on both), **dbserver** as a name LXC container and **testdb** as a name of a database that will exist in the container, **appuser** as a name of database user configured to connect to the database. You can change these strings anything you want without any impact on the result as long as you stay consistent with changes.

## 1. Create Debian/Ubuntu LXC container

Connect to your Turris Omnia router by SSH and create a LXC container for your database. **[Official Manual](https://www.turris.cz/doc/en/howto/lxc)**

**Note**: If you don't see any templates in LuCI or you receive [this](https://forum.turris.cz/t/lxc-container-no-templates/5296/17) error, just change your DNS settings in Foris from whatever you have there to Cloudface or anything else. The problem is caused by DNS which is not resolving template URL correctly.
```
lxc-create -t download -n dbserver
```

- Distribution: **Debian**
- Release: **Stretch**
- Architecture: **armv7l** (that is seven and the lower "L" letter, not 1 (digit)

Alternatively, you can use LuCI to create the LXC container. A distribution doesn't not really matter. This tutorial was tested on Debian and Ubuntu but it would work more likely on other distributions.

## 2. Container configuration

Start the container from LuCI or by executing this command:

```
lxc-start -n dbserver
```

Connect to the container:

```
lxc-attach -n dbserver
```

Install prerequisites:

```
apt-get update
apt-get install nano wget -y
```

Change the container hostname from `LXCNAME` to `dbserver`:

```
nano /etc/hostname
```
and save the change. **Hint:** ^X means CTRL+X.


Set the new hostname to localhost:

```
nano /etc/hosts
```

Change the line

```
127.0.1.1   LXCNAME
```

to this:

```
127.0.1.1   dbserver
```
and save this change.

Exit the LXC container SSH session back to your Turris Omnia session:

```
exit
```
and from here you can easily restart the container:

```
lxc-stop -n dbserver -r
```

Go to **Foris**, to the the **DNS** setting tab and turn on the "Domain of DHCP clients in DNS" if you don't have it enabled already. Now, you can access the LXC container by its domain name - **dbserver**. 

After restart, you can connect to the container by executing:

```
lxc-attach -n dbserver
```

and if your setup was successful, you should see: **root@dbserver:~#** in your terminal.

## 3. Deployment of MySQL to LXC container
Once you are again connected to your container, install the MySQL package:

```
apt-get install mysql-server
```

This will install a lot of other packages, about 300 MB of binaries.

## 4. First time configuration

Firstly, we need to configure `root` (*master*) password for the database. Run the MySQL secure installation utility:

```
mysql_secure_installation utility
```
and follow the instruction.

Now, let's edit the binding of the database.
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
You can edit the line:
```
bind-address		= 127.0.0.1
```
which is default to anything you want. `127.0.0.1` means the binding only to localhost so application must be deployed to the same container. If you want make it available to all interfaces, use:

```
bind-address		= 0.0.0.0
```

To ensure that the database server launches after a reboot, run the following command:

```
systemctl enable mysql
```

Now, we can start the service:

```
systemctl start mysql
```

To create additional user, our database and setup the privileges, we have to run MySQL shell:

```
/usr/bin/mysql -u root -p
```

Now, use your Password vault (if you don't have one, [get one!](https://www.google.com/search?client=firefox-b-d&q=password+vault)) and generate a strong password for a new database user and execute the following command in MySQL shell:

```
CREATE USER 'appuser'@'192.168.%' IDENTIFIED BY 'SomeStrongPassword';
```

A new user - `appuser` will be identified by the password and will be able to connect from any IP starting with `192.168` which is Turris Omnia default for the local network. If you want the public access to your database container, you have to redirect the traffic towards this container. If you need `appuser` to be eligible to connect from anywhere, change the shell command into this:

```
CREATE USER 'appuser'@'%' IDENTIFIED BY 'SomeStrongPassword';
```

Now, we can create a database by this MySQL shell command:

```
CREATE DATABASE testdb;
```

and finally we can assign the privileges:
```
GRANT ALL PRIVILEGES ON testdb.* to 'appuser'@'192.168.%';
FLUSH PRIVILEGES;
```

We can test users and database by executing:
```
SELECT User, Host, authentication_string FROM mysql.user;
SHOW DATABASES;
SHOW GRANTS FOR 'appuser'@'192.168.%';
```

## 5. Remote access configuration for DBeaver
DBeaver is a popular DB management software. Use and creation of a DB connection is pretty must straight forward but there are some issues with connection to the latest MySQL package (8.x) with the connection DBeaver driver.
To establish a connection to your database, you have to edit connection properties at **Connection properties** tab when you are creating a new connection. You have to modify following properties and theirs values:
* **allowPublicKeyRetrieval** | True
* **useSSL** | False

And now you are good to go. Your have fully self-hosted MySQL database in your Turris Omnia!


## References:
[Install MySQL Server on the Ubuntu operating system](https://support.rackspace.com/how-to/install-mysql-server-on-the-ubuntu-operating-system/)
<br>
[Configure MySQL server on the Ubuntu operating system](https://support.rackspace.com/how-to/configure-mysql-server-on-the-ubuntu-operating-system/)
<br>
[MySQL Network Security](https://serversforhackers.com/c/mysql-network-security)
<br>
[MySQL Public Key Retrieval is not allowed (DBeaver configuration fix for MySQL)](https://community.atlassian.com/t5/Confluence-questions/MySQL-Public-Key-Retrieval-is-not-allowed/qaq-p/778956)