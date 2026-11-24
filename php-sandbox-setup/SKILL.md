---
name: php-sandbox-setup
description: Set up this Docker sandbox for Laravel or WordPress work: PHP, Composer, Node, and MySQL or Postgres, at the versions the project declares.
disable-model-invocation: true
---

# PHP sandbox setup

You are already inside the Docker sandbox: configure this machine. It runs Ubuntu with passwordless `sudo` and has no systemd (PID 1 is `tini`), so services start through `service`. Every shell sources `/etc/sandbox-persistent.sh`, which makes it the place for anything that must survive a sandbox restart. Node comes preinstalled.

The project is the working directory. When it holds no `composer.json` or `package.json`, take the **fresh** branch each step names.

## 1. Gather version evidence

List every manifest outside vendored trees:

```bash
find . \( -name vendor -o -name node_modules \) -prune -o \( -name composer.json -o -name package.json \) -print
```

WordPress repos often keep `package.json` in a theme or plugin folder, so a manifest away from the root still counts. Read every source below that exists:

| Need | Evidence |
| --- | --- |
| Framework | `laravel/framework` in `composer.json`; `wp-config.php`, `wp-content/`, or `roots/wordpress` / `johnpbloch/wordpress` (Bedrock) |
| PHP | `.php-version`, `.tool-versions`, `composer.json` `config.platform.php`, Docker / Sail / DDEV / Lando config, CI workflow `php-version`, `composer.json` `require.php`, the locked `laravel/framework` version, `Requires PHP:` headers in theme `style.css` and plugin main files |
| Node and package manager | `.nvmrc`, `.node-version`, `.tool-versions`, `package.json` `engines.node` / `volta` / `packageManager`, the lockfile present |
| Database | `.env` / `.env.example` `DB_*`, `wp-config.php` `DB_*` constants, Docker service images such as `mysql:8.0` or `postgres:16` |
| Redis | `REDIS_*`, or `redis` as `CACHE_STORE`, `QUEUE_CONNECTION` or `SESSION_DRIVER` |

Classify each need by its evidence:

- **Pinned**: one source names a single version (PHP major.minor, Node major, database major). Resolved.
- **Range**: only a constraint such as `^8.2` or `>=20`.
- **Conflicting**: sources disagree.
- **Missing**: nothing says.

For every unresolved need, form a recommendation: the version Docker or CI runs, otherwise the highest version that satisfies every constraint, including the PHP ceiling of the locked framework version.

Done when every need has a value and the file it came from, or a classification and recommendation bound for step 2.

**Fresh**: framework, PHP and database are missing; Node defaults to the preinstalled one.

## 2. Ask the user

Ask in one AskUserQuestion call, recommended option first and labelled `(Recommended)`:

- **Database**, every run: MySQL or Postgres. Recommend what the evidence shows. For WordPress, recommend MySQL and say why: WordPress core speaks only MySQL/MariaDB.
- **PHP version**, unless pinned. Offer the recommendation and the other versions the constraint allows.
- **Node version**, unless pinned or no `package.json` exists.
- **Framework**, fresh branch only: Laravel or WordPress.

Name both files in any question about conflicting evidence. Overflow past four questions goes in a second call.

Done when every need has a single value.

## 3. Install PHP, Composer and CLI tools

Every PHP package comes from Ondřej Surý's repository at `packages.sury.org`, so add it before installing any PHP. It serves Ubuntu releases as well as Debian, and its packages replace Ubuntu's own PHP packages.

```bash
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update && sudo apt-get install -y lsb-release ca-certificates curl
curl -fsSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb
echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
sudo apt-get update
apt-cache policy php<ver>-cli
```

The repository is ready when `apt-cache policy php<ver>-cli` shows a candidate from `packages.sury.org`. If the sandbox's release has no `dists/<codename>` there, stop and tell the user.

Install `php<ver>-<ext>` for each extension in the baseline plus the database driver:

- **Laravel**: `cli bcmath curl gd intl mbstring readline sqlite3 xml zip`
- **WordPress**: `cli curl gd imagick intl mbstring readline xml zip`
- **Driver**: `mysql` or `pgsql`; add `redis` when step 1 found Redis evidence.

With more than one PHP installed, select the chosen one: `sudo update-alternatives --set php /usr/bin/php<ver>`.

Install Composer through the signed installer:

