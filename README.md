<h1>This is a Fork from <a href="https://github.com/markshust/docker-magento">markshust/docker-magento</a></h1>
<h1>This fork is only for macOS silicon</h1>
<br>
<div align="center">
  <img src="https://img.shields.io/badge/magento-2.X-brightgreen.svg?logo=magento&longCache=true" alt="Supported Magento Versions" />
  <img src="https://img.shields.io/badge/apple%20silicon%20support-yes-brightgreen" alt="Apple Silicon Only" />
  <a href="https://opensource.org/licenses/MIT" target="_blank"><img src="https://img.shields.io/badge/license-MIT-blue.svg" /></a>
</div>

## Table of contents

- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Setup](#setup)
- [Updates](#updates)
- [Misc Info](#misc-info)
- [Custom CLI Commands](#custom-cli-commands)
- [Docker Hub](#docker-hub)
- [Known Issues](#known-issues)
- [License](#license)

## Prerequisites

This setup assumes you are running Docker on a computer with at least 6GB of RAM allocated to Docker, a dual-core, and an SSD hard drive. [Download & Install Docker Desktop](https://www.docker.com/products/docker-desktop).

This configuration has been tested on Mac & Linux. Windows is supported through the use of Docker on WSL.


## Usage

This configuration is intended to be used as a Docker-based development environment for Magento 2.

Folders:

- `images`: Docker images for nginx and php
- `compose`: sample setups with Docker Compose

## Setup

### Automated Setup (New Project)

```bash
# Create your project directory then go into it:
mkdir -p ~/Sites/magento
cd $_

# Run this automated one-liner from the directory you want to install your project.
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/onelinesetup | bash -s -- magento.test 2.4.7 community
```

The `magento.test` above defines the hostname to use, and the `2.4.7` defines the Magento version to install. Note that since we need a write to `/etc/hosts` for DNS resolution, you will be prompted for your system password during setup.

After the one-liner above completes running, you should be able to access your site at `https://magento.test`.

#### Install sample data

After the above installation is complete, run the following lines to install sample data:

```bash
bin/magento sampledata:deploy
bin/magento setup:upgrade
```

### Manual Setup

Same result as the one-liner above. Just replace `magento.test` references with the hostname that you wish to use.

#### New Projects

```bash
# Create your project directory then go into it:
mkdir -p ~/Sites/magento
cd $_

# Download the Docker Compose template:
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/template | bash

# Download the version of Magento you want to use with:
bin/download 2.4.7 community
# You can specify the version and type (community, enterprise, mageos, mageos-nightly, mageos-mirror, mageos-hypernode-mirror, or mageos-maxcluster-mirror).
# The mageos type is an alias for mageos-mirror.
# If no arguments are passed, "2.4.7" and "community" are the default values used.

# or for Magento core development:
# bin/start --no-dev
# bin/setup-composer-auth
# bin/cli git clone git@github.com:magento/magento2.git .
# bin/cli git checkout 2.4-develop
# bin/composer install

# Want to install Magento <2.4.6? In bin/setup-install, replace the lines:
#  --elasticsearch-host="$ES_HOST" \
#  --elasticsearch-port="$ES_PORT" \
#  --opensearch-host="$OPENSEARCH_HOST" \
#  --opensearch-port="$OPENSEARCH_PORT" \
#  --search-engine=opensearch \
# with:
#  --elasticsearch-host="$ES_HOST" \
#  --elasticsearch-port="$ES_PORT" \
#  --search-engine=elasticsearch7 \

# Run the setup installer for Magento:
bin/setup magento.test

open https://magento.test
```

#### Existing Projects

```bash
# Create your project directory then go into it:
mkdir -p ~/Sites/magento
cd $_

# Download the Docker Compose template:
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/template | bash

# Take a backup of your existing database:
bin/mysqldump > ~/Sites/existing/magento.sql

# Replace with existing source code of your existing Magento instance:
cp -R ~/Sites/existing src
# or: git clone git@github.com:myrepo.git src

# Start some containers, copy files to them and then restart the containers:
bin/start --no-dev
bin/copytocontainer --all ## Initial copy will take a few minutes...

# If your vendor directory was empty, populate it with:
bin/composer install

# Import existing database:
bin/mysql < ../existing/magento.sql

# Update database connection details to use the above Docker MySQL credentials:
# Also note: creds for the MySQL server are defined at startup from env/db.env
# vi src/app/etc/env.php

# Import app-specific environment settings:
bin/magento app:config:import

# Create a DNS host entry and setup Magento base url
bin/setup-domain yoursite.test

bin/restart

open https://magento.test
```

### Elasticsearch vs OpenSearch
OpenSearch is set as the default search engine when setting up this project. Follow the instructions below if you want to use Elasticsearch instead:
1. Comment out or remove the `opensearch` container in both the [`compose.yaml`](https://github.com/markshust/docker-magento/blob/master/compose/compose.yaml#L69-L84) and [`compose.healthcheck.yaml`](https://github.com/markshust/docker-magento/blob/master/compose/compose.healthcheck.yaml#L36-L41) files
2. Uncomment the `elasticsearch` container in both the [`compose.yaml`](https://github.com/markshust/docker-magento/blob/master/compose/compose.yaml#L86-L106) and [`compose.healthcheck.yaml`](https://github.com/markshust/docker-magento/blob/master/compose/compose.healthcheck.yaml#L43-L48) files
3. Update the `bin/setup-install` command to use the Elasticsearch ratther than OpenSearch. Change:

```
--opensearch-host="$OPENSEARCH_HOST" \
--opensearch-port="$OPENSEARCH_PORT" \
```

to:

```
--elasticsearch-host="$ES_HOST" \
--elasticsearch-port="$ES_PORT" \

```

## Updates

To update your project to the latest version of `docker-magento`, run:

```
bin/update
```

We recommend keeping your docker config files in version control, so you can monitor the changes to files after updates. After reviewing the code updates and ensuring they updated as intended, run `bin/restart` to restart your containers to have the new configuration take effect.

It is recommended to keep your root docker config files in one repository, and your Magento code setup in another. This ensures the Magento base path lives at the top of one specific repository, which makes automated build pipelines and deployments easy to manage, and maintains compatibility with projects such as Magento Cloud.


### Caching

For an improved developer experience, caches are automatically refreshed when related files are updated, courtesy of [cache-clean](https://github.com/mage2tv/magento-cache-clean). This means you can keep all of the standard Magento caches enabled, and this script will only clear the specific caches needed, and only when necessary.

To disable this functionality, uncomment the last line in the `bin/start` file to disable the watcher.

### Database

The hostname of each service is the name of the service within the `compose.yaml` file. So for example, MySQL's hostname is `db` (not `localhost`) when accessing it from within a Docker container. Elasticsearch's hostname is `elasticsearch`.

To connect to the MySQL CLI tool of the Docker instance, run:

```
bin/mysql
```

You can use the `bin/mysql` script to import a database, for example a file stored in your local host directory at `magento.sql`:

```
bin/mysql < magento.sql
```

You also can use `bin/mysqldump` to export the database. The file will appear in your local host directory at `magento.sql`:

```
bin/mysqldump > magento.sql
```

> Getting an "Access denied, you need (at least one of) the SUPER privilege(s) for this operation." message when running one of the above lines? Try running it as root with:
> ```
> bin/clinotty mysql -hdb -uroot -pmagento magento < src/backup.sql
> ```
> You can also remove the DEFINER lines from the MySQL backup file with:
> ```
> sed 's/\sDEFINER=`[^`]*`@`[^`]*`//g' -i src/backup.sql
> ```

### Email / Mailcatcher

View emails sent locally through Mailcatcher by visiting [http://{yourdomain}:1080](http://{yourdomain}:1080). During development, it's easiest to test emails using a third-party module such as [Mageplaza's SMTP module](https://github.com/mageplaza/magento-2-smtp). In order to use mailcatcher, set the mailserver host to `mailcatcher` and set port to `1025`. Note that this port is different from the mailcatcher interface to read the emails.

### Redis

Redis is now the default cache and session storage engine, and is automatically configured & enabled when running `bin/setup` on new installs.

Use the following lines to enable Redis on existing installs:

**Enable for Cache:**

`bin/magento setup:config:set --cache-backend=redis --cache-backend-redis-server=redis --cache-backend-redis-db=0`

**Enable for Full Page Cache:**

`bin/magento setup:config:set --page-cache=redis --page-cache-redis-server=redis --page-cache-redis-db=1`

**Enable for Session:**

`bin/magento setup:config:set --session-save=redis --session-save-redis-host=redis --session-save-redis-log-level=4 --session-save-redis-db=2`

You may also monitor Redis by running: `bin/redis redis-cli monitor`

For more information about Redis usage with Magento, <a href="https://devdocs.magento.com/guides/v2.4/config-guide/redis/redis-session.html" target="_blank">see the DevDocs</a>.

### Blackfire.io

These docker images have built-in support for Blackfire.io. To use it, first register your server ID and token with the Blackfire agent:

```
bin/root blackfire-agent --register --server-id={YOUR_SERVER_ID} --server-token={YOUR_SERVER_TOKEN}
```

Next, open up the `bin/start` helper script and uncomment the line:

```
#bin/root /etc/init.d/blackfire-agent start
```

Finally, restart the containers with `bin/restart`. After doing so, everything is now configured and you can use a browser extension to profile your Magento store with Blackfire.

## Custom CLI Commands

- `bin/analyse`: Run `phpstan analyse` within the container to statically analyse code, passing in directory to analyse. Ex. `bin/analyse app/code`
- `bin/bash`: Drop into the bash prompt of your Docker container. The `phpfpm` container should be mainly used to access the filesystem within Docker.
- `bin/blackfire`: Disable or enable Blackfire. Accepts argument `disable`, `enable`, or `status`. Ex. `bin/blackfire enable`
- `bin/cache-clean`: Access the [cache-clean](https://github.com/mage2tv/magento-cache-clean) CLI. Note the watcher is automatically started at startup in `bin/start`. Ex. `bin/cache-clean config full_page`
- `bin/check-dependencies`: Provides helpful recommendations for dependencies tailored to the chosen Magento version.
- `bin/cli`: Run any CLI command without going into the bash prompt. Ex. `bin/cli ls`
- `bin/clinotty`: Run any CLI command with no TTY. Ex. `bin/clinotty chmod u+x bin/magento`
- `bin/cliq`: The same as `bin/cli`, but pipes all output to `/dev/null`. Useful for a quiet CLI, or implementing long-running processes.
- `bin/composer`: Run the composer binary. Ex. `bin/composer install`
- `bin/configure-linux`: Adds the Docker container's IP address to the system's `/etc/hosts` file if it's not already present. Additionally, it prompts the user to open port 9003 for Xdebug if desired.
- `bin/copyfromcontainer`: Copy folders or files from container to host. Ex. `bin/copyfromcontainer vendor`
- `bin/copytocontainer`: Copy folders or files from host to container. Ex. `bin/copytocontainer --all`
- `bin/create-user`: Create either an admin user or customer account.
- `bin/cron`: Start or stop the cron service. Ex. `bin/cron start`
- `bin/debug-cli`: Enable Xdebug for bin/magento, with an optional argument of the IDE key. Defaults to PHPSTORM Ex. `bin/debug-cli enable PHPSTORM`
- `bin/deploy`: Runs the standard Magento deployment process commands. Pass extra locales besides `en_US` via an optional argument. Ex. `bin/deploy nl_NL`
- `bin/dev-test-run`: Facilitates running PHPUnit tests for a specified test type (e.g., integration). It expects the test type as the first argument and passes any additional arguments to PHPUnit, allowing for customization of test runs. If no test type is provided, it prompts the user to specify one before exiting.
- `bin/dev-urn-catalog-generate`: Generate URN's for PhpStorm and remap paths to local host. Restart PhpStorm after running this command.
- `bin/devconsole`: Alias for `bin/n98-magerun2 dev:console`
- `bin/docker-compose`: Support V1 (`docker-compose`) and V2 (`docker compose`) docker compose command, and use custom configuration files, such as `compose.yml` and `compose.dev.yml`
- `bin/docker-stats`: Display container name and container ID, status for CPU, memory usage(in MiB and %), and memory limit of currently-running Docker containers.
- `bin/download`: Download specific Magento version from Composer to the container, with optional arguments of the version (2.4.7 [default]) and type ("community" [default], "enterprise", or "mageos"). Ex. `bin/download 2.4.7 enterprise`
- `bin/ece-patches`: Run the Cloud Patches CLI. Ex: `bin/ece-tools apply`
- `bin/fixowns`: This will fix filesystem ownerships within the container.
- `bin/fixperms`: This will fix filesystem permissions within the container.
- `bin/grunt`: Run the grunt binary. Ex. `bin/grunt exec`
- `bin/install-php-extensions`: Install PHP extension in the container. Ex. `bin/install-php-extensions sourceguardian`
- `bin/log`: Monitor the Magento log files. Pass no params to tail all files. Ex. `bin/log debug.log`
- `bin/magento`: Run the Magento CLI. Ex: `bin/magento cache:flush`
- `bin/magento-version`: Determine the Magento version installed in the current environment.
- `bin/mftf`: Run the Magento MFTF. Ex: `bin/mftf build:project`
- `bin/mysql`: Run the MySQL CLI with database config from `env/db.env`. Ex. `bin/mysql -e "EXPLAIN core_config_data"` or`bin/mysql < magento.sql`
- `bin/mysqldump`: Backup the Magento database. Ex. `bin/mysqldump > magento.sql`
- `bin/n98-magerun2`: Access the [n98-magerun2](https://github.com/netz98/n98-magerun2) CLI. Ex: `bin/n98-magerun2 dev:console`
- `bin/node`: Run the node binary. Ex. `bin/node --version`
- `bin/npm`: Run the npm binary. Ex. `bin/npm install`
- `bin/phpcbf`: Auto-fix PHP_CodeSniffer errors with Magento2 options. Ex. `bin/phpcbf <path-to-extension>`
- `bin/phpcs`: Run PHP_CodeSniffer with Magento2 options. Ex. `bin/phpcs <path-to-extension>`
- `bin/phpcs-json-report`: Run PHP_CodeSniffer with Magento2 options and save to `report.json` file. Ex. `bin/phpcs-json-report <path-to-extension>`
- `bin/pwa-studio`: (BETA) Start the PWA Studio server. Note that Chrome will throw SSL cert errors and not allow you to view the site, but Firefox will.
- `bin/redis`: Run a command from the redis container. Ex. `bin/redis redis-cli monitor`
- `bin/remove`: Remove all containers.
- `bin/removeall`: Remove all containers, networks, volumes, and images, calling `bin/stopall` before doing so.
- `bin/removenetwork`: Remove a network associated with the current directory's name.
- `bin/removevolumes`: Remove all volumes.
- `bin/restart`: Stop and then start all containers.
- `bin/root`: Run any CLI command as root without going into the bash prompt. Ex `bin/root apt-get install nano`
- `bin/rootnotty`: Run any CLI command as root with no TTY. Ex `bin/rootnotty chown -R app:app /var/www/html`
- `bin/setup`: Run the Magento setup process to install Magento from the source code, with optional domain name. Defaults to `magento.test`. Ex. `bin/setup magento.test`
- `bin/setup-composer-auth`: Setup authentication credentials for Composer.
- `bin/setup-domain`: Setup Magento domain name. Ex: `bin/setup-domain magento.test`
- `bin/setup-grunt`: Install and configure Grunt JavaScript task runner to compile .less files
- `bin/setup-install`: Automates the installation process for a Magento instance.
- `bin/setup-integration-tests`: Script to set up integration tests.
- `bin/setup-pwa-studio`: (BETA) Install PWA Studio (requires NodeJS and Yarn to be installed on the host machine). Pass in your base site domain, otherwise the default `master-7rqtwti-mfwmkrjfqvbjk.us-4.magentosite.cloud` will be used. Ex: `bin/setup-pwa-studio magento.test`.
- `bin/setup-pwa-studio-sampledata`: This script makes it easier to install Venia sample data. Pass in your base site domain, otherwise the default `master-7rqtwti-mfwmkrjfqvbjk.us-4.magentosite.cloud` will be used. Ex: `bin/setup-pwa-studio-sampledata magento.test`.
- `bin/setup-ssl`: Generate an SSL certificate for one or more domains. Ex. `bin/setup-ssl magento.test foo.test`
- `bin/setup-ssl-ca`: Generate a certificate authority and copy it to the host.
- `bin/spx`: Disable or enable output compression to enable or disbale SPX. Accepts params `disable` (default) or `enable`. Ex. `bin/spx enable`
- `bin/start`: Start all containers, good practice to use this instead of `docker-compose up -d`, as it may contain additional helpers.
- `bin/status`: Check the container status.
- `bin/stop`: Stop all project containers.
- `bin/stopall`: Stop all docker running containers
- `bin/update`: Update your project to the most recent version of `docker-magento`.
- `bin/xdebug`: Disable or enable Xdebug. Accepts argument `disable`, `enable`, or `status`. Ex. `bin/xdebug enable`


## Docker Hub

View Dockerfiles for the latest tags:

- [markoshust/magento-nginx (Docker Hub)](https://hub.docker.com/r/markoshust/magento-nginx/)
    - [`1.18`, `1.18-8`](images/nginx/1.18)
    - [`1.22`, `1.22-0`](images/nginx/1.22)
    - [`1.24`, `1.24-0`](images/nginx/1.24)
- [markoshust/magento-php (Docker Hub)](https://hub.docker.com/r/markoshust/magento-php/)
    - [`8.1-fpm`, `8.1-fpm-5`](images/php/8.1)
    - [`8.2-fpm`, `8.2-fpm-4`](images/php/8.2)
    - [`8.3-fpm`, `8.3-fpm-2`](images/php/8.3)
- [markoshust/magento-opensearch (Docker Hub)](https://hub.docker.com/r/markoshust/magento-opensearch/)
    - [`1.2`, `1.2-0`](images/opensearch/1.2)
    - [`2.5`, `2.5-1`](images/opensearch/2.5)
    - [`2.12`, `2.12-0`](images/opensearch/2.12)
- [markoshust/magento-elasticsearch (Docker Hub)](https://hub.docker.com/r/markoshust/magento-elasticsearch/)
    - [`7.16`, `7.16-0`](images/elasticsearch/7.16)
    - [`7.17`, `7.17-1`](images/elasticsearch/7.17)
    - [`8.4`, `8.4-0`](images/elasticsearch/8.4)
    - [`8.5`, `8.5-0`](images/elasticsearch/8.5)
    - [`8.7`, `8.7-0`](images/elasticsearch/8.7)
    - [`8.11`, `8.11-0`](images/elasticsearch/8.11)
    - [`8.13`, `8.13-0`](images/elasticsearch/8.13)
- [markoshust/magento-rabbitmq (Docker Hub)](https://hub.docker.com/r/markoshust/magento-rabbitmq/)
    - [`3.8`, `3.8-0`](images/rabbitmq/3.8)
    - [`3.9`, `3.9-0`](images/rabbitmq/3.9)
    - [`3.11`, `3.11-1`](images/rabbitmq/3.11)
    - [`3.12`, `3.12-0`](images/rabbitmq/3.12)
- [markoshust/ssh (Docker Hub)](https://hub.docker.com/r/markoshust/magento-ssh/)
    - [`latest`](images/ssh)


## Known Issues

None at the moment.

## License

[MIT](https://opensource.org/licenses/MIT)
