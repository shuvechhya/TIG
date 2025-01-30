This document describes the process of monitoring the metrics for different applications mysql,nginx and linux system 

The following vps was used for the initial testing phase.
a. CPU:
b. Memory:
c. Storage:
d. IP: 10.10.40.108

# TIG
**T- Telegraf**

**What is Telegraf?**

Telegraf is an open-source plugin-driven agent for collecting data (Metrics and Events) from your stacks, IoT sensors, and system.
![[Pasted image 20240902170933.png]]

**I-Influx**

**What is influx?**

InfluxDB is an open-source time series database (TSDB) developed by the company InfluxData. It is used for storage and retrieval of time series data in fields such as operations monitoring, application metrics, Internet of Things sensor data, and real-time analytics. It also has support for processing data from Graphite.

**G-Grafana**

**What is grafana?**

Grafana open source software enables you to query, visualize, alert on, and explore your metrics, logs, and traces wherever they are stored. Grafana OSS provides you with tools to turn your time-series database (TSDB) data into insightful graphs and visualizations. The Grafana OSS plugin framework also enables you to connect other data sources like NoSQL/SQL databases, ticketing tools like Jira or ServiceNow, and CI/CD tooling like GitLab.

### 1.Mysql

A popular open-source relational database management system (RDBMS) that uses Structured Query Language (SQL) for accessing and managing data. 

### 2. Prerequisites

**docker:**  Docker streamlines the deployment and management of MySQL by packaging it in a portable, consistent container, simplifying integration with other TIG stack components.

**OS requirements:**
To install Docker Engine, you need the 64-bit version of one of these Ubuntu versions:

- Ubuntu Noble 24.04 (LTS)
- Ubuntu Jammy 22.04 (LTS)
- Ubuntu Focal 20.04 (LTS)
Docker Engine for Ubuntu is compatible with x86_64 (or amd64), armhf, arm64, s390x, and ppc64le (ppc64el) architectures.

**mysql** : It is installed and configured on your Linux system so that we can monitor the metrics.

### 3. Installing Prerequisites

- Uninstall old versions:
```
sudo apt-get remove docker docker-engine docker.io containerd runc

```

- Add Docker's official GPG key:
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

```

- Add Docker repository:
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

- Install Docker Engine:
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- verify installation:
```
docker --version 
docker ps 
```

- Enable Docker to Start on Boot:
```
sudo systemctl enable docker
```

### 4. Installing mysql

###### a. Installing mysql

- To install it, update the package:
```
sudo apt update
```

- Install the `mysql-server` package:
```
sudo apt install mysql-server
```

- Start the service:
```
sudo systemctl start mysql.service
```

###### b. Configuring mysql

###### Steps to configure MySQL Securely:

- Open the MySQL Prompt :
```
sudo mysql
```

- Run the following command to change the root user's authentication method to `mysql_native_password`, which allows password-based authentication:
```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
```

- After changing the authentication method, exit the MySQL prompt:
```
exit
```

- Now, you can safely run the `mysql_secure_installation` script:
```
sudo mysql_secure_installation
```

- This script will guide you through several prompts to secure your MySQL installation. Here’s what to expect:

- Validate Password Plugin: You’ll be asked if you want to enable the Validate Password Plugin, which enforces strong password policies for MySQL users. If you choose to enable it, you’ll have three options:

- 0 = LOW: Passwords must be at least 8 characters long.
- 1 = MEDIUM: Passwords must be at least 8 characters long and include numbers, mixed case, and special characters.
- 2 = STRONG: In addition to the MEDIUM requirements, passwords must meet dictionary file criteria.
After enabling this plugin, any new MySQL user passwords will need to satisfy the chosen policy.

Set Root Password: The script will prompt you to set a password for the MySQL root user. Enter the password you chose earlier when modifying the authentication method.

- Revert the Authentication Method:
```
mysql -u root -p
```

- Enter the password you set earlier to log in.
```
ALTER USER 'root'@'localhost' IDENTIFIED WITH auth_socket;`
```
- This allows you to connect to MySQL as the root user without a password by using the `sudo mysql` command.

