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
    
<h3>1- Postgres</h3>
<p>
Install postgres<br/>
<code>sudo pacman -Sy postgresql</code><br/>
Set a password to postgres user<br/>
<code>sudo passwd postgres</code><br/>
Login to postgres<br/>
<code>su - postgres</code><br/>
Init database<br/>
<code>initdb -D /var/lib/postgres/data</code><br/>
<code>su - $admin-user</code>

Run service<br/>
<code>sudo systemctl enable postgresql.service</code><br/>
<code>sudo systemctl start postgresql.service</code><br/>

Create gitlab database and user<br/>
<code>su - postgres</code><br/>
<code>psql -d template1</code><br/>
<code>CREATE USER gitlab WITH PASSWORD 'your_password_here';</code><br/>
<code>ALTER USER gitlab SUPERUSER;</code><br/>
<code>CREATE DATABASE gitlabhq_production OWNER gitlab;</code><br/>
<code>\q</code><br/>
</p>

<h3>2- Redis</h3>
<p>
Install redis<br/>
<code>sudo pacman -Sy redis</code><br/>

Configuration<br/>
<code>sudo vim /etc/redis/redis.conf</code><br/>
[EDIT] unixsocket /run/redis/redis.sock<br/>
[EDIT] unixsocketperm 770

Start redis service<br/>
<code>sudo systemctl enable redis.service</code><br/>
<code>sudo systemctl start redis.service</code>
</p>

<h3>3- Ruby</h3>
<p>
Install ruby<br/>
<code>sudo pacman -Sy ruby</code>
</p>

<h3>4- Gitlab</h3>
<p>
Install gitlab<br/>
<code>sudo pacman -Sy gitlab</code><br/>

Configure host and port<br/>
<code>sudo vim /etc/webapps/gitlab/gitlab.yml</code><br/>

Custom port for Puma<br/>
<code>sudo vim /etc/webapps/gitlab/puma.rb</code><br/>
[EDIT] bind 'unix:///run/gitlab/gitlab.socket'<br/>
[EDIT] bind 'tcp://127.0.0.1:8080'<br/>

Configure Secret strings<br/>
<code>sudo su</code><br/>
<code>mkdir /etc/webapps/gitlab</code><br/>
<code>mkdir /etc/webapps/gitlab-shell</code><br/>
<code>hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab/secret</code><br/>
<code>chmod 640 /etc/webapps/gitlab/secret</code><br/>
<code>hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab-shell/secret</code><br/>
<code>chmod 640 /etc/webapps/gitlab-shell/secret</code><br/>
<code>su - user</code><br/>

Configure redis<br/>
<code>vim /etc/webapps/gitlab/resque.yml</code><br/>
[EDIT]<br/>
development:<br/>
  url: unix:/run/redis/redis.sock<br/>
test:<br/>
  url: unix:/run/redis/redis.sock<br/>
production:<br/>
  url: unix:/run/redis/redis.sock<br/>

Configure database<br/>
<code>vim /etc/webapps/gitlab/database.yml</code><br/>
[EDIT]<br/>
#<br/>
#PRODUCTION<br/>
#<br/>
production:<br/>
  adapter: postgresql<br/>
  encoding: unicode<br/>
  database: gitlabhq_production<br/>
  pool: 10<br/>
  username: gitlab<br/>
  password: "your_password_here"<br/>
  #host: localhost<br/>
  #port: 5432<br/>
  socket: /run/postgresql/.s.PGSQL.5432<br/>

Run gitlab-gitaly.service<br/>
<code>sudo systemctl enable gitlab-gitaly.service</code><br/>
<code>sudo systemctl start gitlab-gitaly.service</code><br/>

Initialize gitlab database<br/>
<code>cd /usr/share/webapps/gitlab</code><br/>
<code>sudo -u gitlab $(cat environment | xargs) bundle exec rake gitlab:setup </code>
</p>

<h3>5- Nginx</h3>
<p>
Install nginx<br/>
<code>sudo pacman -Sy nginx</code><br/>

Run nginx<br/>
<code>sudo systemctl enable nginx.service</code><br/>
<code>sudo systemctl start nginx.service</code><br/>

Configure<br/>
<code>sudo mkdir /etc/nginx/sites-available/</code><br/>
<code>sudo vim /etc/nginx/sites-available/gitlab</code><br/>
[EDIT]<br/>
upstream gitlab-workhorse {<br/>
  server unix:/run/gitlab/gitlab-workhorse.socket fail_timeout=0;<br/>
}<br/>
<br/>
server {<br/>
  listen 80;                  # IPv4 HTTP<br/>
  #listen 443 ssl http2;      # uncomment to enable IPv4 HTTPS + HTTP/2<br/>
  #listen [::]:80;            # uncomment to enable IPv6 HTTP<br/>
  #listen [::]:443 ssl http2; # uncomment to enable IPv6 HTTPS + HTTP/2<br/>
  server_name example.com;<br/>
<br/>
  access_log  /var/log/gitlab/nginx_access.log;<br/>
  error_log   /var/log/gitlab/nginx_error.log;<br/>
<br/>
  #ssl_certificate ssl/example.com.crt;<br/>
  #ssl_certificate_key ssl/example.com.key;<br/>
<br/>
  location ~ ^/(assets)/ {<br/>
    root /usr/share/webapps/gitlab/public;<br/>
    gzip_static on; # to serve pre-gzipped version<br/>
    expires max;<br/>
    add_header Cache-Control public;<br/>
  }<br/>
<br/>
  location / {<br/>
      # unlimited upload size in nginx (so the setting in GitLab applies)<br/>
      client_max_body_size 0;<br/>
<br/>
      # proxy timeout should match the timeout value set in /etc/webapps/gitlab/puma.rb<br/>
      proxy_read_timeout 60;<br/>
      proxy_connect_timeout 60;<br/>
      proxy_redirect off;<br/>
<br/>
      proxy_set_header Host $http_host;<br/>
      proxy_set_header X-Real-IP $remote_addr;<br/>
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;<br/>
      proxy_set_header X-Forwarded-Proto $scheme;<br/>
<br/>
      #proxy_set_header X-Forwarded-Ssl on;<br/>
<br/>
      proxy_pass http://gitlab-workhorse;<br/>
  }<br/>
<br/>
  error_page 404 /404.html;<br/>
  error_page 422 /422.html;<br/>
  error_page 500 /500.html;<br/>
  error_page 502 /502.html;<br/>
  error_page 503 /503.html;<br/>
  location ~ ^/(404|422|500|502|503)\.html$ {<br/>
    root /usr/share/webapps/gitlab/public;<br/>
    internal;<br/>
  }<br/>
}<br/>
<br/>
Add gitlab site to nginx<br/>
<code>vim /etc/nginx/nginx.conf</code><br/>
[EDIT ADD IN HTTP BLOCK]<br/>
include /etc/nginx/sites-available/*;<br/>

Restart nginx<br/>
<code>sudo systemctl restart nginx.service</code><br/>

Run gitlab<br/>

start gitlab service target<br/>
<code>sudo systemctl enable gitlab.target</code><br/>
<code>sudo systemctl start gitlab.target</code><br/>
</p>

<h3>6- run gitlab</h3>
<p>
Visit localhost or server ip
</p>
