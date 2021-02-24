## User for software installations

Create software installation user

```shell
sudo useradd --user-group --system --base-dir /var/lib --create-home --comment 'software installations' sw-installer
```

Note: If the user already exists, then expect to see "useradd: user 'sw-installer' already exists". Review the output of the next command carefully.

Check passwd entry
```shell
getent passwd sw-installer
```

Fix .forward file
```shell
sudo -u sw-installer bash -l -c 'umask 077 ; echo root > ~/.forward'
```

Check that home directory has skeleton files
```shell
sudo ls -alF ~sw-installer/
```


## Shared miniconda upgrader script

Create upgrader script

```shell
sudo -e /admin/scripts/miniconda_shared_upgrader
```

```shell
#!/bin/bash
set -euo pipefail
mc=/usr/local/miniconda
conda=$mc/bin/conda
if [ -x "$conda" ]
then
  "$conda" update --yes --quiet conda
  "$conda" clean  --yes --quiet --all
  chmod -R go-w,a+rX -- "$mc"
  find "$mc"/pkgs -mindepth 1 -maxdepth 1 -type d -execdir rmdir --ignore-fail-on-non-empty -- '{}' +
fi
```

Check contents

```shell
cat /admin/scripts/miniconda_shared_upgrader
```

Fix permissions

```shell
sudo chmod -c a+rx /admin/scripts/miniconda_shared_upgrader
```

## Miniconda installation

Create the installation directory

```shell
sudo install -v -d -o sw-installer -g sw-installer -m 0755 /usr/local/miniconda
```

Note: If the directory already exists, then there will be output only if there is an error.

Check the directory (should be empty)

```shell
ll -a /usr/local/miniconda/
```

TODO: what to do about files already in the installation directory?

Become software installation user

```shell
sudo -u sw-installer -i
```

Change to safe location and download miniconda installer

```shell
cd $(mktemp -d)
curl -LRO https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
```

Install miniconda

```shell
bash ./Miniconda3-latest-Linux-x86_64.sh -b -f -p /usr/local/miniconda && echo SUCCESS
```

Run upgrader for the first time to get latest packages and to fix permissions

```shell
/admin/scripts/miniconda_shared_upgrader && echo SUCCESS
```

Check miniconda installation

```shell
/usr/local/miniconda/bin/conda info
/usr/local/miniconda/bin/conda list
```

Stop being software installation user

```shell
exit
```

Check permissions

```shell
find /usr/local/miniconda/ -printf '%M %u %g\n' | sort | uniq -c
```

Note: Everything should be owned by the software installation user but readable by everyone, and nothing should be world-writable (ignoring symlink permissions).

Add cron.d file for miniconda upgrader

```shell
sudo -e /etc/cron.d/miniconda-upgrader
```

```
# Upgrade packages in shared miniconda installation
23 06 * * Mon  sw-installer  /admin/scripts/miniconda_shared_upgrader >/dev/null
```

Check cron.d file

```shell
ls -ld /etc/cron.d/miniconda-upgrader
cat -n /etc/cron.d/miniconda-upgrader
```
