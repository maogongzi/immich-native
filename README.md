# Native Immich

This repository provides instructions and helper scripts to install [Immich](https://github.com/immich-app/immich) without Docker, natively.

### Notes

 * This is tested on Ubuntu 22.04 (on both x86 and aarch64) as the host distro, but it will be similar on other distros.

 * If your distro is not running Python v3.10 or v3.11 (e.g., Ubuntu 24.04), you may need to force a poetry update. Find `if false` in `install.sh`, and set it to true.

 * This guide installs Immich to `/var/lib/immich`. To change it, replace it to the directory you want in this README and `install.sh`'s `$IMMICH_PATH`.

 * The [install.sh](install.sh) script currently is using Immich v1.105.1. It should be noted that due to the fast-evolving nature of Immich, the install script may get broken if you replace the `$TAG` to something more recent.

 * `mimalloc` is deliberately disabled as this is a native install and sharing system library makes more sense.

 * `pgvector` is used instead of `pgvecto.rs` that the official Immich uses to remove an additional Rust build dependency.

 * Microservice and machine-learning's host is opened to 0.0.0.0 in the default configuration. This behavior is changed to only accept 127.0.0.1 during installation. Only the main Immich service's port, 3001, is opened to 0.0.0.0.

 * Only the basic CPU configuration is used. Hardware-acceleration such as CUDA is unsupported. In my personal experience, importing about 10K photos on a x86 processor doesn't take an unreasonable amount of time (less than 30 minutes).

## 1. Install dependencies

 * [Node.js](https://github.com/nodesource/distributions)

 * [PostgreSQL](https://www.postgresql.org/download/linux)

 * [Redis](https://redis.io/docs/install/install-redis/install-redis-on-linux)

As the time of writing, Node.js v20 LTS, PostgreSQL 16 and Redis 7.2.4 was used.

 * [pgvector](https://github.com/pgvector/pgvector)

pgvector is included in the official PostgreSQL's APT repository:

``` bash
sudo apt install postgresql(-16)-pgvector
```

 * [FFmpeg](https://github.com/FFmpeg/FFmpeg)

Immich uses FFmpeg to process media.

Either install FFmpeg using APT by `sudo apt install ffmpeg` (not recommended due to Ubuntu shipping older versions),

or use [FFmpeg Static Builds](https://johnvansickle.com/ffmpeg) and install it to `/usr/bin`.

### Other APT packages

``` bash
sudo apt install python3-venv python3-dev uuid-runtime
```

A separate Python's virtualenv will be stored to `/var/lib/immich`.

## 2. Prepare `immich` user

This guide isolates Immich to run on a separate `immich` user.

This provides basic permission isolation and protection.

``` bash
sudo adduser \
  --home /var/lib/immich/home \
  --shell=/sbin/nologin \
  --no-create-home \
  --disabled-password \
  --disabled-login \
  immich
sudo mkdir -p /var/lib/immich
sudo chown immich:immich /var/lib/immich
sudo chmod 700 /var/lib/immich
```

## 3. Prepare PostgreSQL DB

Create a strong random string to be used with PostgreSQL immich database.

You need to save this and write to the `env` file later.

``` bash
sudo -u postgres psql
postgres=# create database immich;
postgres=# create user immich with encrypted password 'YOUR_STRONG_RANDOM_PW';
postgres=# grant all privileges on database immich to immich;
postgrse=# ALTER USER immich WITH SUPERUSER;
postgres=# \q
```

## 4. Prepare `env`

Save the [env](env) file to `/var/lib/immich`, and configure on your own.

You'll only have to set `DB_PASSWORD`.

``` bash
sudo cp env /var/lib/immich
sudo chown immich:immich /var/lib/immich/env
```

## 5. Build and install Immich

Clone this repository to somewhere anyone can access (like /tmp) and run `install.sh` as root.

Anytime Immich is updated, all you have to do is run it again.

In summary, the `install.sh` script does the following:

#### 1. Clones and builds Immich.

#### 2. Installs Immich to `/var/lib/immich` with minor patches.

  * Sets up a dedicated Python venv to `/var/lib/immich/app/machine-learning/venv`.

  * Replaces `/usr/src` to `/var/lib/immich`.

  * Limits listening host from 0.0.0.0 to 127.0.0.1. If you do not want this to happen (make sure you fully understand the security risks!), comment out the `sed` command in `install.sh`'s "Use 127.0.0.1" part.

## 6. Install systemd services

Because the install script switches to the immich user during installation, you must install systemd services manually:

``` bash
sudo cp immich*.service /etc/systemd/system/
sudo systemctl daemon-reload
for i in immich*.service; do
  sudo systemctl enable $i
  sudo systemctl start $i
done
```

## Done!

Your Immich installation should be running at 3001 port, listening from localhost (127.0.0.1).

Immich will additionally use localhost's 3002 and 3003 ports.

Please add firewall rules and apply https proxy and secure your Immich instance.

## Uninstallation

``` bash
# Run as root!

# Remove Immich systemd services
for i in immich*.service; do
  systemctl stop $i
  systemctl disable $i
done
rm /etc/systemd/system/immich*.service
systemctl daemon-reload

# Remove Immich files
rm -rf /var/lib/immich

# Delete immich user
deluser immich

# Remove Immich DB
sudo -u postgres psql
postgres=# drop user immich;
postgres=# drop database immich;
postgres=# \q

# Optionally remove dependencies
# Review /var/log/apt/history.log and remove packages you've installed
```
