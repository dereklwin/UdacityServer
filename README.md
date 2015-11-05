# UdacityServer
Take a baseline installation of Linux and prepare it to host web applications

i. The IP address and SSH port so your server can be accessed by the reviewer.
Public IP Address:
52.24.247.4
port:
2200

ii. The complete URL to your hosted web application.
ec2-52-24-247-4.us-west-2.compute.amazonaws.com

iii. A summary of software you installed and configuration changes made.
sudo apt-get install apache2
sudo apt-get install python-setuptools libapache2-mod-wsgi
sudo apt-get install postgresql
sudo apt-get install git
sudo apt-get install python-sqlalchemy
sudo apt-get install python-psycopg2
sudo apt-get install python-flask
sudo apt-get install python-pip
sudo pip install oauth2client
sudo pip install --upgrade google-api-python-client
sudo apt-get install fail2ban
sudo apt-get install cron-apt

1. Launch your Virtual Machine with your Udacity account
2. Follow the instructions provided to SSH into your server
    ssh -i ~/.ssh/udacity_key.rsa -p 22 root@52.24.247.4

3. Create a new user named grader
    "sudo adduser grader"

4. Give the grader the permission to sudo
    "sudo touch /etc/sudoers.d/grader"
    "sudo nano /etc/sudoers.d/grader"

    inside grader, add the line
    "grader ALL=(ALL) NOPASSWD:ALL"

    copy ssh key from root
    "cp /root/.ssh/authorized_keys /home/grader/.ssh/authorized_keys"

    RSAAuthentication yes
PubkeyAuthentication yes


5. Update all currently installed packages
    "sudo apt-get update" : updates the source list
    "sudo apt-get upgrade": will upgrade all packages that have updates 

6. Change the SSH port from 22 to 2200
    "sudo nano /etc/ssh/sshd_config"
    change "Port 22" to "Port 2200"

    while there make sure passwords are disabled and public key authentication is used
    "RSAAuthentication yes"
    "PubkeyAuthentication yes"

    Disable ssh on root
    "PermitRootLogin no"

7. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
    
    “sudo ufw status” check status of firewall
    “sudo ufw default deny incoming” deny all incoming traffic
    “sudo ufw default allow outgoing”
    “sudo ufw allow 2200/tcp” to allow for ssh tcp connections
    “sudo ufw allow www” allows http requests
    “sudo ufw allow ntp” allows http requests
    “sudo ufw enable” enables the firewall.

8.Configure the local timezone to UTC
    "date" - to see current timezone
    find desired timezone in zoneinfo
    "cd /usr/share/zoneinfo/"

    create symbolic link to UTC
    "sudo unlink /etc/localtime"
    "sudo ln -s /usr/share/zoneinfo/UTC /etc/localtime"

9. Install and configure Apache to serve a Python mod_wsgi application
    sudo apt-get install apache2
    sudo apt-get install python-setuptools libapache2-mod-wsgi

    "sudo nano /etc/apache2/sites-enabled/000-default.conf"

    set server name and admin as grader
    "ServerName http://52.24.247.4/"
    "ServerAdmin grader@http://52.24.247.4/"

    add the below line before the closing "</VirtualHost>"
    "WSGIScriptAlias / /var/www/UdacityCatalog/myapp.wsgi
        <Directory /var/www/UdacityCatalog/>
            Order deny,allow
            Allow from all
        </Directory>"

    The WSGIScriptAlias directive tells wsgi that "/var/www/UdacityCatalog/myapp.wsgi" contains WSGI scripts and that it is an alias to the "/" path.
    The order directive "Order deny,allow" evaluates all deny before it matches an allow directive. 
    Currently we allow all hosts to access to the apache server with "Allow from all"

