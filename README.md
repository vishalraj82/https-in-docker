# HTTPS in docker 

As a web application developer, one of the most common challenge faced is, not having the local development environment close enough to the production environment. While there can be many aspects to this, in this post we will focus on the following two

- Having a domain name, instead of something like `http://localhost:8080`
- Having a valid `HTTPS` certificate on the local development machine

Also, I am going to use docker for this demonstration. So lets begin.

I am going to use Wordpress as my example application. So lets head over the the docker page for [Wordpress](https://hub.docker.com/_/wordpress). Scrolling to the bottom shows us a sample configuration which can be used with docker-compose to have Wordpress running with [MySQL](https://hub.docker.com/_/mysql).

Open a terminal and make a folder with name, say `wordpress-with-http` and move inside it. Now create a file with name `docker-compose.yml` and paste the contents copied from [here](https://hub.docker.com/_/wordpress). 

```
version: '3.9'

services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

Save and exit from the file. Now lets start the docker containers - `docker-compose up`. This will fire up the Wordpress and MySQL containers with appropriate configuration. Open a browser and type the URL - [`http://localhost:8080`](http://localhost:8080) and press enter. At this point, we can see that we have now successfully opened up the Wordpress setup page.

In order to access the Wordpress app over domain name, we need to make an entry in the `/etc/hosts` files and add the following at the end of the file - `127.0.0.1  my-wordpress-blog.local`. After this, now we should be able to access - [`http://my-wordpress-blog.local:8080`](http://my-wordpress-blog.local:8080) in browser.

In order to have HTTPS in the local development environment, we will use a utility called [mkcert](https://github.com/FiloSottile/mkcert). In order to have mkcert, we first need to install the dependency - `libnss3-tools`. Open a terminal and run - `sudo apt install libnss3-tools -y`. Now lets download the pre-built mkcert binary from the [github releases page](https://github.com/FiloSottile/mkcert/releases). Download the appropriate binary. Since I am using Ubuntu on my develoment machine, so I will use [mkcert-v1.4.3-linux-amd64](https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64). Download the binary file and move it to `/usr/local/bin`. We also need to make the file executable - `chmod +x mkcert-v1.4.3-linux-amd64`. Now lets create a softlink with name - `mkcert` - `ln -s mkcert-v1.4.3-linux-amd64 mkcert`. The first step is to become a valid Certificate Authority for local machine - `mkcert -install`. This will install the root CA for local machine.

Now lets get back to generating self-signed SSL certificates. Lets move back to our development folder `wordpress-with-https`. Here we will create directory `proxy` and inside it `certs` and `conf`. Lets move inside `proxy/certs` and generate the certificates.

```
vishalr@ubuntu ~/wordpress-with-https> mkcert \
>  -cert-file my-wordpress-blog.local.crt \
>  -key-file my-wordpress-blog.local.key \
>  my-wordpress-blog.local

Created a new certificate valid for the following names
 - "my-wordpress-blog.local"

The certiciate is at my-wordpress-blog.local.crt and the key at "my-wordpress-blog.local.key"

It will expire on 14 August 2021

vishalr@ubuntu ~/wordpress-with-https>
```

This will generate the SSL key and certificate file which is valid for domain - [`my-wordpress-blog.local`](http://my-wordpress-blog.local). Now lets modify the contents of file `docker-compose.yml` to use nginx as the proxy. Add the following contents under `services` tag.

```
services:
  proxy:
    image: nginx:1.19.10-alpine
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./proxy/conf/nginx.conf:/etc/nginx/nginx.conf
      - ./proxy/certs:/etc/nginx/certs
    depends_on:
      - wordpress
```

Now lets focus on the nginx configuration to use it as proxy. Edit the file `wordpress-with-https/proxy/conf/nginx.conf` and add the following configuration.

```
events {
  worker_connections 1024;
}

http {
  server {
    listen 80;
    server_name my-wordpress-blog.local;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl;
    server_name my-wordpress-blog.local;

    ssl_certificate /etc/nginx/certs/my-wordpress-blog.local.crt;
    ssl_certificate_key /etc/nginx/certs/my-wordpress-blog.local.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
      proxy_buffering off;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Port $server_port;

      proxy_pass http://wordpress;
    }
}
```

If the docker containers are still running, press `Ctrl + C` to stop them. Now lets fire up the docker containers with the updated contents in `docker-compose.yml`. If all has been done correctly, we should have everything ready. Now lets open the browser and enter the url - [`http://my-wordpress-blog.local`](http://my-wordpress-blog.local). This should redirect to [`https://my-wordpress-blog.local`](https://my-wordpress-blog.local). Now if you look at the top left of the browser, the lock icon is green, which means that the browser has accepted our locally generated self signed ssl certificates.

The github repository can be found [here](https://github.com/vishalraj82/https-in-docker)
