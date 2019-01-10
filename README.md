# Deploying a Strapi API on AWS (EC2 & RDS & S3)
> Goal: build a HTTPS-secured Strapi API with dev & staging & prod modes

Everything done in the following is [AWS Free Tier](https://aws.amazon.com/free/) eligible.
Make sure you have already skimmed the [Strapi docs](https://strapi.io/documentation/3.x.x) before you start.

## § Create EC2 instance / RDS instance / S3 bucket

> You can skim this section if you are familiar with AWS

### ⊙ EC2
1. Click `Launch Instance` button
2. Choose AMI: `Ubuntu Server 18.04 LTS (HVM), SSD Volume Type`
3. Choose Instance Type: `General purpose, t2.micro`
4. Configure Instance: *as you like*
5. Add Storage: `General Purpose SSD (gp2), 8 GB`
6. Add Tags: *as you like*
7. Configure Security Group:
    * SSH (22) - `My IP` or `Anywhere` or *as you like*
    * HTTP (80) - `Anywhere`
    * HTTPS (443) - `Anywhere`
8. Review: click `Launch` button, then a modal pops up. If you are a:
    * Newbie: Choose `Create a new key pair` named `strapi-cms`, download it as `strapi-cms.pem`
    * Veteran: *as you like*
9. Finally, click `Launch Instances` button

### ⊙ RDS
1. Click `Create database` button
2. Select engine: `PostgreSQL`
3. Choose use case: *as you like*
4. Specify DB details:
    * DB engine version: `PostgreSQL 10.x-R1`
    * DB instance class: `db.t2.micro`
    * Multi-AZ deployment: `No`
    * Storage: `General Purpose (SSD), 20 GB`
    * DB instance identifier: *as you like*
    * Master username: *as you like*
    * Password: *as you like, recommend https://passwordsgenerator.net*
5. Configure advanced settings
    * Public accessibility: `Yes` *(that's why you need a super strong password)*
    * Database name: `strapi`
    * Monitoring & Maintenance window: *choose an idle peroid in your timezone*
    * Click `Create database` button
6. Instance Details panel - Security groups
    * Edit inbound rules: PostgreSQL (5432) - `Anywhere`
7. Create databases for dev & staging modes (GUI recommend: https://dbeaver.io)
    * Development mode: `strapi_dev`
    * Staging mode: `strapi_staging`
    * Production mode: `strapi` *(already exists)*

### ⊙ S3
0. Click `Create bucket` button
1. Name and region
    * Bucket name: *as you like*
2. Configure options: *as you like*
3. Set permissions
    * - [ ] `Block new public ACLs and uploading public objects (Recommended)`
    * - [ ] `Remove public access granted through public ACLs (Recommended)`
    * - [x] `Block new public bucket policies (Recommended)`
    * - [x] `Block public and cross-account access if bucket has public policies (Recommended)`
    * `Do not grant Amazon S3 Log Delivery group write access to this bucket`
4. Review: click `Create bucket` button

## § Point your domain to EC2
Point the A / CNAME records to the EC2's IPv4 Public IP / Public DNS (IPv4), such as:
* Development mode: `dev-cms.yourdomain.com`
* Staging mode: `staging-cms.yourdomain.com`
* Production mode: `cms.yourdomain.com`

## § Warm up EC2

### ⊙ Login to EC2 & update
Switch to the directory where the key pair `strapi-cms.pem` locates:

```shell
$ ssh -i strapi-cms.pem ubuntu@<ec2-public-ip>
$ sudo apt update
```

### ⊙ Install Nginx

```shell
$ sudo apt install nginx
```

### ⊙ Install nvm & Node.js & PM2
* Install [nvm](https://github.com/creationix/nvm#install-script):
    ```shell
    $ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
    ```
* Install Node.js:
    ```shell
    $ nvm install 10
    ```
* Install [PM2](https://pm2.io):
    ```shell
    $ npm i pm2 -g
    ```

### ⊙ Install Certbot
According to https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx :

```shell
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository universe
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install python-certbot-nginx
```

## § Create a local Strapi project

### ⊙ Installation and initialization
Firstly, install Strapi (refer to https://strapi.io/documentation/3.x.x/getting-started/installation.html):

```shell
$ npm i -g strapi@alpha
```

Secondary, create a new project (refer to https://strapi.io/documentation/3.x.x/cli/CLI.html#strapi-new):

```shell
$ strapi new strapi-cms \
    --dbclient=postgres \
    --dbport=5432 \
    --dbhost=<RDS endpoint> \
    --dbname=strapi_dev \
    --dbusername=<RDS master username> \
    --dbpassword=<RDS password>
```

Finally, **DO NOT** rush to `strapi start` for now.
There are some preparatory procedures to accomplish.
(By the way, you may be interested in https://strapi.io/documentation/3.x.x/advanced/usage-tracking.html)

### ⊙ Complete database configuration
> If you prefer best practices like https://12factor.net/config and https://github.com/i0natan/nodebestpractices/blob/master/sections/projectstructre/configguide.md , you'd better use tools like [dotenv](https://www.npmjs.com/package/dotenv) instead.

`strapi-cms/config/environments/development/database.json` has been completed during the initialization.
Complete `strapi-cms/config/environments/{staging|production}/database.json` based on it. For example:

```js
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "strapi-hook-bookshelf",
      "settings": {
        "client": "postgres",
        "host": "<RDS endpoint>",
        "port": 5432,
        "database": "<strapi_staging|strapi>",
        "username": "<RDS master username>",
        "password": "<RDS password>"
      },
      "options": {
        "ssl": false
      }
    }
  }
}
```

### ⊙ Fix listening ports
Modify `port` in `strapi-cms/config/environments/{staging|production}/server.json`:
* development: 1337 (default)
* staging: 1338
* production: 1339

### ⊙ Install S3 upload plugin
> Refer to https://strapi.io/documentation/3.x.x/guides/upload.html#install-providers

```shell
$ npm i -S strapi-provider-upload-aws-s3@alpha
```

### ⊙ Add npm scripts for staging & production mode
> You can use [PM2 - Ecosystem File](https://pm2.io/doc/en/runtime/guide/ecosystem-file/) instead if you'd like to

`npm start` is for development mode by default.
You can set `NODE_ENV` before it according to https://strapi.io/documentation/3.x.x/guides/deployment.html .
However, for the sake of compatibility, [cross-env](https://www.npmjs.com/package/cross-env) is introduced.

```shell
$ npm i -D cross-env
```

So `strapi-cms/package.json` may look like:

```js
{
  ...
  "scripts": {
    "setup": "cd admin && npm run setup",
    "start": "node server.js",                         // for development mode
    "staging": "cross-env NODE_ENV=staging npm start", // for staging mode 
    "prod": "cross-env NODE_ENV=production npm start", // for production mode
    "strapi": "node_modules/strapi/bin/strapi.js",
    "lint": "node_modules/.bin/eslint api/**/*.js config/**/*.js plugins/**/*.js",
    "postinstall": "node node_modules/strapi/lib/utils/post-install.js"
  },
  ...
}
```

### ⊙ Add `nginx.conf` to the project root

> If you prefer best practices using `/etc/nginx/{sites-available|sites-enabled}`, you may need help from https://nginxconfig.io or https://github.com/h5bp/server-configs-nginx . I prefer single file `nginx.conf` because of simplicity.

Create `strapi-cms/nginx.conf` with the config below, replace all `yourdomain.com` with yours.

```nginx
# The original template is generated by https://nginxconfig.io/?0.domain=yourdomain.com&0.path=%2Fhome%2Fubuntu%2Fstrapi-cms&0.document_root=%2Fhome%2Fubuntu%2Fstrapi-cms%2Fpublic&0.redirect=false&0.email=info@yourdomain.com&0.php=false&0.proxy&0.proxy_pass=http:%2F%2F127.0.0.1:1337&0.root=false&expires_assets=&expires_media=&expires_svg=&expires_fonts=&access_log=off%20%23%20use%20PM2%20logs&error_log=off%20%23%20use%20PM2%20logs&file_structure=unified
user www-data;
pid /run/nginx.pid;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
  multi_accept on;
  worker_connections 65535;
}

http {
  charset utf-8;
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  server_tokens off;
  types_hash_max_size 2048;
  client_max_body_size 10M;

  # MIME
  include mime.types;
  default_type application/octet-stream;

  # disabled logging, use `PM2 logs` instead
  access_log off;
  error_log off;

  # gzip
  gzip on;
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;

  # SSL
  ssl_session_timeout 1d;
  ssl_session_cache shared:SSL:50m;
  ssl_session_tickets off;

  # modern configuration
  ssl_protocols TLSv1.2;
  ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256;
  ssl_prefer_server_ciphers on;

  # OCSP Stapling
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 208.67.222.222 208.67.220.220 valid=60s;
  resolver_timeout 2s;

  # load configs
  include /etc/nginx/conf.d/*.conf;

  #####################################################
  ## the above are stable, the below would be custom ##
  #####################################################
  
  map $host $strapi_cms {
    cms.yourdomain.com 127.0.0.1:1339;
    staging-cms.yourdomain.com 127.0.0.1:1338;
    dev-cms.yourdomain.com 127.0.0.1:1337;
  }
  
  # single SSL certificate for multiple domains
  ssl_certificate /etc/letsencrypt/live/cms.yourdomain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/cms.yourdomain.com/privkey.pem;
  ssl_trusted_certificate /etc/letsencrypt/live/cms.yourdomain.com/fullchain.pem;

  # https://github.com/strapi/strapi/issues/2424#issuecomment-445964719
  map $http_x_forwarded_host $custom_forwarded_host {
    default "$host";
    strapi "strapi";
  }

  server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
  
    server_name cms.yourdomain.com staging-cms.yourdomain.com dev-cms.yourdomain.com;
  
    # reverse proxy
    location / {
      proxy_pass http://$strapi_cms$request_uri;
      proxy_http_version  1.1;
      proxy_cache_bypass  $http_upgrade;
      
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $custom_forwarded_host;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Forwarded-Port $server_port;
    }
  
    # security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # . files
    location ~ /\.(?!well-known) {
      deny all;
    }
  }

  # http => https
  server {
    listen 80;
    listen [::]:80;
  
    server_name cms.yourdomain.com staging-cms.yourdomain.com dev-cms.yourdomain.com;
  
    # ACME-challenge
    location ^~ /.well-known/acme-challenge/ {
      root /var/www/_letsencrypt;
    }
  
    location / {
      return 301 https://$server_name$request_uri;
    }
  }
}
```

### ⊙ Run
```shell
$ cd strapi-cms && strapi start
```

Your browser will open http://localhost:1337 automatically later.
Create the admin with a strong password.

### ⊙ Complete S3 settings
Visit [PLUGINS > Files Upload](http://localhost:1337/admin/plugins/upload/configurations/development) to complete the S3 settings for all modes.

### ⊙ GraphQL
If you'd like to equip GraphQL, visit [GENERAL > Marketplace](http://localhost:1337/admin/install-plugin) to download.
Refer to https://strapi.io/documentation/3.x.x/guides/graphql.html for further info.

### ⊙ Push to a [Github free private repo](https://blog.github.com/2019-01-07-new-year-new-github/) (or Gitlab, etc)

```json
$ git init
$ git add -A
$ git commit -m 'init'
$ git remote add origin <private-git-repo-url>
$ git push -u origin master
```

## § Deploy Strapi on EC2

### ⊙ Pull the repo and install npm dependencies
```shell
$ cd ~
$ git clone <private-git-repo-url>
$ cd strapi-cms
$ npm i
```

### ⊙ Run all modes with PM2
Thanks to [this Stack Overflow comment](https://stackoverflow.com/questions/31579509/can-pm2-run-an-npm-start-script/37775318#comment64893202_37775318), you can run npm scripts with PM2:

```shell
$ pm2 start npm --name "strapi-dev" -- start
$ pm2 start npm --name "strapi-staging" -- run staging
$ pm2 start npm --name "strapi-prod" -- run prod
```

Also, you need to set up the [PM2 Startup Hook](https://pm2.io/doc/en/runtime/guide/startup-hook/) in case of reboot:

```shell
$ pm2 startup
$ pm2 save
```

You can check if the startup hook works by rebooting the machine if you'd like to:

```shell
$ sudo reboot

$ ssh -i strapi-cms.pem ubuntu@<ec2-public-ip>
$ pm2 ls
```

### ⊙ Replace `/etc/nginx/nginx.conf` with yours

```shell
$ cd ~/strapi-cms
$ sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
$ sudo cp nginx.conf /etc/nginx/nginx.conf
```

### ⊙ Obtain SSL certificates
According to https://nginxconfig.io :

```shell
# HTTPS - certbot (before first run): create ACME-challenge common directory
$ sudo mkdir -p /var/www/_letsencrypt && chown www-data /var/www/_letsencrypt

# HTTPS - certbot (before first run): disable SSL directives
$ sudo sed -i -r 's/(listen .*443)/\1;#/g; s/(ssl_(certificate|certificate_key|trusted_certificate) )/#;#\1/g' /etc/nginx/nginx.conf

# Reload Nginx config
$ sudo systemctl reload nginx

# HTTPS - certbot: obtain certificates
$ sudo certbot certonly
    --webroot -n --agree-tos --force-renewal \
    -w /var/www/_letsencrypt \
    --email <your-email> \
    -d cms.<yourdomain.com> \
    -d staging-cms.<yourdomain.com> \
    -d dev-cms.<yourdomain.com>

# HTTPS - certbot (after first run): enable SSL directives
$ sudo sed -i -r 's/#?;#//g' /etc/nginx/nginx.conf

# Reload Nginx config again
$ sudo systemctl reload nginx
```

All done! Visit https://{cms|staging-cms|dev-cms}.yourdomain.com to see if it works.

### ⊙ Summary
Now you have:
* A Strapi API running on dev & staging & prod mode simultaneously
* PM2-guarded processes with reboot startup hooks
* Forced HTTPS-secured traffic for all
* Auto-renew SSL certificates for free
