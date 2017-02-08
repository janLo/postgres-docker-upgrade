# Ansible Playbook to upgrade a dockerized postges

Upgrading a dockerized postgres is not as straight forward as a simple dist upgrade.
After some trying I found a way to do it without much hassle.
This contains an ansible playbook to do all the necessary steps.

To run the playbook you first have to set a few config variables.
First you need to set the host(s) you want to process.
This is done via the ansible inventory file.
There is a example `hosts` file that configures localhost as a target.
Just modify it or create a new one.
All operations are done for all servers in the `targets` group.

The basic config is done via environment.
The following variables must be set:
* ***PG_DATA_DIR*** The directory on the docker host where the postgres data lives (e.g. /mnt/container/postgres).
* ***PG_FROM*** The version of postgres you want to upgrade FROM (e.g. 9.4).
* ***TG_TO*** The Version of postgres you want to upgrade TO (e.g. 9.6)
* ***PG_PW*** Postgres password for the new cluster.

Then just run the ansible playbook:
```
$ PG_DATA_DIR=/mnt/container/postgres \
  PG_FROM=9.4 \
  PG_TO=9.6 \
  PG_PW=xxx \
  ansible-playbook -i hosts playbook.yml
```

If everything works as expected you should only get yellow and gree messages ;)

If anything goes wrong, there will be a backup of your data suffixd with `.backup`!

## Manual steps

These are the steps if you want to run the process manually:

te new data dir
* start new image to init cluster:
```
  # docker run --rm -ti -v <newdir>:/var/lib/postgresql/data -e POSTGRES_PASSWORD=xxx postgres:<new_version> postgres --single
```

* start old container to mount volumes from
```
  #  docker run --rm -ti --name old_pg -v /usr/lib/postgresql/9.4 -v /usr/share/postgresql/9.4/  postgres:<old_version> /bin/bash 
```
* Start new container to migrate
```
  # docker run --rm -ti -v <new_dir>:/tmp/new -v <old_dir>:/tmp/old --volumes-from old_pg -e postgres:<new_version> /bin/bash
```
* Run the upgrade
```
  # cd /tmp
  # su postgres -s /bin/bash
  # /usr/lib/postgresql/9.6/bin/pg_upgrade -b /usr/lib/postgresql/<old_version>/bin/ -B /usr/lib/postgresql/<new_version>/bin/  -d /tmp/old/  -D /tmp/new/
```
* Stop both containers
* Start the regular new container
* Run the cleanup
```
  # docker exec -t -i -u postgres <containername> /bin/bash
  # "/usr/lib/postgresql/9.6/bin/vacuumdb" --all --analyze-in-stages
```