10. Install and configure PostgreSQL:
    sudo apt-get install postgresql

    Do not allow remote connections
    "sudo nano /etc/postgresql/9.3/main/pg_hba.conf"
    By default PostgreSQL only allows remote connections. This can be verified by looking in pg_hba.conf and see all addresses are for localhost

    local   all             postgres                                peer
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    # "local" is for Unix domain socket connections only
    local   all             all                                     peer
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            md5
    # IPv6 local connections:
    host    all             all             ::1/128                 md5

    Create a new user named catalog that has limited permissions to your catalog application database

    run commands as postgres and start psql
    "sudo -u postgres psql"

    create user
    "CREATE USER catalog ;"
    set password
    "ALTER USER catalog WITH PASSWORD 'drowssap';"
    create database
    "CREATE DATABASE cities;"

    allow catalog to connect to cities database
    "GRANT CONNECT ON DATABASE cities TO catalog;"

    connect to cities
    "\c cities"
    allow only limited access to each schema to catalog
    "GRANT SELECT, UPDATE, INSERT, DELETE, REFERENCES ON cities TO catalog;"
    "GRANT SELECT, UPDATE, INSERT, DELETE, REFERENCES ON destinations TO catalog;"
    "GRANT SELECT, UPDATE, INSERT, DELETE, REFERENCES ON userdb TO catalog;"
    allow useage on sequence
    "GRANT USAGE, SELECT ON SEQUENCE cities_id_seq TO catalog;"

    Install git, clone and setup your Catalog App project (from your GitHub repository from earlier in the Nanodegree program) so that it functions correctly when visiting your server’s IP address in a browser. Remember to set this up appropriately so that your .git directory is not publicly accessible via a browser!

    clone catalog to /var/www
    "cd /var/www"
    "git clone https://github.com/dereklwin/UdacityCatalog.git"

    protect the .git folder by using .htaccess 
    "sudo touch /var/www/UdacityCatalog/.htaccess"
    "sudo nano /var/www/UdacityCatalog/.htaccess"

    inside .htaccess redirect all requests to .git folder to 404
    "RedirectMatch 404 /\.git"

    changes in catalog app are in 
    https://github.com/dereklwin/UdacityCatalog

    restart apache server
    "sudo service apache2 restart"

    Troubleshooting:
    if there are issues, first look at apache error log
    "sudo cat /var/log/apache2/error.log"

Extra:

-The firewall has been configured to monitor for repeat unsuccessful login attempts and appropriately ban attackers; 

    "sudo apt-get install fail2ban"

    To configure fail2ban, make a 'local' copy the jail.conf file in /etc/fail2ban
    "cd /etc/fail2ban"
    "sudo cp jail.conf jail.local"

    Now edit the file:
    "sudo nano jail.local"
    # if there are 3 failed attempts in 10 min, ban for one hour
    bantime  = 3000
    findtime = 600
    maxretry = 3

    # send email to grader when a user is banned
    destemail = grader@localhost
    # set default action to ban & send an e-mail with whois report to the destemail.
    action = %(action_mw)s

    "monitor ssh and ssh-ddos and apache"
    set [ssh], [ssh-ddos], and [apache]
    "enabled  = true"

    restart fail to ban and check status
    "sudo service fail2ban restart"
    "sudo fail2ban-client status"

-cron scripts have been included to automatcially manage package updates
    "sudo apt-get install cron-ap"

    create file in weekly cron folder
    "sudo touch /etc/cron.weekly/autoupdt"
    make chmode executable my everyone
    "sudo chmod 755 /etc/cron.weekly/autoupdt"
    "sudo nano /etc/cron.weekly/autoupdt"
    
    add the following code to auto update and upgrade
    "#!/bin/bash
        apt-get update
        apt-get upgrade -y
        apt-get autoclean"

-vm includes monitoring applications that provide automated feedback on application availability status and/or system security alerts
    "sudo apt-get install munin"
    "sudo nano /etc/munin/munin.conf"
    
    htmldir /var/www/munin/
    uncomment Dbdir, htmldir, logdir, rundir, tmpldir
    Dbdir stores all of the rrdfiles containing the actual monitoring information, htmldir stores the images and site files, logdir maintains the logs, and rundir holds the state  files.   
    
    replace [localhost.localdomain] with [grader.52.24.247.4]
    
    edit munin apache config
    "sudo nano /etc/munin/apache.conf"
    
    edit
    # Enable this for template generation
    Alias /munin /var/www/munin/
    
    <Directory /var/www/munin/>
        Order allow,deny
        Allow from all


    visit "http://52.24.247.4/munin/" to view monitor

