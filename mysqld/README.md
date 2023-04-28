**Installing MySQL for PASA pipeline** 
================

- [Download](#download)
- [Configure](#configure)
- [PASA Configuration](#pasa-configuration)

This is a tutorial on how to download and set mysqld without root access, this is usefull if you are working in a HPC and you need to install mysql to run a pipeline, in my case I needed mysql working in the HPC to run [funannotate update](https://funannotate.readthedocs.io/en/latest/update.html) faster, to do this you need to change pasa configuration to use mysqld instead of sqllite, as it allows multithreating, this changed the runnig times from about 400 hrs in a node with 1.5tb and 80 cores to around 30 hrs. 

following instructions from [here](https://wasteofserver.com/mysql-server-without-root/), whit some modifications.

# Download

start going to [downlad](https://dev.mysql.com/downloads/mysql/) website and chose the MySQL comunitty server generic linux and chose the 64 bits option, this is specific for the HPC I'm working on it might be different for other systems, then follow the link and you could either download to your home computer and then use scp to move the tar file to the hpc or copy the link and use wget to download directly in the hpc.

```
# to transfer from home computer
scp <file> <ssh transfer node:path>

# to download directly into HPC
# not sure if this link will work for everyone, if not just follown the above link to the webpage and get a new one
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
```

untar the file in the path you want to have it, you also have to change the permits to make that directory open for all users using chmod.

```
tar xvf  paht/to/file/mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
chmod 777 paht/to/file/mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
```

# Configure

then you have to make a .my.cnf file in youre home directory, use any text editor you feel confortable with

```
nano .my.cnf 

```
then you write your configuration file, it shoul look like this:

```
[server]
user=username
basedir=/home/username/mysql/mysql-8.0.33-linux-glibc2.12-x86_64
datadir=/home/username/mysql/data

[mysqld]
pid-file=/home/username/mysql/other/mysqld.pid
socket=/home/username/mysql/mysql-8.0.33-linux-glibc2.12-x86_64/mysqld.sock
port=31666
lc_messages_dir = /home/username/mysql/mysql-8.0.33-linux-glibc2.12-x86_64/share
lc_messages = en_US

[client]
socket=/home/username/mysql/mysql-8.0.33-linux-glibc2.12-x86_64/mysqld.sock
```

you will have to modify the above text to fit the path and username you are using:
- [server]
    - user: your username, use your linux user, not sure if can be changed but it works with it
    - basedir: write the path to the directory you downloaded and untar, don't have to by in your home directory, it can be other path
    - datadir: directory named data that you create, it will be empty for the moment and can be placed at any path

- [mysqld]
    - pid-file: path and the name of the file that will be created, it can be anywhere, and is recommended not to put it in the basedir
    - socket: path and name of the file, that will be created, not sure if can be placed anywhere but it works if you set it in your basedir
    - port: I copy this from this tutorial [here](https://wasteofserver.com/mysql-server-without-root/)
    -lc_message_dir = share folder inside your basedir
    -lc_message = language of the error mesagges, here is set for US english

- [client]
    - socket: same than the above

for the error directoy I went into the share folder of mysql basedir and copy the file errmsg.sys from the english folder into share, not sure if it is necesarry so you can skip this step and if you recive a message about not finding the error files you cat do this.

```
cd mysql-8.0.33-linux-glibc2.12-x86_64/share
cp english/errmsg.sys .
```

then you have to add the mysql bin folder to your path, but it has to be the first one, in my case there was a mysql installed in the root bin folder, so I had to add the path to mysqld that I download before the root path, to do this:

add this line to your .bashrc, remember to change the path to you basedir bin path

```
PATH="/home/username/mysql/mysql-8.0.33-linux-glibc2.12-x86_64/bin:$PATH"
export PATH
```

Then you can star your data directory, the path of datadir must be the same that in your .my.cnf file

```
mysqld --initialize-insecure --datadir=/home/username/mysql/data
```

once you started your data directory you can star your server

- NOTE: make sure to add & at the end of this command 

Otherwhise the server will start but you will be traped there, and the only way to leave is closing the terminal, ctrl+c doesnt work

```
mysqld_safe --defaults-file=/home/username/.my.cnf &
```

once you do that you can use mysql

```
mysql --host=localhost  -u root
```

## Notes

I was testing if you can start mysqld in several nodes and seems that if you started in one node and try to start it in another node then both nodes stop working, so if you are going to work with mysql do it just with one node, didn't tried to have different .my.conf files and set different datadir, socket, and pid, so that might work but I'm not sure.

Be sure to stop mysqld after you finish your work

in my case what worked is close the node I was working with, then open another interactive node and then run mysqld stop.

```
mysqld stop
```
this produce and error that stop the server, then you can start it again in another node in a batch job or in a interactive node. 


# PASA Configuration

Now you can follow mysql configuration for pasa pipeline from [here](https://github.com/PASApipeline/PASApipeline/wiki/setting-up-pasa-mysql)

Start a interactive node and start the server

```
mysqld_safe --defaults-file=/home/username/.my.cnf &
```
Then open mysqld prompt

```
mysql -u root
```
Then you have to follow the instruction to set the pasa users and password

In the mysql promt type the uncomented lines, you can change the user name and/or the identified by for the pasword you chosee, but you will have to modify the conf.txt file of pasa

```
 # create a 'pasa_write' user destined for reading/writing/creating capabilities
 
 create user 'pasa_write'@'localhost' identified by 'pasa_write_pwd';

 # this below will allow user 'pasa_write' to create databases named with a _pasa suffix 
 # (and so not just any database name)
 
 grant all privileges on `%_pasa`.* to 'pasa_write'@'localhost';


 # create a 'pasa_access' user that we'll use for read-only database access.
 # this is useful for generating reports, pasa-web access, etc., where read-only is best.
 
 create user 'pasa_access'@'localhost' identified by 'pasa_access';
```


once done you need to go to the pasa_cnf folder in my case is the following as I used a conda env to install pasa
```
cd /home/ramsesr/.conda/envs/funannotate_env/opt/pasa-2.5.2/pasa_conf
```

once there you will see the following files

```
pasa.alignAssembly.Template.txt
pasa.annotationCompare.Template.txt
pasa.CONFIG.template
README.conf
sample_test.conf
```

if you read the README.conf it tells you to copy the pasa.CONFIG.template and name it conf.txt that is the name that you need

```
cp pasa.CONFIG.template conf.txt 
nano conf.txt
```

Then you have to replace the user and password to what you assinged for the pasa user, I also changed the PASA_ADMIN_EMAIL section

```
# read-write username and password
MYSQL_RW_USER=pasa_write
MYSQL_RW_PASSWORD=pasa_write_pwd

#### END OF MANDATORY SETTINGS ######


#########################################
#### WEB INTERFACE (OPTIONAL) SETTINGS ##
#########################################

# read-only username and password
MYSQL_RO_USER=pasa_access
MYSQL_RO_PASSWORD=pasa_access

## PASA admin settings

#emails sent to admin on job launch, success, and failure
PASA_ADMIN_EMAIL=youremail@google.com
```

Now your set to run funannotate with mysqld option

```
funannotate update -i annotate --cpus 80 --pasa_db mysql
```
