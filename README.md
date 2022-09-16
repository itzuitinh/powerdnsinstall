### Install MariaDB All Server
```
sudo apt update
sudo apt install mariadb-server
sudo systemctl start mariadb.service
sudo mysql_secure_installation
```
### Create DB for PowerDNS
```
sudo mysql -u root -p
CREATE DATABASE powerdns;
GRANT ALL ON powerdns.* TO 'powerdns'@'localhost' IDENTIFIED BY 'Str0ngPasswOrd';
FLUSH PRIVILEGES;
USE powerdns;
```
```
CREATE TABLE domains (
  id                    SERIAL PRIMARY KEY,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  TEXT NOT NULL,
  notified_serial       BIGINT DEFAULT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  options               TEXT DEFAULT NULL,
  catalog               TEXT DEFAULT NULL,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE UNIQUE INDEX name_index ON domains(name);
CREATE INDEX catalog_idx ON domains(catalog);


CREATE TABLE records (
  id                    BIGSERIAL PRIMARY KEY,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(65535) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  disabled              BOOL DEFAULT 'f',
  ordername             VARCHAR(255),
  auth                  BOOL DEFAULT 't',
  CONSTRAINT domain_exists
  FOREIGN KEY(domain_id) REFERENCES domains(id)
  ON DELETE CASCADE,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE INDEX rec_name_index ON records(name);
CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX recordorder ON records (domain_id, ordername text_pattern_ops);


CREATE TABLE supermasters (
  ip                    INET NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) NOT NULL,
  PRIMARY KEY(ip, nameserver)
);


CREATE TABLE comments (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  comment               VARCHAR(65535) NOT NULL,
  CONSTRAINT domain_exists
  FOREIGN KEY(domain_id) REFERENCES domains(id)
  ON DELETE CASCADE,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE INDEX comments_domain_id_idx ON comments (domain_id);
CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT REFERENCES domains(id) ON DELETE CASCADE,
  kind                  VARCHAR(32),
  content               TEXT
);

CREATE INDEX domainidmetaindex ON domainmetadata(domain_id);


CREATE TABLE cryptokeys (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT REFERENCES domains(id) ON DELETE CASCADE,
  flags                 INT NOT NULL,
  active                BOOL,
  published             BOOL DEFAULT TRUE,
  content               TEXT
);

CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
  id                    SERIAL PRIMARY KEY,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```
#### Check Tables
```
MariaDB [powerdns]> show tables;
+--------------------+
| Tables_in_powerdns |
+--------------------+
| comments           |
| cryptokeys         |
| domainmetadata     |
| domains            |
| records            |
| supermasters       |
| tsigkeys           |
+--------------------+
7 rows in set (0.000 sec)

MariaDB [powerdns]> EXIT
Bye
```
### Install PowerDNS
```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
ls -lh /etc/resolv.conf 
sudo unlink /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo apt update 
sudo apt install pdns-server pdns-backend-mysql
# Ubuntu 22.04/20.04
echo "deb [arch=amd64] http://repo.powerdns.com/ubuntu focal-auth-master main" | sudo tee /etc/apt/sources.list.d/pdns.list
sudo apt update
sudo apt install pdns-server pdns-backend-mysql
```
### Config DB
```
sudo vim /etc/powerdns/pdns.d/pdns.local.gmysql.conf 
# MySQL Configuration
# Launch gmysql backend
launch+=gmysql
# gmysql parameters
gmysql-host=localhost
gmysql-port=3306
gmysql-dbname=powerdns
gmysql-user=powerdns
gmysql-password=Str0ngPasswOrd
gmysql-dnssec=yes
# gmysql-socket=
```
### Config Master Server
```
vi /etc/powerdns/pdns.conf
allow-axfr-ips=<ip slave>
allow-dnsupdate-from=127.0.0.0/8,::1
allow-notify-from=0.0.0.0/0,::/0
disable-axfr=no
disable-syslog=no
include-dir=/etc/powerdns/pdns.d
launch=
log-dns-queries=yes
log-timestamp=yes
loglevel=5
master=yes
#primary=yes
slave=no
daemon=yes
guardian=yes
query-logging=yes
webserver-port=8081
api=yes
api-key=<keyapi>
default-soa-content=ns1.domain. ns2.domain. 2021101402 21600 3600 1209600 21600


```
Restart services
```
sudo systemctl restart pdns
sudo systemctl enable pdns
```
### In Slave Server
```
vi /etc/powerdns/pdns.conf
launch=
guardian=yes
daemon=yes
log-dns-details=on
slave=yes
slave-cycle-interval=60
logging-facility=0
log-dns-queries=yes
loglevel=5
include-dir=/etc/powerdns/pdns.d
autosecondary=yes
local-address=0.0.0.0
master=no
secondary=yes
superslave=yes
allow-notify-from=<master ip>
allow-dnsupdate-from=<master ip>
default-soa-content=ns1.domain. ns2.domain. 2021101402 21600 3600 1209600 21600

```
#### Update DB in slave server
```
mysql -u powerdns -p
use powerdns;
insert into pdns.supermasters values ('ip master', 'ns2.salve', 'admin');
exit
```
restart services
```
sudo systemctl restart pdns
sudo systemctl enable pdns
```
### Install PowerDNS-Admin On Master node
```
sudo apt install python3-dev
sudo apt install -y libmysqlclient-dev libsasl2-dev libldap2-dev libssl-dev libxml2-dev libxslt1-dev libxmlsec1-dev libffi-dev pkg-config apt-transport-https virtualenv build-essential python3-venv
curl -sL https://deb.nodesource.com/setup_16.x | sudo bash -
sudo apt install -y nodejs
curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/yarn-keyring.gpg
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install -y yarn
sudo su -
git clone https://github.com/ngoduykhanh/PowerDNS-Admin.git /opt/web/powerdns-admin
cd /opt/web/powerdns-admin
python3 -mvenv ./venv
source ./venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
cp /opt/web/powerdns-admin/configs/development.py /opt/web/powerdns-admin/configs/production.py
vim /opt/web/powerdns-admin/configs/production.py
```
Comment out SQLite SQLALCHEMY_DATABASE_URI line and uncomment MySQL one:
```
### DATABASE CONFIG
SQLA_DB_USER = 'powerdns'
SQLA_DB_PASSWORD = 'Str0ngPasswOrd'
SQLA_DB_HOST = '127.0.0.1'
SQLA_DB_NAME = 'powerdns'
SQLALCHEMY_TRACK_MODIFICATIONS = True

### DATABASE - MySQL
#SQLALCHEMY_DATABASE_URI = 'sqlite:///' + os.path.join(basedir, 'pdns.db')
SQLALCHEMY_DATABASE_URI = 'mysql://'+SQLA_DB_USER+':'+SQLA_DB_PASSWORD+'@'+SQLA_DB_HOST+'/'+SQLA_DB_NAME
```
```
export FLASK_APP=powerdnsadmin/__init__.py
export FLASK_CONF=../configs/production.py
flask db upgrade
flask db migrate -m "Init DB"
yarn install --pure-lockfile
```
Configure systemd service and Nginx
```
sudo vim /etc/systemd/system/powerdns-admin.service
[Install]
WantedBy=multi-user.target

[Unit]
Description=PowerDNS-Admin
Requires=powerdns-admin.socket
After=network.target

[Service]
PIDFile=/run/powerdns-admin/pid
User=pdns
Group=pdns
WorkingDirectory=/opt/web/powerdns-admin
ExecStart=/opt/web/powerdns-admin/venv/bin/gunicorn --pid /run/powerdns-admin/pid --bind unix:/run/powerdns-admin/socket 'powerdnsadmin:create_app()'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target

```

