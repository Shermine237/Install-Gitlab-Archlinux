<h1>INSTAL GITLAB ON ARCHLINUX BAREMETAL</h1>
<br/>
<h2>Components<h2>
<ul>
    <li>Postgres</li>
    <li>Redis</li>
    <li>Ruby</li>
    <li>Gitlab</li>
    <li>Nginx</li>
</ul>
<br/>
<h2>Installation</h2>
<ol>
    <li>Postgres</li>
<br/>
<p>
Install postgres
[RUN] sudo pacman -Sy postgresql

Run service
[RUN] sudo systemctl enable postgresql.service
[RUN] sudo systemctl start postgresql.service

Set a password to postgres user
[RUN] sudo passwd postgres

Login to postres
[RUN] su -l postgres

Init database
[RUN] initdb -D /var/lib/postgres/data

Create gitlab database and user
[RUN] psql -d template1
[RUN] CREATE USER gitlab WITH PASSWORD 'your_password_here';
[RUN] ALTER USER gitlab SUPERUSER;
[RUN] CREATE DATABASE gitlabhq_production OWNER gitlab;
[RUN] \q
</p>

    <li>Redis</li>
<br/>
<p>
Install redis
[RUN] sudo pacman -Sy redis

Configuration
[RUN] vim /etc/redis/redis.conf
[EDIT] unixsocket /run/redis/redis.sock
[EDIT] unixsocketperm 770

Start redis service
[RUN] sudo systemctl enable redis.service
[RUN] sudo systemctl start redis.service
</p>

    <li>Ruby</li>
<br/>
<p>
Install ruby
[RUN] sudo pacman -Sy ruby
</p>

    <li>Gitlab</li>
<br/>
<p>
Install gitlab
[RUN] sudo pacman -Sy gitlab

Configure host and port
[RUN] sudo vim /etc/webapps/gitlab/gitlab.yml

Custom port for Puma
[RUN] sudo vim /etc/webapps/gitlab/puma.rb
[EDIT] bind 'unix:///run/gitlab/gitlab.socket'
[EDIT] bind 'tcp://127.0.0.1:8080'

Configure Secret strings
[RUN] sudo su
[RUN] mkdir /etc/webapps/gitlab
[RUN] mkdir /etc/webapps/gitlab-shell
[RUN] hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab/secret
[RUN] chmod 640 /etc/webapps/gitlab/secret
[RUN] hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab-shell/secret
[RUN] chmod 640 /etc/webapps/gitlab-shell/secret
[RUN] su - user

Configure redis
[RUN] vim /etc/webapps/gitlab/resque.yml
[EDIT]
development:
  url: unix:/run/redis/redis.sock
test:
  url: unix:/run/redis/redis.sock
production:
  url: unix:/run/redis/redis.sock

Configure database
[RUN] vim /etc/webapps/gitlab/database.yml
[EDIT]
#
# PRODUCTION
#
production:
  adapter: postgresql
  encoding: unicode
  database: gitlabhq_production
  pool: 10
  username: gitlab
  password: "your_password_here"
  # host: localhost
  # port: 5432
  socket: /run/postgresql/.s.PGSQL.5432

Run gitlab-gitaly.service
[RUN] sudo systemctl enable gitlab-gitaly.service
[RUN] sudo systemctl start gitlab-gitaly.service

Initialize gitlab database
[RUN] cd /usr/share/webapps/gitlab
[RUN] sudo -u gitlab $(cat environment | xargs) bundle-2.7 exec rake gitlab:setup
</p>

    <li>Nginx</li>
<br/>
<p>
Install nginx
sudo pacman -Sy nginx

Run nginx
[RUN] sudo systemctl enable nginx.service
[RUN] sudo systemctl start nginx.service

Configure
[RUN] sudo mkdir /etc/nginx/sites-available/
[RUN] sudo vim /etc/nginx/sites-available/gitlab
[EDIT]
upstream gitlab-workhorse {
  server unix:/run/gitlab/gitlab-workhorse.socket fail_timeout=0;
}

server {
  listen 80;                  # IPv4 HTTP
  #listen 443 ssl http2;      # uncomment to enable IPv4 HTTPS + HTTP/2
  #listen [::]:80;            # uncomment to enable IPv6 HTTP
  #listen [::]:443 ssl http2; # uncomment to enable IPv6 HTTPS + HTTP/2
  server_name example.com;

  access_log  /var/log/gitlab/nginx_access.log;
  error_log   /var/log/gitlab/nginx_error.log;

  #ssl_certificate ssl/example.com.crt;
  #ssl_certificate_key ssl/example.com.key;

  location ~ ^/(assets)/ {
    root /usr/share/webapps/gitlab/public;
    gzip_static on; # to serve pre-gzipped version
    expires max;
    add_header Cache-Control public;
  }

  location / {
      # unlimited upload size in nginx (so the setting in GitLab applies)
      client_max_body_size 0;

      # proxy timeout should match the timeout value set in /etc/webapps/gitlab/puma.rb
      proxy_read_timeout 60;
      proxy_connect_timeout 60;
      proxy_redirect off;

      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;

      #proxy_set_header X-Forwarded-Ssl on;

      proxy_pass http://gitlab-workhorse;
  }

  error_page 404 /404.html;
  error_page 422 /422.html;
  error_page 500 /500.html;
  error_page 502 /502.html;
  error_page 503 /503.html;
  location ~ ^/(404|422|500|502|503)\.html$ {
    root /usr/share/webapps/gitlab/public;
    internal;
  }
}

Add gitlab site to nginx
[RUN] vim /etc/nginx/nginx.conf
[EDIT ADD IN HTTP BLOCK]
include /etc/nginx/sites-available/*;

Restart nginx
[RUN] sudo systemctl restart nginx.service

Run gitlab

start gitlab service target
[RUN] sudo systemctl enable gitlab.target
[RUN] sudo systemctl start gitlab.target
</p>

    <li>run gitlab</li>
<br/>
<p>
Visit localhost or server ip
</p>
</ol>
