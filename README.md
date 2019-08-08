# Shlink Docker image

[![Build Status](https://img.shields.io/travis/shlinkio/shlink-docker-image.svg?style=flat-square)](https://travis-ci.org/shlinkio/shlink-docker-image)
[![Docker build status](https://img.shields.io/docker/cloud/build/shlinkio/shlink.svg?style=flat-square)](https://hub.docker.com/r/shlinkio/shlink/)
[![Docker pulls](https://img.shields.io/docker/pulls/shlinkio/shlink.svg?style=flat-square)](https://hub.docker.com/r/shlinkio/shlink/)
[![Latest Stable Version](https://img.shields.io/github/tag/shlinkio/shlink-docker-image.svg?style=flat-square)](https://github.com/shlinkio/shlink-docker-image/releases/latest)
[![License](https://img.shields.io/github/license/shlinkio/shlink-docker-image.svg?style=flat-square)](https://github.com/shlinkio/shlink-docker-image/blob/master/LICENSE)
[![Paypal donate](https://img.shields.io/badge/Donate-paypal-blue.svg?style=flat-square&logo=paypal&colorA=aaaaaa)](https://acel.me/donate)

[Shlink](https://shlink.io) is a self-hosted URL shortener which includes both a REST and a CLI interface in order to interact with it.

This image provides an easy way to set up [shlink](https://shlink.io) on a container-based runtime.

It exposes a shlink instance served with [swoole](https://www.swoole.co.uk/), which persists data in a local [sqlite](https://www.sqlite.org/index.html) database.

## Usage

Shlink docker image exposes port `8080` in order to interact with its HTTP interface.

It also expects these two env vars to be provided, in order to properly generate short URLs at runtime.

* `SHORT_DOMAIN_HOST`: The custom short domain used for this shlink instance. For example **doma.in**.
* `SHORT_DOMAIN_SCHEMA`: Either **http** or **https**.

So based on this, to run shlink on a local docker service, you should run a command like this:

```bash
docker run --name shlink -p 8080:8080 -e SHORT_DOMAIN_HOST=doma.in -e SHORT_DOMAIN_SCHEMA=https shlinkio/shlink
```

### Interact with shlink's CLI on a running container.

Once the shlink container is running, you can interact with the CLI tool by running `shlink` with any of the supported commands.

For example, if the container is called `shlink_container`, you can generate a new API key with:

```bash
docker exec -it shlink_container shlink api-key:generate
```

Or you can list all tags with:

```bash
docker exec -it shlink_container shlink tag:list
```

Or process remaining visits with:

```bash
docker exec -it shlink_container shlink visit:process
```

All shlink commands will work the same way.

You can also list all available commands just by running this:

```bash
docker exec -it shlink_container shlink
```

## Use an external DB

The image comes with a working sqlite database, but in production you will probably want to usa a distributed database.

It is possible to use a set of env vars to make this shlink instance interact with an external MySQL or PostgreSQL database.

* `DB_DRIVER`: **[Mandatory]**. Use the value **mysql** or **postgres** to prevent the sqlite database to be used.
* `DB_NAME`: [Optional]. The database name to be used. Defaults to **shlink**.
* `DB_USER`: **[Mandatory]**. The username credential for the database server.
* `DB_PASSWORD`: **[Mandatory]**. The password credential for the database server.
* `DB_HOST`: **[Mandatory]**. The host name of the server running the database engine.
* `DB_PORT`: [Optional]. The port in which the database service is running.
    * Default value is based on the driver:
        * **mysql** -> `3306`
        * **postgres** -> `5432`

> PostgreSQL is supported since v1.16.1 of this image. Do not try to use it with previous versions.

Taking this into account, you could run shlink on a local docker service like this:

```bash
docker run --name shlink -p 8080:8080 -e SHORT_DOMAIN_HOST=doma.in -e SHORT_DOMAIN_SCHEMA=https -e DB_DRIVER=mysql -e DB_USER=root -e DB_PASSWORD=123abc -e DB_HOST=something.rds.amazonaws.com shlinkio/shlink
```

You could even link to a local database running on a different container:

```bash
docker run --name shlink -p 8080:8080 [...] -e DB_HOST=some_mysql_container --link some_mysql_container shlinkio/shlink
```

> If you have considered using SQLite but sharing the database file with a volume, read [this issue](https://github.com/shlinkio/shlink-docker-image/issues/40) first.

## Supported env vars

A few env vars have been already used in previous examples, but this image supports others that can be used to customize its behavior.

This is the complete list of supported env vars:

* `SHORT_DOMAIN_HOST`: The custom short domain used for this shlink instance. For example **doma.in**.
* `SHORT_DOMAIN_SCHEMA`: Either **http** or **https**.
* `SHORTCODE_CHARS`: A charset to use when building short codes. Only needed when using more than one shlink instance ([Multi instance considerations](#multi-instance-considerations)).
* `DB_DRIVER`: **sqlite** (which is the default value), **mysql** or **postgres**.
* `DB_NAME`: The database name to be used when using an external database driver. Defaults to **shlink**.
* `DB_USER`: The username credential to be used when using an external database driver.
* `DB_PASSWORD`: The password credential to be used when using an external database driver.
* `DB_HOST`: The host name of the database server  when using an external database driver.
* `DB_PORT`: The port in which the database service is running when using an external database driver. Defaults to **3306**.
* `DISABLE_TRACK_PARAM`: The name of a query param that can be used to visit short URLs avoiding the visit to be tracked. This feature won't be available if not value is provided.
* `DELETE_SHORT_URL_THRESHOLD`: The amount of visits on short URLs which will not allow them to be deleted. Defaults to `15`.
* `LOCALE`: Defines the default language for error pages when a user accesses a short URL which does not exist. Supported values are **es** and **en**. Defaults to **en**.
* `VALIDATE_URLS`: Boolean which tells if shlink should validate a status 20x (after following redirects) is returned when trying to shorten a URL. Defaults to `true`.
* `NOT_FOUND_REDIRECT_TO`: If a URL is provided here, when a user tries to access an invalid short URL, he/she will be redirected to this value. If this env var is not provided, the user will see a generic `404 - not found` page.
* `REDIS_SERVERS`: A comma-separated list of redis servers where Shlink locks are stored (locks are used to prevent some operations to be run more than once in parallel).

    This is important when running more than one Shlink instance ([Multi instance considerations](#multi-instance-considerations)). If not provided, Shlink stores locks on every instance separately.

    If more than one server is provided, Shlink will expect them to be configured as a [redis cluster](https://redis.io/topics/cluster-tutorial).

    In the future, these redis servers could be used for other caching operations performed by shlink.

An example using all env vars could look like this:

```bash
docker run \
    --name shlink \
    -p 8080:8080 \
    -e SHORT_DOMAIN_HOST=doma.in \
    -e SHORT_DOMAIN_SCHEMA=https \
    -e DB_DRIVER=mysql \
    -e DB_NAME=shlink \
    -e DB_USER=root \
    -e DB_PASSWORD=123abc \
    -e DB_HOST=something.rds.amazonaws.com \
    -e DB_PORT=3306 \
    -e DISABLE_TRACK_PARAM="no-track" \
    -e DELETE_SHORT_URL_THRESHOLD=30 \
    -e LOCALE=es \
    -e VALIDATE_URLS=false \
    -e "NOT_FOUND_REDIRECT_TO=https://www.google.com" \
    -e "REDIS_SERVERS=tcp://172.20.0.1:6379,tcp://172.20.0.2:6379" \
    shlinkio/shlink
```

## Provide config via volumes

Rather than providing custom configuration via env vars, it is also possible ot provide config files in json format.

Mounting a volume at `config/params` you will make shlink load all the files on it with the `.config.json` suffix.

The whole configuration should have this format, but it can be split into multiple files that will be merged:

```json
{
    "disable_track_param": "my_param",
    "delete_short_url_threshold": 30,
    "locale": "es",
    "short_domain_schema": "https",
    "short_domain_host": "doma.in",
    "validate_url": false,
    "not_found_redirect_to": "https://my-landing-page.com",
    "redis_servers": [
        "tcp://172.20.0.1:6379",
        "tcp://172.20.0.2:6379"
    ],
    "db_config": {
        "driver": "pdo_mysql",
        "dbname": "shlink",
        "user": "root",
        "password": "123abc",
        "host": "something.rds.amazonaws.com",
        "port": "3306"
    }
}
```

> This is internally parsed to how shlink expects the config. If you are using a version previous to 1.17.0, this parser is not present and you need to provide a config structure like the one [documented previously](https://github.com/shlinkio/shlink-docker-image/tree/v1.16.3#provide-config-via-volumes).

Once created just run shlink with the volume:

```bash
docker run --name shlink -p 8080:8080 -v ${PWD}/my/config/dir:/etc/shlink/config/params shlinkio/shlink
```

## Multi instance considerations

These are some considerations to take into account when running multiple instances of shlink.

* The first time shlink is run, it generates a charset used to generate short codes, which is a shuffled base62 charset.

    If you are using several shlink instances, you will probably want all of them to use the same charset.

    You can get a shuffled base62 charset by going to [https://shlink.io/short-code-chars](https://shlink.io/short-code-chars), and then you just need to pass it to all shlink instances using the `SHORTCODE_CHARS` env var.

    If you don't do this, each shlink instance will use a different charset. However this shouldn't be a problem in practice, since the chances to get a collision will be very low.

* Some operations performed by Shlink should never be run more than once at the same time (like creating the database for the first time, or downloading the GeoLite2 database). For this reason, Shlink uses a locking system.

    However, these locks are locally scoped to each Shlink instance by default.

    You can (and should) make the locks to be shared by all Shlink instances by using a redis server/cluster. Just define the `REDIS_SERVERS` env var with the list of servers.

## Versions

Currently, the versions of this image match the shlink version it contains.

For example, installing `shlinkio/shlink:1.15.0`, you will get an image containing shlink v1.15.0.

There are no official shlink images previous to v1.15.0.