```
vi /etc/systemd/system/powerdns-admin.service.d/override.conf
[Service]
Environment="FLASK_CONF=../configs/production.py"
```
```
 sudo vim /etc/systemd/system/powerdns-admin.socket
 [Unit]
Description=PowerDNS-Admin socket

[Socket]
ListenStream=/run/powerdns-admin/socket

[Install]
WantedBy=sockets.target

```
```
sudo vim /etc/tmpfiles.d/powerdns-admin.conf
d /run/powerdns-admin 0755 pdns pdns -

```
```
sudo systemctl daemon-reload
sudo systemctl restart powerdns-admin.socket
sudo systemctl enable powerdns-admin.socket
sudo chown -R pdns:pdns /run/powerdns-admin
sudo chown -R pdns:pdns /opt/web/powerdns-admin
```
```
systemctl status powerdns-admin.socket
```
Install and Configure Nginx for Powerdns-Admin
```
sudo apt install nginx

```
```
sudo vim /etc/nginx/conf.d/powerdns-admin.conf
```
```
server {
  listen *:80;
  server_name               domain.com;

  index                     index.html index.htm index.php;
  root                      /opt/web/powerdns-admin;
  access_log                /var/log/nginx/powerdns_admin_access.log combined;
  error_log                 /var/log/nginx/powerdns_admin_error.log;

  client_max_body_size              10m;
  client_body_buffer_size           128k;
  proxy_redirect                    off;
  proxy_connect_timeout             90;
  proxy_send_timeout                90;
  proxy_read_timeout                90;
  proxy_buffers                     32 4k;
  proxy_buffer_size                 8k;
  proxy_set_header                  Host $host;
  proxy_set_header                  X-Real-IP $remote_addr;
  proxy_set_header                  X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_headers_hash_bucket_size    64;

  location ~ ^/static/  {
    include  /etc/nginx/mime.types;
    root /opt/web/powerdns-admin/powerdnsadmin;

    location ~*  \.(jpg|jpeg|png|gif)$ {
      expires 365d;
    }

    location ~* ^.+.(css|js)$ {
      expires 7d;
    }
  }

  location / {
    proxy_pass            http://unix:/run/powerdns-admin/socket;
    proxy_read_timeout    120;
    proxy_connect_timeout 120;
    proxy_redirect        off;
  }
}

```
```
sudo nginx -t
sudo systemctl restart nginx
```