iv. A list of any third-party resources you made use of to complete this project.
https://www.digitalocean.com/
https://help.ubuntu.com/community/
http://stackoverflow.com/
http://www.postgresql.org/docs/9.3/interactive/index.html


Open your ~/.ssh/udacity_key.rsa file in a text editor and copy the contents of that file.

-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAyW0BXQTSSXCegi/xqaXIyCnOK07AB3bmHfBVaTJV3OU+UhJJ
gQ44fBp2pbE+H4vWJJY7NA5wfYHk7fGDHvg2mARHZDMAaR4Kezz1aFBfZuNfprZq
5ulctMhCT4DTS6DW5z7SHPXrNwICqV16XFyDG6hU9y97Fk9SMi0L8Pxg3mGn/2TA
Fz30GbIeQOoaj604CN3iWfNudjAbW5FMV+rDKphBkYwhu5TGfRBiKUDBz7UeWrD8
zYX+GN+oSytGGvIMHl3dv4peER8dicdYcHuaHJuceUSeHKAC7IkTaTcxRljm8WdQ
Hh9my/fqLm2tIODT3OArcgNFLr8kvniwLAf+jQIDAQABAoIBAG22qyRwiN4pspz0
4mvmekvUwZDDT0OBluw9yTgIi85LK7vmbBUYmtm2TGQJ++2Q7G53Sf4b01f5lamp
gCMxTgNVaVGBmjqne0wPMxjDloNjW+lhuS7Xc4ChB8VoRS8Ph57jj+zoYltPBAYe
fZSra1p4QPd27FOFlx7vfG6h+V2G2mloElRe5qpHFu1U98SlmgBtQvT2EVxQObNN
Vn9ea/VH021KLHdzZ3DN/B7+g99rubhItBSZQlZHqwx6IQ1DrV0XDFL6M0V70bmm
3XsqOMK8HHIkjXOgGvDahe7hDoBd14mQ1tjtluXQJefG3XImqBstn+noetmUk/hV
sCdSakECgYEA5OptT506oQk/yKJlIeB0chcAX3AfNzClI6Iy4ug1GY/3qzqKOVYg
VFXC27l90Mgv0gDS3EZwVUX3tr6/DEHaJXLC7+OcRR2SZQN3vDcqaGrk9ZZ/313M
v97Mr663MSmvFQWdjKWxlnndCTquPlCGe1FPVv8MFo1yPkWmzCvtaV0CgYEA4UHx
S5kFUWCuy6/yZJxWLcQWgBQ9qbQ+oJRrzWxRboNi186x60DUwnGeI+Low8rNf+ZG
XnN/M9TvfiTokXzc7saYfOTXJ3YmER4cyPsXqkS47l6E+rKYCS4gSFLM9Gtf/YrM
aChikfRozlI59LKtN7GgbVr5YrHMgz2t4RE0JvECgYA/rCktjOlC46S3NNx2eM1K
8rTq1vAH1OMKL1KCJN6oNpBIM2dBHYCulJA3t7eUPCp4+jusg3c5cNW/If1X9nUs
F2i7ew77dodCy50hYCLOmnUHDo6Q3bFW6Sz77NgNt694ZHB3L5te5JSjvYu7z4Ao
iuxLoXOGTl+pjIwhnFJUDQKBgQDWBKw4snOeBOkut8Xql6s9om/qUtDfe1SBh2MB
cyfPg1+XQVhD933uHLsux3l2BSrImUZEmSHDYk4FoRWinWrgJqpdB6PwZ031t5GL
1x199ftq5z0bYDIZjsy3SoxWseoq4AQj9jLpD7nARdmwx07Sep69J9GIVvvDugeJ
rqnJUQKBgAkJCYlV4hzb72EaLZunCC7o/YCwjMMoy+2568+cU3Q3KYt0Ad781wIS
BRB6V6Ba1vWGg+4yFs4NofagzIXvLD5zfrsyUguBTn9sehJlijBXj3RlXfrVY3yW
tMyn6iw5SBCkrop0dfCgyYrGsrNVOfDiNLaV9ZAmPNVklGxOdB2r
-----END RSA PRIVATE KEY-----