# PHP security backports for Ubuntu 18.04

Patched **PHP 8.1** packages for **Ubuntu 18.04 LTS (bionic)**, published as a
signed APT repository.

If your 18.04 servers got PHP 8.1 from Ondřej Surý's PPA (`ppa:ondrej/php`),
they stopped getting PHP updates long ago. The PPA no longer publishes for
18.04, and since July 2024 it contains no 18.04 packages at all. This
repository rebuilds Sury's current, patched PHP 8.1 packages for 18.04, so
those servers can be fixed with a normal `apt upgrade`.

> [!IMPORTANT]
> This is an unofficial, community rebuild. It is not affiliated with or
> endorsed by the PHP project, Ondřej Surý, or Canonical.
> Ubuntu 18.04 and PHP 8.1 are end-of-life: use this to stay patched while you
> move to Ubuntu 24.04 and a supported PHP, not as long-term support.

For Ubuntu 20.04, see the sister repository
[php-focal-backports](https://github.com/wmfairuz/php-focal-backports).

## Contents

- [What's fixed](#whats-fixed)
- [Is this for me?](#is-this-for-me)
- [Quick start](#quick-start) (one command)
- [Check that it worked](#check-that-it-worked)
- [What's in the repository](#whats-in-the-repository)
- [Can I trust these packages?](#can-i-trust-these-packages)
- [Removing the repository](#removing-the-repository)
- [Troubleshooting](#troubleshooting)
- [For maintainers](#for-maintainers)
- [Credits and license](#credits-and-license)

## What's fixed

| PHP | Package version in this repo     | What `php -v` shows | CVE-2026-109618 / CVE-2026-109619 |
| --- | -------------------------------- | ------------------- | --------------------------------- |
| 8.1 | `8.1.34-9+pbb1~ubuntu18.04.1`    | 8.1.34              | fixed (backport)                  |

CVE-2026-109618 and CVE-2026-109619 are the critical (CVSS 9.2) remote code
execution bugs fixed in PHP 8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 on 2026-10-21.
PHP no longer maintains 8.1; Sury backported the fix, which is why the version
number does not change. The package also carries every earlier PHP 8.1
security fix, up to and including 8.1.34.

Other PHP versions for 18.04 may follow.

## Is this for me?

**Yes, if** your server runs Ubuntu 18.04 on amd64 (x86-64) and already has
PHP 8.1, usually from Ondřej's PPA.

**No, if** your server is:

- **Ubuntu 20.04**: use [php-focal-backports](https://github.com/wmfairuz/php-focal-backports).
- **Ubuntu 22.04 or 24.04**: use Ondřej's PPA (`ppa:ondrej/php`) directly. It
  already has the fix.
- **arm64 / ARM**: only amd64 packages are built.
- **Any PHP other than 8.1 on 18.04**: not built here yet. Ubuntu's own
  `php7.2` is left alone.
- **18.04 without PHP 8.1 yet**: a fresh install needs a `php-common` package
  newer than Ubuntu 18.04's, which this repository does not ship yet. It
  upgrades existing PHP 8.1 installs only.
- **cPanel, Plesk, or a Docker image**: update PHP through those instead.

Find your release and architecture with:

```bash
lsb_release -cs        # must print: bionic
dpkg --print-architecture   # must print: amd64
```

## Quick start

Take a snapshot or backup of the server first: Ondřej's old 18.04 packages are
no longer downloadable, so you cannot `apt` your way back to them.

Then run **one command**:

```bash
curl -fsSL https://raw.githubusercontent.com/wmfairuz/php-bionic-backports/main/install.sh | sudo bash
```

It checks the server is Ubuntu 18.04 amd64, adds the repository after
verifying its signing key, shows which PHP packages it will upgrade and asks
before changing anything, upgrades only the PHP 8.1 packages you already have
(keeping your `php.ini` and pool files), restarts PHP-FPM / Apache, and
confirms PHP 8.1 is patched. It stops without changing anything if apt would
remove a package. Running it again is safe.

To run it unattended (for example from Ansible), skip the prompt:

```bash
curl -fsSL https://raw.githubusercontent.com/wmfairuz/php-bionic-backports/main/install.sh | sudo bash -s -- --yes
```

Read [`install.sh`](install.sh) before piping it to `sudo bash` if your policy
requires it. From then on, future fixes arrive with your normal
`sudo apt update && sudo apt upgrade`.

### Manual setup

The same steps by hand.

**1. Add the repository** (once per server):

```bash
curl -fsSL https://wmfairuz.github.io/php-bionic-backports/key.gpg \
  | sudo tee /usr/share/keyrings/php-bionic-backports.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/php-bionic-backports.gpg] https://wmfairuz.github.io/php-bionic-backports bionic main" \
  | sudo tee /etc/apt/sources.list.d/php-bionic-backports.list

sudo apt update
```

**2. See what will change** before changing anything:

```bash
apt list --upgradable 2>/dev/null | grep pbb
```

**3. Upgrade only the PHP 8.1 packages you already have** (nothing new is
installed, and other system packages are left alone):

```bash
sudo apt install --only-upgrade $(apt list --upgradable 2>/dev/null | grep '+pbb' | cut -d/ -f1)
```

If apt asks whether to keep or replace a modified config file (`php.ini`, a
pool file), answer **N** to keep yours.

**4. Restart PHP** so the running processes load the new code:

```bash
sudo systemctl restart php8.1-fpm
sudo systemctl restart apache2   # only if you use Apache with mod_php
```

From now on, future fixes arrive with your normal `sudo apt update && sudo apt upgrade`.

## Check that it worked

PHP 8.1 should now come from this repository (`pbb` in the version):

```bash
dpkg -l 'php8.1-common' | grep ^ii
```

```
ii  php8.1-common  8.1.34-9+pbb1~ubuntu18.04.1  amd64  documentation, examples and common module for PHP
```

Also confirm PHP-FPM came back up:

```bash
systemctl status php8.1-fpm --no-pager | grep -E '●|Active:'
```

For a full report, the
[`check-php-up-2026.sh`](https://github.com/wmfairuz/server-helper/blob/main/check-php-up-2026.sh)
script (read-only by default) lists every PHP on the server and should print
**PATCHED** for each one.

## What's in the repository

The same set of packages Sury publishes for PHP 8.1, so `apt` sees them as
newer versions of what you already have:

`cli`, `fpm`, `cgi`, `common`, `dev`, `embed`, `phpdbg`, `litespeed`,
`libapache2-mod-php8.1`, and the extensions `bcmath`, `bz2`, `curl`, `dba`,
`enchant`, `gd`, `gmp`, `imap`, `interbase`, `intl`, `ldap`, `mbstring`,
`mysql`, `odbc`, `opcache`, `pgsql`, `pspell`, `readline`, `snmp`, `soap`,
`sqlite3`, `sybase`, `tidy`, `xml`, `xsl`, `zip`.

**Not included:**

- Extensions Sury ships as separate source packages, such as `php8.1-redis`,
  `php8.1-imagick`, `php8.1-memcached`, `php8.1-xdebug`, `php8.1-mongodb`, or
  anything from PECL. The copies you already have keep working with these
  packages, and they are not affected by these CVEs.
- `php-common` and the unversioned `php`, `php-cli`, … metapackages. Your
  existing `php-common` (from Ondřej's PPA) is kept.

### One change from Sury's packages: telemetry is removed

Since February 2026, Sury's PHP-FPM and Apache module send the package name,
version and a timestamp (encrypted) to `telemetry.sury.org` over UDP port 53
every time they start, to check for updates. These builds leave that code out,
and the build fails if any package still contains the telemetry host name.
Nothing else in the source or packaging is changed.

## Can I trust these packages?

You are trusting this repository's build, so everything that goes into it is
public and checkable:

- **Source**: Sury's own source packages, downloaded from
  `packages.sury.org` and checked against the SHA-256 hashes pinned in
  [`sources/`](sources/). The build stops if any hash differs.
- **Build**: GitHub Actions runs [`build.sh`](build.sh) in a clean
  `ubuntu:18.04` container, against 18.04's own libraries. The workflow is
  [`.github/workflows/build.yml`](.github/workflows/build.yml); full build logs
  and checksums are attached to every
  [release](https://github.com/wmfairuz/php-bionic-backports/releases).
- **Signing**: the APT repository is signed by a dedicated key that exists only
  as a GitHub Actions secret (not the same key as php-focal-backports). Check
  that the key you installed is this one:

  ```bash
  gpg --no-default-keyring --keyring /usr/share/keyrings/php-bionic-backports.gpg --fingerprint
  ```

  The fingerprint must be `FC69 F242 2E46 9B01 55CD  9A12 F274 4485 1D5C AB41`
  (also in [`key.asc`](key.asc)).

- **Not done**: PHP's own test suite is not run, and the builds are not
  bit-for-bit reproducible.

## Removing the repository

```bash
sudo rm /etc/apt/sources.list.d/php-bionic-backports.list \
        /usr/share/keyrings/php-bionic-backports.gpg
sudo apt update
```

The installed packages stay as they are; they just stop receiving updates from
here. After moving a server to Ubuntu 22.04 or 24.04, remove this repository
and use Ondřej's PPA again.

## Troubleshooting

**`apt update` says the repository is not signed / `NO_PUBKEY`**:
the key file is missing or empty. Repeat step 1 and check its fingerprint
(see above).

**`404 Not Found` for `.../dists/bionic/...`**: the server is not on 18.04, or
the `.list` line has a typo. Only `bionic` exists.

**`apt list --upgradable` shows nothing with `pbb`**: either PHP 8.1 is already
up to date, or it is not installed. Check what you have with
`dpkg -l 'php*' | grep ^ii`.

**`php8.1-common : Depends: php-common (>= 1:81~)`**: the server has Ubuntu
18.04's own `php-common` (1:60), not Ondřej's. See
[Is this for me?](#is-this-for-me): fresh installs are not supported yet.

**`Repository '…ondrej/php…' changed its 'Label' value`**: Ondřej renamed his
PPA, and apt refuses to refresh it (keeping an old index whose files 404)
until you accept that. The installer does this for you; by hand, accept the
label change only:

```bash
sudo apt-get update -o Acquire::AllowReleaseInfoChange::Label=true
```

**`404 Not Found` for `ppa.launchpad.net/ondrej/php/...`**:
Ondřej's PPA still exists for 18.04 but no longer contains any packages, and
apt may keep using its old index. The installer only upgrades packages from
this repository and leaves the rest alone, but plain `apt upgrade` may still
trip over it. Once PHP 8.1 comes from here, remove the PPA:
`sudo add-apt-repository --remove ppa:ondrej/php && sudo apt update`.
Extensions you installed from it (`php8.1-redis`, …) stay installed and keep
working.

**apt wants to remove packages**: stop and read the list. This usually means
an extension from a separate source package (for example `php8.1-redis`) pins
an exact older `php8.1-common`. Open an issue with the full apt output.

**PHP-FPM fails to start after the upgrade**: run `sudo php-fpm8.1 -t` to see
the config error, and check `journalctl -u php8.1-fpm -n 50`.

Anything else: [open an issue](https://github.com/wmfairuz/php-bionic-backports/issues)
with your `lsb_release -a`, `dpkg -l 'php*' | grep ^ii`, and the full error.

## For maintainers

### Ship a new security fix

When Sury publishes a fixed revision of PHP 8.1:

```bash
scripts/update-sources.sh           # re-pin sources/ to Sury's current packages
git diff sources/                   # review: only the versions you expect
git commit -am "Update to Sury's <versions>" && git push
```

Wait for the build on `main` to pass, then publish:

```bash
git tag -a v$(date +%Y.%m.%d) -m "<what changed>"
git push origin v$(date +%Y.%m.%d)
```

The tag rebuilds everything, signs the repository, deploys it to GitHub Pages,
and creates a release with logs and checksums. Update the table in
[What's fixed](#whats-fixed) when versions change.

To rebuild the **same** sources (for example after a `build.sh` fix), bump
`REVISION` in `.github/workflows/build.yml` instead, so the `+pbbN` suffix goes
up and servers see an upgrade.

To add another PHP version, add its `sources/php<ver>.sha256` (copy one from
php-focal-backports, or run `scripts/update-sources.sh` after creating an empty
file), add it to the build matrix, and widen `PHP_RE` in `install.sh`.

### Build locally

```bash
mkdir -p sources-dl out
(cd sources-dl && while read -r _ f; do
   curl -fsSLO "https://packages.sury.org/php/pool/main/p/php8.1/${f//+/%2B}"
 done < ../sources/php8.1.sha256)
docker run --rm --platform linux/amd64 -v "$PWD:/repo:ro" -v "$PWD/sources-dl:/sources:ro" \
  -v "$PWD/out:/out" -e PHP=8.1 ubuntu:18.04 bash /repo/build.sh
```

### Signing key

The private key is the `APT_SIGNING_KEY` Actions secret and is not stored
anywhere else. If it is lost or must be rotated, generate a new ed25519 signing
key, replace the secret and `key.asc`, update the fingerprint in this README
and `install.sh`, and tell users to repeat step 1 of the quick start.

## Credits and license

- PHP packaging and security backports: Ondřej Surý ([deb.sury.org](https://deb.sury.org)).
  If this helps your business, consider
  [supporting his work](https://deb.sury.org/#donate) or
  [Freexian's PHP LTS](https://www.freexian.com/lts/php/), which offers
  commercially supported PHP packages.
- Build approach adapted from [Niek/php7](https://github.com/Niek/php7).
- This repository's scripts and documentation: [MIT](LICENSE). PHP and Sury's
  packaging keep their own licenses, included in each package.
