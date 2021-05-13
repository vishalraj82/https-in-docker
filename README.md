# HTTPS in docker 

With the evolution of docker, application development has taken to a new paradigm. Docker makes running any app pretty much easy. One of the benefits of docker is, if it runs on one machine, it will run everywhere, exactly the same.

Here we are going to explore how can bring the development experience to be closest to the production environment for better development experience. We will use wordpress as the app, mysql as the data storage and nginx as the HTTP proxy over the wordpress app.

#### Creating a basic docker-compose.yml with wordpress and mysql

We will start with the basic configuration of docker compose to have Wordpress and MySQL up and running.

```
version: '3.9'

services:
  blog:
    image: wordpress:5.7.1-php7.3-apache
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: blogdb
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html
  blogdb:
    image: mysql:5.7.34
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

With this configuration, the wordpress app can be accessed in browser via `http://localhost:8080`. But we want it to be closer to the production environment. For this we need

- A domain name to access the app
- HTTPS for the domain name

For the first part, we can make an entry in `/etc/hosts` files with `127.0.0.1 my-wordpress-blog.local`
With this, we can use the aforementioned domain name in browser to access the app, but the port `8080` and the protocol `HTTP` is still there.

For the second part, we will use a utility called [`mkcert`](https://github.com/FiloSottile/mkcert). Specifically we are going to use the pre-compiled binary from [here](https://github.com/FiloSottile/mkcert/releases). As of today, the latest version of `mkcert` is [`v1.4.3`](https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64)

Before downloading the `mkcert` utility, lets install its dependencies - `sudo apt install libnss3-tools -y`

Download the `mkcert` from the location mentioned above and move it to `/usr/local/bin`. Create a soft link to the binary with name `mkcert` and set the executable flag `chmod +x mkcert-v1.4.3-linux-amd64`

Lets move to the folder `https-in-docker/proxy/certs`. Here we will generate the self signed certificates

`mkcert -cert-file my-wordpress-blog.local.crt -key-file my-wordpress-blog.local.key my-wordpress-blog.local`

This will create the self signed SSL certificates for the domain name - `my-wordpress-blog.local`, which we will use further, to access the wordpress app.