```bash
curl -fsSL https://getcomposer.org/installer -o /tmp/composer-setup.php
echo "$(curl -fsSL https://composer.github.io/installer.sig)  /tmp/composer-setup.php" | sha384sum -c -
sudo php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

Then let the project name the remaining extensions: run `composer check-platform-reqs --lock` beside each `composer.lock` (plain `composer check-platform-reqs` where only `composer.json` exists), install every `missing` extension as `php<ver>-<ext>`, and rerun until it passes.

For WordPress, install WP-CLI:

```bash
curl -fsSL -o /tmp/wp-cli.phar https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
sudo install -m 755 /tmp/wp-cli.phar /usr/local/bin/wp
```

Done when `php -v` shows the chosen version, every `check-platform-reqs` run passes, and `composer --version` (plus `wp --info` for WordPress) runs.

**Fresh** Laravel: also `composer global require laravel/installer`, and append `export PATH="$HOME/.config/composer/vendor/bin:$PATH"` to `/etc/sandbox-persistent.sh`.

## 4. Install Node

Keep the preinstalled Node when `node -v` satisfies the chosen version. Otherwise install it into `/usr/local/bin`, which shadows `/usr/bin/node` and leaves it intact:

```bash
sudo npm install -g n && sudo n <major> && hash -r
```

Pick the package manager from `packageManager`, else the lockfile (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, otherwise npm), and install pnpm or yarn with `sudo npm install -g <pm>@<version>`, taking the version from `packageManager` when it names one.

Done when `node -v` satisfies the chosen version and `<pm> --version` runs.

## 5. Install and start the database

Install `mysql-server` or `postgresql` from Ubuntu's repositories unless step 2 settled a different major. A Postgres major comes from PGDG: `sudo apt-get install -y postgresql-common && sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y`, then `postgresql-<major>`. A MySQL major outside Ubuntu's line goes back to the user as a question: does Ubuntu's version serve?

A postinst that cannot start the service leaves the package half-configured. Start the service as below, then run `sudo dpkg --configure -a`.

Start it and register the guarded start line in `/etc/sandbox-persistent.sh` so it returns after a restart:

```bash
sudo service mysql start   # postgresql: sudo service postgresql start
grep -q 'service mysql start' /etc/sandbox-persistent.sh || \
  echo 'pgrep -x mysqld >/dev/null || sudo service mysql start >/dev/null 2>&1' >> /etc/sandbox-persistent.sh
# postgresql: pgrep -x postgres ... sudo service postgresql start
```

Create the database and user the project already declares in `.env` or `wp-config.php`:

```bash
sudo mysql -e "CREATE DATABASE IF NOT EXISTS \`<db>\`; CREATE USER IF NOT EXISTS '<user>'@'%' IDENTIFIED BY '<pass>'; GRANT ALL ON \`<db>\`.* TO '<user>'@'%';"
sudo -u postgres psql -c "CREATE ROLE \"<user>\" LOGIN CREATEDB PASSWORD '<pass>';" && sudo -u postgres createdb -O "<user>" "<db>"
```

Credentials that need changing:

- A Docker service name as host (`mysql`, `pgsql`, `db`, `redis`) becomes `127.0.0.1`.
- `root` with an empty password becomes a dedicated user named after the database, with a generated password, because Ubuntu's MySQL root authenticates by socket only.
- No declared credentials (fresh branch included): database and user named after the project directory, generated password.

Write changed credentials into a file only when git does not track it (`git ls-files --error-unmatch <file>` fails). For a tracked file, such as a committed `wp-config.php`, list the needed change in the report.

With Redis evidence, install `redis-server`, start it with `sudo service redis-server start`, and register a guarded start line the same way.

Done when the project's credentials connect over TCP: `mysql -h127.0.0.1 -u<user> -p<pass> <db> -e 'SELECT 1'` or `PGPASSWORD=<pass> psql -h 127.0.0.1 -U <user> -d <db> -c 'SELECT 1'`, and `redis-cli ping` returns `PONG` when Redis was installed.

## 6. Install project dependencies

**Fresh**: skip to step 7.

- `composer install` beside each `composer.json`.
- In each `package.json` directory, install by lockfile: `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --frozen-lockfile` (Yarn 1) or `yarn install --immutable` (Yarn 2+); with no lockfile, `<pm> install`.
- **Laravel**: copy `.env.example` to `.env` when `.env` is missing, run `php artisan key:generate` when `APP_KEY` is empty, then `php artisan migrate`.
- **WordPress**: with core present and no `wp-config.php`, run `wp config create` using the step 5 credentials, then `wp db check`. A repo without core (only `wp-content`) goes in the report.

Done when every install exits 0 and `php artisan migrate:status` or `wp db check` succeeds.

## 7. Report

Rerun each step's done-check, since later installs can shift defaults. Then report:

- A table of PHP, Composer, Node, package manager, database and Redis: installed version, source (file or user answer), check status.
- Database credentials, and every file you edited.
- Lines added to `/etc/sandbox-persistent.sh`.
- Work left for the user: changes to tracked files, asset builds, database dumps to import, WordPress core download or install.