###### c. Docker compose
```
version: '3.6'

services:

  telegraf:

    image: telegraf

    container_name: telegraf

    restart: always

    volumes:

    - /opt/srvstatus:/opt/srvstatus:ro

    - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro

    - /var/run/docker.sock:/var/run/docker.sock

    depends_on:

      - influxdb

    links:

      - influxdb

    ports:

    - '8125:8125'

  influxdb:

    image: influxdb:1.8-alpine

    container_name: influxdb

    restart: always

    environment:

      - INFLUXDB_DB=influx

      - INFLUXDB_ADMIN_USER=admin

      - INFLUXDB_ADMIN_PASSWORD=admin

    ports:

      - '8086:8086'

    volumes:

      - influxdb_data:/var/lib/influxdb

  grafana:

    image: grafana/grafana

    container_name: grafana-server

    restart: always

    depends_on:

      - influxdb

environment:

      - GF_SECURITY_ADMIN_USER=admin

      - GF_SECURITY_ADMIN_PASSWORD=admin

      - GF_INSTALL_PLUGINS=

    links:

      - influxdb

    ports:

      - '3000:3000'

    volumes:

      - grafana_data:/var/lib/grafana

  logstash:

    image: docker.elastic.co/logstash/logstash:7.9.3

    container_name: logstash

    restart: always

    volumes:

      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro

    ports:

      - "5044:5044"

      - "9600:9600"

    depends_on:

      - influxdb

    links:

      - influxdb

volumes:

  grafana_data: {}

  influxdb_data: {}
```

###### d. Creating user and database in influx db
 - To access the influx db container
 ```
 docker exec -it <container_id> /bin/sh
```

- To use influx 
```
Use influx 
```

- To create database
```
create database <database_name>
```

- To create a user in influx
```
create user <username> with password '<your_password>'
```

- Grant all access to the user
```
GRANT ALL ON <database_name> TO <username>;
```

###### e. Telegraf configuration for sql monitoring 
```
[global_tags]

[agent]

  interval = "60s"

  round_interval = true

  metric_batch_size = 1000

  metric_buffer_limit = 10000

  collection_jitter = "0s"

  flush_interval = "10s"

  flush_jitter = "0s"

  precision = ""

  hostname = "127.0.0.1"

  omit_hostname = false

[[outputs.influxdb]]

  urls = ["http://10.10.40.108:8086"]

  database = "sql_metrics"

  timeout = "5s"

  username = "newuser"

  password = "newpassword123"

[[inputs.cpu]]

  percpu = true

  totalcpu = true

  collect_cpu_time = false

  report_active = false

[[inputs.disk]]

  ignore_fs = ["autofs", "binfmt_misc", "cgroup", "configfs", "debugfs", "devfs"]

[[inputs.kernel]]

[[inputs.mem]]

[[inputs.processes]]

[[inputs.swap]]

[[inputs.system]]

[[inputs.mysql]]

  servers = ["root:$huvechhy@Bajracharya206@tcp(10.10.40.108:3306)/"]

  metric_version = 2
```

### 5. Ensuring SQL Metrics Delivery

```
use <database_name>
```

```
show measurements
```

![[Pasted image 20240902170822.png]]

### 6. Grafana 

- Create a data source of influx 

In HTTP

```
URL: http:// server-ip:influx_port
```

![[Pasted image 20240902171231.png]]

- After configuring you can check it with save and test
![[Pasted image 20240902171318.png]]

### 7. Dashboard 

