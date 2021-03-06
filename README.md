# IMTEK Simulation MongoDB on SMB

[![Docker Image Version (latest semver)](https://img.shields.io/docker/v/imteksim/mongod-on-smb?label=dockerhub&sort=semver)](https://hub.docker.com/repository/docker/imteksim/mongod-on-smb) [![GitHub Workflow Status](https://img.shields.io/github/workflow/status/IMTEK-Simulation/mongod-on-smb-container-image/publish)](https://github.com/IMTEK-Simulation/mongod-on-smb-container-image/actions?query=workflow%3Apublish)

Copyright 2020, 2021 IMTEK Simulation, University of Freiburg

Author: Johannes Hoermann, johannes.hoermann@imtek.uni-freiburg.de

## Summary

Mount an smb share holding raw db within mongo conatiner and publish
standard port 27017 via TLS/SSL encryption globally.

## Usage

Launch mongod service with

```shell
docker run --privileged --cap-add SYS_ADMIN imteksim/mongod-on-smb:latest
```

See https://hub.docker.com/_/mongo and https://github.com/docker-library/mongo for upstream details.

## Envionment variables

* `SMB_HOST` - name of host providing smb share, default: `sambaserver`
* `SMB_SHARE` - name of share, default: `sambashare`
* `SMB_MOUNT_OPTIONS` - CIFS mount options for smb share, default: `rw,iocharset=utf8,vers=1.0,cache=none,credentials=/run/secrets/smb-credentials,file_mode=0600,dir_mode=0700`

## Secrets

`/run/secrets/tls_key_cert.pem` - key and certificate for mongod
`/run/secrets/rootCA.pem` - root CA
`/run/secrets/smb-credentials`- credentials file for smb share, see i.e. mount.cifs(8), https://www.samba.org/~ab/output/htmldocs/manpages-3/mount.cifs.8.html

## Setup with Podman

Podman runs without elevated privileges. The `cifs` driver for smb shares requires
elevated privileges for mount operations. Thus, it must be replaced
by a pure userland approach. The described setup is based on the FUSE
drivers `smbnetfs` and `bindfs`. See `compose/local/mongodb/docker-entrypoint.sh` 
for more information.

### Capabilities

Granted capabilities are prefixed by `CAP_`, i.e.

    cap_add:
      - CAP_SYS_ADMIN

for Podman compared to

    cap_add:
      - SYS_ADMIN

for Docker within the `compose.yml` file. This capability in connection with

    devices:
      - /dev/fuse

is necessary for enabling the use of FUSE file system drivers within the unprivileged
container.

### Secrets

podman does not handle `secrets` the way docker does. Similar behavior can be achieved with
a per-user configuration file `$HOME/.config/containers/mounts.conf` on the host containing, 
for example, a line

    /home/user/containers/secrets:/run/secrets

that will make the content of `/home/user/containers/secrets` on the host available under
`/run/secrets` within *all containers* of the evoking user. The owner and group within 
the container will be `root:root` and file permissions will correspond to permissions 
on the host file system. Thus, an entrypoint script might have to adapt permissions.

### Debugging

Look at the database at `https://localhost:8081` or try to connect to the database
from within the mongo container with

    mongo --tls --tlsCAFile /run/secrets/rootCA.pem --tlsCertificateKeyFile \
        /run/secrets/tls_key_cert.pem --host mongodb

or from the host system

     mongo --tls --tlsCAFile keys/rootCA.pem \
        --tlsCertificateKeyFile keys/mongodb.pem --sslAllowInvalidHostnames

if the FQDN in the server's certificate has been set to the service's name 
'mongodb'.

### Wipe database

Enter a running `mongodb` container instance, i.e. with

    podman exec -it mongodb bash

find `mongod`'s pid, i.e. with 

```console
$
...
mongodb     41  0.3  1.4 1580536 112584 ?      SLl  13:06   0:06 mongod --config /etc/mongod.conf --auth --bind_ip_all
...
```
end it, i.e. with `kill 41`, to release all database files, and purge the database directory with

    rm -rf /data/db/*


## References

- Certificates:
  - https://medium.com/@rajanmaharjan/secure-your-mongodb-connections-ssl-tls-92e2addb3c89
- Docker setup
  - Mounting samba share in docker container:
    - https://github.com/moby/moby/issues/22197
    - https://stackoverflow.com/questions/27989751/mount-smb-cifs-share-within-a-docker-container
  - Sensitive data:
    - https://docs.docker.com/compose/compose-file/#secrets
    - https://docs.docker.com/compose/compose-file/#secrets-configuration-reference
  - MongoDB, mongo-express & docker:
    - https://hub.docker.com/_/mongo
    - https://docs.mongodb.com/manual/administration/security-checklist/
    - https://docs.mongodb.com/manual/tutorial/configure-ssl
    - https://hub.docker.com/_/mongo-express
    - https://github.com/mongo-express/mongo-express
    - https://github.com/mongo-express/mongo-express/blob/e4777b6f8bce62d204e9c4204801e2cb7a7b8898/config.default.js#L41
    - https://github.com/mongo-express/mongo-express-docker
    - https://github.com/mongo-express/mongo-express/pull/574
- Podman setup
  - Sensitive data
    - https://www.projectatomic.io/blog/2018/06/sneak-secrets-into-containers/
  - FUSE-related
    - https://bindfs.org/
    - https://bindfs.org/docs/bindfs-help.txt
    - https://rhodesmill.org/brandon/2010/mounting-windows-shares-in-linux-userspace/
