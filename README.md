# Configuration and examples to run Portus on Oracle Linux 7 (or CentOS 7)

This repository contains sample configuration files for Docker Distribution (Registry) V2 and Portus to run as Docker containers.

## Assumptions:

1. Registry images will be stored in `/var/lib/registry/data` on the host
2. Certificates will be stored in `/var/lib/registry/certs` on the host
3. Registry configuration will be stored in `/var/lib/registry` on the host
4. Portus configuration will be stored in `/var/lib/registry/portus` on the host
5. MySQL or MariaDB data will be stored in `/var/lib/registry/db` on the host
6. You have a valid 3rd-party CA signed SSL certificate and corresponding private key for the host

These directories need to be created manually before starting any of the containers below.

## Step 1: Base Configuration Replacements

1. Checkout this repository to your local machine
2. Replace `your.registry.fqdn` in `registry/config.yml` with the actual fully qualified domain name of your host. 
3. Replace `your.registry.fqdn` in `portus/config-local.yml` with the actual fully qualified domain name of your host. 
4. Replace `registry.fqdn` in `portus/config-local.yml` with your domain name.
5. Place your SSL certificate chain _including intermediate CA certificates and private key_ at `/var/lib/registry/certs/server.crt`
6. Place your SSL private key at `/var/lib/registry/certs/server.key`
7. Copy `registry/config.yml` to `/var/lib/registry/config.yml`
8. Copy `portus/config-local.yml` to `/var/lib/registry/portus/config-local.yml`

*Note:* To successfully configure the Docker Registry to use the webhook to notify Portus, the full SSL chain including private key is required.

## Step 2: Start the MySQL or MariaDB Container

Use the following command to start a MySQL container to store the Portus data:

```
docker run -d --restart=always --name mysql \
 -e MYSQL_DATABASE=portus \
 -e MYSQL_USER=portus \
 -e MYSQL_PASSWORD=portus \
 -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
 -v /var/lib/registry/db:/var/lib/mysql \
 mysql:latest
```

This will initialize a MySQL database and generate a random root password. You will need to check the output of `docker logs mariadb` to find the root password generated. 
It will also create a database for Portus and a user named `portus` with the password `portus`. For security purposes, it is strongly recommended that you change this
command-line to use a different password and then change the `docker run` command that starts portus to use the same password.

You could also use MariaDB by replacing the `--name` and image (`mariadb:latest`) names.

## Step 3: Start the Registry Container

Use the following command to start the Docker Registry:

```
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /var/lib/registry/config.yml:/etc/docker/registry/config.yml \
  -v /var/lib/registry/data:/var/lib/registry \
  -v /var/lib/registry/certs:/certs \
  -v /var/lib/registry/auth:/auth \
registry:2
```

Remember that all the required directories must be created prior to starting the container.

## Step 4: Start the Portus Container

Use the following command to start Portus:

```
docker run -d --restart=always --name portus \
 -v /var/lib/registry/certs:/certificates \
 -v /var/lib/registry/certs/server.crt:/certificates/portus-ca.crt \
 -v /var/lib/registry/certs/server.key:/secrets/portus.key \
 -v /var/lib/registry/portus/config-local.yml:/srv/Portus/config/config-local.yml \
 --link mariadb:mariadb \
 -e MARIADB_SERVICE_HOST=mariadb \
 -e MARIADB_USER=portus \
 -e MARIADB_PASSWORD=portus \
 -e MARIADB_DATABASE=portus \
 -e PORTUS_SECRET_KEY_BASE=$(openssl rand -hex 64) \
 -e PORTUS_PORTUS_PASSWORD=$(openssl rand -hex 64) \
 -p 443:443 \
 -p 80:80 \
opensuse/portus:2.0
```

Switch `opensuse/portus:2.0` to `opensuse/portus:head` if you'd like to run the most recent build from GitHub.

*Note:* if you used a different method to create the MariaDB database or provided a custom username/password, you need to replace the default values in this command.

You should use `docker inspect portus` to find the `PORTUS_SECRET_KEY_BASE` and `PORTUS_PORTUS_PASSWORD` values used when the container was created. This will allow you to recreate the container using the existing database and maintain the correct `portus` user password (which is also stored in the database).