We can import Dashboard from grafana dashboard [Grafana Dashboards](https://grafana.com/grafana/dashboards/) for monitoring the metrics.

![[Pasted image 20240903094746.png]]
![[Pasted image 20240902204531.png]]

# Linux system overview
Linux is an operating system. In fact, one of the most popular platforms on the planet, Android, is powered by the Linux operating system.

### 1. Prerequisites

**Telegraf:** To install telegraf in ubuntu and send linux logs to influx db running on docker on a different server 

### 2. Install telegraf on ubuntu 

- Add the influxData Repository 
- Download the GPG key 
```
curl --silent --location -O https://repos.influxdata.com/influxdata-archive.key
```

- Verify the GPG key 
```
echo "943666881a1b8d9b849b74caebf02d3465d6beb716510d86a39f6c8e8dac7515 influxdata-archive.key" | sha256sum -c -
```
If the output is "influxdata-archive.key: OK", proceed to the next step

- Convert the key to a GPG format and save it
```
cat influxdata-archive.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive.gpg > /dev/null
```

- Add the InfluxData repository to your sources list
```
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
```

- Update the package list 
```
sudo apt-get update
```

- Install Telegraf
```
sudo apt-get install telegraf
```

- Start and enable Telegraf 
```
sudo systemctl start telegraf
sudo systemctl enable telegraf
```

- Verify the installation
```
sudo systemctl status telegraf
```

- Generate telegraf.conf
```
telegraf config > telegraf.conf
```

### 3. Telegraf configuration for linux

```
nano /etc/telegraf/telegraf.conf
```

```
[global_tags]
[agent]
  interval = "60s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  hostname = "127.0.0.1"
  omit_hostname = false

[[outputs.influxdb]]
  urls = ["http://10.10.40.108:8086"]
  database = "linux"
  timeout = "5s"
  username = "linuxuser"
  password = "linux123"

[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false

[[inputs.disk]]
  ignore_fs = ["autofs", "binfmt_misc", "cgroup", "configfs", "debugfs", "devfs"]

[[inputs.kernel]]

[[inputs.mem]]

[[inputs.processes]]

[[inputs.swap]]

[[inputs.system]]

```

### 4. Dashboard
We can import Dashboard from grafana dashboard [Grafana Dashboards](https://grafana.com/grafana/dashboards/) for monitoring the metrics.

![[Screenshot (576).png]]

# Nginx 
Nginx is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server.

**Basic HTTP server features**
- Serving static and index files, autoindexing; open file descriptor cache;
- Accelerated reverse proxying with caching; load balancing and fault tolerance;
- Accelerated support with caching of FastCGI, uwsgi, SCGI, and memcached servers; load balancing and fault tolerance;
- Modular architecture. Filters include gzipping, byte ranges, chunked responses, XSLT, SSI, and image transformation filter. Multiple SSI inclusions within a single page can be processed in parallel if they are handled by proxied or FastCGI/uwsgi/SCGI servers;
- SSL and TLS SNI support;
- Support for HTTP/2 with weighted and dependency-based prioritization;
- Support for HTTP/3.

### 1. Prerequisites

**Ubuntu System:** Ensure you have a running Ubuntu system.

### 2. Installing Nginx 

Update system and packages 
```
sudo apt update 
```

Install nginx 
```
sudo apt install nginx 
```

Adjusting the firewall
```
sudo ufw app list
```

```
sudo ufw allow 'Nginx HTTP'
```

```
sudo ufw status
```

Checking your Web Server 
```
systemctl status nginx
```

### 3. Configuring geoip in nginx 

Verify if Nginx compiled with status module support:
```
nginx -V 2>&1 | grep -o with-http_stub_status_module
```

create a new configuration file, to enable locally accessible statistic page with metrics, from status module:
```
vi /etc/nginx/conf.d/status.conf
```

 put configuration parameters in there:
```
server {

    listen 0.0.0.0:9090;
    location /nginx_status {
        stub_status on;

        access_log off;
        allow all;
        deny all;
    }
}
```

Now reload Nginx:
```
nginx -s reload
```

If everything OK, you can check local metrics by making curl on 127.0.0.1:9090/nginx_status address:
```
curl 127.0.0.1:9090/nginx_status

Active connections: 1
server accepts handled requests
 13 13 307
Reading: 0 Writing: 1 Waiting: 0
```

Check that your Nginx also compiled with GEOIP module:

```
nginx -V 2>&1 | grep -o with-http_geoip_module
```

If everything is OK, change the logging section in the nginx.conf and add new custom log format:
```
#Logging Settings   
#Enabling request time and GEO codes   
log_format custom '$remote_addr - $remote_user [$time_local]'  
                  '"$request" $status $body_bytes_sent'  
                  '"$http_referer" "$http_user_agent"'  
                  '"$request_time" "$upstream_connect_time"'  
                  '"$geoip_city" "$geoip_city_country_code"';
access_log /var/log/nginx/access.log custom;  
error_log /var/log/nginx/error.log;
```

Download latest GeoIP.dat and GeoLiteCity.dat files and put them in to /etc/nginx/geoip. The download link is as follows:
[GeoIP Legacy Database](https://www.miyuru.lk/geoiplegacy)
```
mkdir /etc/nginx/geoip  
cd /etc/nginx/geoip
```

Using scp 
```
scp /mnt/c/Users/Dell/Downloads/GeoCity.dat.gz ubuntu@10.10.40.108:/etc/nginx/geoip
gunzip GeoCity.dat.gz 

scp /mnt/c/Users/Dell/Downloads/GeoIP.dat.gz ubuntu@10.10.40.108:/etc/nginx/geoip
gunzip GeoIP.dat.gz
```

Grant permissions:
```
chown ubuntu:ubuntu /etc/nginx/geoip
 sudo chmod 644 /etc/nginx/geoip/GeoCity.dat
 sudo chown root:root /etc/nginx/geoip/GeoIP.dat
```

After that, you need to add a few strings in to the http {} section of the nginx.conf, to enabling GEO data:

```
# Add GEO IP support  
geoip_country /etc/nginx/geoip/GeoIP.dat; # the country IP data  
geoip_city /etc/nginx/geoip/GeoLiteCity.dat; # the city IP data
```

Nginx.conf
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}


http {
        geoip_country /etc/nginx/geoip/GeoIP.dat;
        geoip_city /etc/nginx/geoip/GeoCity.dat;
        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ##
        # Logging Settings
        ##
        # Enabling request time and GEO codes
        log_format custom '$remote_addr - $remote_user [$time_local]'
                  '"$request" $status $body_bytes_sent'
                  '"$http_referer" "$http_user_agent"'
                  '"$request_time" "$upstream_connect_time"'
                  '"$geoip_city" "$geoip_city_country_code"';

        access_log /var/log/nginx/access.log custom;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript             te>
        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;

        server {
            listen 80; server_name 10.10.40.108;

            root /usr/share/nginx/html;
            index index.html;

        location / {
            try_files $uri $uri/ =404;
            }
        }
}
```

To check syntax of conf 
```
nginx -t
```
### 4. Telegraf.conf for nginx 
```
[global_tags]

[agent]
  interval =           "60s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  hostname = "127.0.0.1"
  omit_hostname = false

[[outputs.influxdb]]
  urls = ["http://influxdb:8086"]
  database = "influx"
  timeout = "5s"
  username = "telegraf"
  password = "metricsmetricsmetricsmetrics"


[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false


[[inputs.disk]]
  ignore_fs = ["autofs", "binfmt_misc", "cgroup", "configfs", "debugfs", "devfs", ">
[[inputs.nginx]]
   urls = ["http://10.10.40.108:9090/nginx_status"]
   response_timeout = "5s"
[[inputs.logparser]]
   files = ["/var/log/nginx/access.log"]
   from_beginning = true
   name_override = "nginx_access_log"

   [inputs.logparser.grok]
     patterns = ["%{CUSTOM_LOG_FORMAT}"]
     custom_patterns = '''
        CUSTOM_LOG_FORMAT %{CLIENT:client_ip} %{NOTSPACE:ident} %{NOTSPACE:auth} \[>      '''

#[[inputs.exec]]
#   commands = [
#     "/opt/srvstatus/venv/bin/python /opt/srvstatus/service.py"
#   ]
#   timeout = "5s"
#   name_override = "services_stats"
#   data_format = "json"
#   tag_keys = [
#    "service"
#   ]


[[inputs.diskio]]

[[inputs.kernel]]

[[inputs.mem]]

[[inputs.processes]]

[[inputs.swap]]

[[inputs.system]]

[[inputs.tail]]
  name_override = "nginx_access"
  files = ["/var/log/nginx/access.log"]
  from_beginning = true
  name_suffix = "_nginx"
  tagexclude = ["path"]
  data_format = "influx"

[[outputs.influxdb]]
  urls = ["http://influxdb:8086"]
  database = "influx"
```

### 5. Dashboard
We can import Dashboard from grafana dashboard [Grafana Dashboards](https://grafana.com/grafana/dashboards/) for monitoring the metrics.
