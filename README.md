# Udacity-Linux-Server-Configuration

You can visit http://52.15.201.185 for the deployed Item Catalog

### Step 1: SSH into the server with SSH Keys

- Download and move thr key `LightsailDefaultPrivateKey-*.pem` into the local folder `~/.ssh` and rename it `lightsail_key.rsa`.
- In your terminal, type: `chmod 600 ~/.ssh/lightsail_key.rsa`.
- To connect to the instance via the terminal: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@[PUBLIC IP]`,

### Step 2: Update and upgrade installed packages
```
sudo apt-get update
sudo apt-get upgrade
```

### Step 3: Change the SSH port from 22 to 2200

- Edit the `/etc/ssh/sshd_config` file: `sudo nano /etc/ssh/sshd_config`.
- Change the port number on line 5 from `22` to `2200`.
- `sudo service ssh restart`.

### Step 4: Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123) and match the Network configuration on the AWS console
  - `sudo ufw allow 2200/tcp`
  - `sudo ufw allow 80/tcp`
  - `sudo ufw allow 123/udp`
  - `sudo ufw enable`

### Step 5: Create a new user account named `grader` and Give `grader` access

- logged in as `ubuntu`: `sudo adduser grader`. 

### Step 6: Give `grader` the permission to sudo

- Edits the sudoers file: `sudo visudo`.
- Add a new line to give sudo privileges to `grader` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

### Step 7: Create an SSH key pair for `grader` using the `ssh-keygen` tool

- Run `ssh-keygen`and save in `~/.ssh`
- Log in as `grader`:
  -  `mkdir .ssh`
  -  `sudo nano ~/.ssh/authorized_keys` and paste the content of the public key.
  - Give the permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  - Restart SSH: `sudo service ssh restart`
- To log in use: `ssh -i ~/.ssh/grader_key -p 2200 grader@52.15.201.185`. (The name of the key is `grader_key`)


### Step 8: Configure the local timezone to UTC

- While logged in as `grader`, configure the time zone: `sudo dpkg-reconfigure tzdata`. You should see something like that:

  ```
  Current default time zone: 'America/Montreal'
  Local time is now:      Thu Oct 19 21:55:16 EDT 2017.
  Universal Time is now:  Fri Oct 20 01:55:16 UTC 2017.
  ```

### Step 9: Install and configure Apache to serve a Python mod_wsgi application

- `sudo apt-get install apache2`.
- Enable `mod_wsgi` using: `sudo a2enmod wsgi`.

### Step 10: Install and configure PostgreSQL

- `sudo apt-get install postgresql`.
- Switch to the `postgres` user: `sudo su - postgres`.
- Open PostgreSQL interactive terminal with `psql`.
- Create the `catalog` user with a password and give them the ability to create databases:
  ```
  postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
  postgres=# ALTER ROLE catalog CREATEDB;
  ``
- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.
- Create a new Linux user called `catalog`: `sudo adduser catalog`. Enter password and fill out information.
- Give to `catalog` user the permission to sudo. Run: `sudo visudo`.
- Add line to give sudo privileges to `catalog` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  catalog  ALL=(ALL:ALL) ALL
  ```
- While logged in as `catalog`, create a database: `createdb catalog`.
- Run `psql` and then run `\l` to see that the new database has been created. The output should be like this:
  ```
                                    List of databases
     Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
  -----------+----------+----------+-------------+-------------+-----------------------
   catalog   | catalog  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
   template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
  (4 rows)
  ```
- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.

### Step 11: Install git

- `git`: `sudo apt-get install git`.


### Step 12: Clone and setup the Item Catalog project from the GitHub repository 

- Create `/var/www/catalog/` directory and clone the Github project `sudo git clone https://github.com/mergenthaler/eponymous-law-catalog.git`.
- From the `/var/www` directory, change the ownership of the `catalog` directory to `grader` using: `sudo chown -R grader:grader catalog/`.
- Rename the `app.py` file to `__init__.py`. (Important for running the Flask App with mod_wsgi)

- Replace the sqlite to postgress In `database.py`. (If you encounter operational errors while using SQLITE check that the path is absolute. Its a common problem )
   ```
   engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')
   ``` 


### Step 13: Install the virtual environment and dependencies

- `sudo apt-get install python3-pip`.
- Install the virtual environment: `sudo apt-get install python-virtualenv`
- Create the virtual environment: `sudo virtualenv -p python3 venv3` in `/var/www/catalog/catalog/`
- Change the ownership to `grader` with: `sudo chown -R grader:grader venv3/`.
- Activate the new environment: `. venv3/bin/activate`.
- Install:
  ```
  pip install httplib2
  pip install requests
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```

### Step 14: Set up and enable a virtual host

- Create `/etc/apache2/sites-available/catalog.conf` with the following content

  ```
  <VirtualHost *:80>
    ServerName 52.15.201.185
  ServerAlias ec2-52-15-201-185.us-east-2.compute.amazonaws.com
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static/
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```

### Step 15: Set up the Flask application

- Create `/var/www/catalog/catalog.wsgi` file add the following lines:

  ```
  activate_this = '/var/www/catalog/catalog/venv3/bin/activate_this.py'
  with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))
  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, "/var/www/catalog/")
 
  from catalog import app as application
  application.secret_key = "..."
  ```

### Step 16: Set up the database schema and populate the database needed for the Catalog  reload apache and visit the app

#Extra precautions 
###  Use `Fail2Ban` to ban attackers 

###Special thanks to https://github.com/boisalai for a very helpful guide and links to important references
