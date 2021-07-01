# Dockerize PHP applications. #

This is a part of the local infrastructure project which aims to create easy to install and use environment for PHP development based on Ubuntu LTS.

1. [Ubuntu post-installation scripts](https://github.com/Vitalii-Luka/ubuntu_post_install_script) - install software,
clone repositories with `Docker infrastructure` and `Dockerizer for PHP` tool. Infrastructure is launched automatically
during setup and you do not need start it manually. Check this repo to get more info about what software is installed,
where the files are located and why we think this software is needed.

2. [Docker infrastructure](https://github.com/Vitalii-Luka/docker_infrastructure) - run [Traefik](https://traefik.io/)
reverse-proxy container with linked MySQL 5.6, 5.7, MariaDB 10.1, 10.3, phpMyAdmin and Mailhog containers.
Infrastructure is cloned and run automatically by the [Ubuntu post-installation scripts](https://github.com/Vitalii-Luka/ubuntu_post_install_script).
Check this repository for more information on how the infrastructure works, how to use xDebug, LiveReload etc.

3. `Dockerizer for PHP` (this repository) - install any Magento 2 version in 1
command. Add Docker files to your existing PHP projects in one command. This repository is cloned automatically
by the [Ubuntu post-installation scripts](https://github.com/Vitalii-Luka/ubuntu_post_install_script). Read below
to get more information on available commands and what this tool does.


## Preparing the tool ##

Works best without additional adjustments with systems installed by the [Ubuntu post-installation scripts](https://github.com/Vitalii-Luka/ubuntu_post_install_script).

To use this application, you must switch to PHP 7.3 or 7.4.

After cloning the repository (if you haven't run the commands mentioned above):
1) copy `./config/auth.json.sample` file to `./config/auth.json` and enter your credentials instead of the placeholders;
2) add your root password to the file `.env.local` (in the root folder of this app): `USER_ROOT_PASSWORD=<your pass here>`
3) run `composer install`

Other settings can be found in the file `.env.dist`. There you can find default database container name and information
about the folders where projects and SSL keys are stores. Move these environment settings to your `.env.local` file
if you need to customize them (especially database connection settings like DATABASE_HOST and DATABASE_PORT).


## Dockerize existing PHP applications ##

The `dockerize` command copies Docker files to the current folder and updates them as per project settings.
You will be asked to enter domains, choose PHP version, MySQL container and web root folder.
If you a mistype a PHP version or domain names - just re-run the command, it will overwrite existing Docker files.

Example usage in the fully interactive mode:

```bash
php ${PROJECTS_ROOT_DIR}dockerizer_for_php/bin/console dockerize
```

Example full usage with all parameters:

```bash
php ${PROJECTS_ROOT_DIR}dockerizer_for_php/bin/console dockerize --php=7.2 --mysql-container=mariadb103 --webroot='pub/' --domains='example.com www.example.com example-2.com www.example-2.com'
```

Magento 1 example with the custom web root:

```bash
php ${PROJECTS_ROOT_DIR}dockerizer_for_php/bin/console dockerize --php=5.6 --mysql-container=mysql56 --webroot='/' --domains='example.com www.example.com'
```

The file `/etc/hosts` is automatically updated with new domains. Traefik configuration is updated with the new SSL certificates.

Docker containers are not run automatically, so you can still edit configurations before running them.


## Using a custom Dockerfile ##

You can use custom Dockerfile based on the DockerHub Images if needed.

Example `docker-compose.yml` fragment (use `build` instead of `image`):

```yml
services:
  php-apache:
    container_name: example.com
    build:
      context: .
      dockerfile: docker/Dockerfile
      args:
        - EXECUTION_ENVIRONMENT=${EXECUTION_ENVIRONMENT}
```

Custom Dockerfile start:

```Dockerfile
ARG EXECUTION_ENVIRONMENT
FROM defaultvalue/php:7.4-${EXECUTION_ENVIRONMENT}
```


## Starting and stopping compositions in development mode ##

Please, refer the Docker and docker-compose documentation for information on docker commands.

Stop composition:

```bash
docker-compose down
docker-compose -f docker-compose-env.yml down
```

Start composition, especially after making any changed to the `.yml` files:

```bash
docker-compose up -d --force-recreate
docker-compose -f docker-compose-env.yml up -d --force-recreate
```

Rebuild container if Dockerfile was changed:

```bash
docker-compose up -d --force-recreate --build
docker-compose -f docker-compose-env.yml -d --force-recreate --build
```


## Adding more environments ##

We often need more then just a production environment - staging, test, development etc. Use the following command to
add more environments to your project:

```bash
php ${PROJECTS_ROOT_DIR}dockerizer_for_php/bin/console env:add <env_name>
```

This will:
- copy the `docker-compose-dev.yml` template and rename it (for example, to `docker-compose-staging.yml`);
- modify the `mkcert` information string in the `docker-compose.file`;
- generate new SSL certificates for all domains from the `docker-compose*.yml` files;
- reconfigure `Traefik` and `virtual-host.conf`, update `.htaccess`;
- add new entries to the `/etc/hosts` file if needed.

Container name is based on the main (actually, the first) container name from the `docker-compose.yml`
file suffixed with the `-<env_name>`. This allows running multiple environments at the same time.

Composition is not restarted automatically, so you can edit everything before finally running it.

#### CAUTION! ####

1) SSL certificates are not specially prefixed! If you add two environments in different folders (let's say
`dev` and `staging`) then the certificates will be overwritten for one of them.
Instead of manually configuring the certificates you can first copy new `docker-compose-dev.yml`
to the folder where you're going to add new `staging` environment.

2) If your composition runs other named services (e.g., those that have `container_name`)
then you'll have to rename them manually by moving those services to the new environment file and changing
the container name like this is done for the PHP container. You're welcome to automate this as well.


## Hardware testing ##

The `test:hardware` sets up Magento and perform a number of tasks to test environment:
- build images to warm up Docker images cache because they aren't on the Dockerhub yet;
- install Magento 2 (2.0.18 > PHP 5.6, 2.1.18 > PHP 7.0, 2.2.11 > PHP 7.1, 2.3.2 > PHP 7.2, 2.3.4 > PHP 7.3);
- commit Docker files;
- test Dockerizer's `env:add` - stop containers, dockerize with another domains, add env, and run composition;
- run `deploy:mode:set production`;
- run `setup:perf:generate-fixtures` to generate data for performance testing
(medium size profile for v2.2.0+, small for previous version because generating data takes too much time);
- run `indexer:reindex`.

Usage for hardware test and Dockerizer self-test (install all instances and ensure they work fine):

```bash
php bin/console test:hardware
```

Log files are written to `./dockerizer_for_php/var/log/`.


## Generating SSL certificates ##

Manually generated SSL certificates must be places in `~/misc/certs/` or other folder defined in the
`SSL_CERTIFICATES_DIR` environment variable (see below about variables). This folder is linked to a Docker containers
with Traefik and with web server. This can be shared with VirtualBox or other virtualization tools if needed.

If the SSL certificates are not valid in Chrome/Firefox when you first run Magento then run the following command and restart the browser:

```bash
mkcert -install
```


## Helpful Aliases ##

A number of helpful aliases are added to your `~/.bash_aliases` file if you use the Ubuntu post-installation script.
They make using Docker and Dockerizer even easier.


## Manual installation (Ubuntu-like and MacOS) ##

Manually clone infrastructure and Dockerizer repositories to the `~/misc/apps/` folder. The folder `~/misc/certs/`
must be created as well. Set required environment variables like this (use `~/.bash_profile` for MacOS):

```bash
echo "
export PROJECTS_ROOT_DIR=${HOME}/misc/apps/
export SSL_CERTIFICATES_DIR=${HOME}/misc/certs/
export EXECUTION_ENVIRONMENT=development" >> ~/.bash_aliases
```

All other commands must be executed taking this location into account, e.g. like this:

```bash
cp ~/misc/apps/dockerizer_for_php/config/auth.json.sample ~/misc/apps/dockerizer_for_php/config/auth.json
php ~/misc/apps/dockerizer_for_php/bin/console magento:setup 2.3.4 --domains="example.com www.example.com"
php ~/misc/apps/dockerizer_for_php/bin/console dockerize
```


## Environment variables explained ##

- `PROJECTS_ROOT_DIR` - your projects location. All projects are deployed here;
- `SSL_CERTIFICATES_DIR` - directory with certificates to mount to the web server container and Traefik reverse-proxy;
- `EXECUTION_ENVIRONMENT` - either `development` or `production`. Used to pull Docker image from [Dockerhub](https://hub.docker.com/repository/docker/defaultvalue/php).


## Images testing before release ##

The command `test:dockerfiles` and the option `--execution-environment` (`-e`) in other commands are used to install
Magento with Sample Data using the local Dockerfiles from the [Docker infrastructure](https://github.com/DefaultValue/docker_infrastructure)
project. This option MUST NOT be used while installing Magento - use custom Dockerfiles based on the prebuild images
as described [above](https://github.com/DefaultValue/dockerizer_for_php#using-a-custom-dockerfile).


## For MacOS Users ##

Configuration for `docker-sync` is included. Though, working with Docker on Mac is anyway difficult, slow
and drains battery due to the files sync overhead.

@TODO: write how to run containers and what should be changed in the docker-compose* files on Mac.
@TODO: MacOS support is experimental and require additional testing. Will be tested more and improved in the future releases.


