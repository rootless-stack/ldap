# rootless-stack / ldap

**Needs:** [base](https://github.com/rootless-stack/base).

[lldap](https://github.com/lldap/lldap) with PostgreSQL: the user
directory the other stacks authenticate against.

| Container | Image | Runs as |
| --- | --- | --- |
| `ldap-postgres` | `postgres:18-alpine` | 70, no capabilities |
| `ldap-lldap` | `lldap/lldap:v0.6` | 1000, no capabilities |
| `ldap-relay` | `haproxy:3.4-alpine` | the `ldap-svc` user itself (keep-id) |

All three share an internal network with no internet access and publish
no port.  The relay turns lldap's two TCP listeners into Unix sockets:

| Socket | Serves |
| --- | --- |
| `/run/rootless-ipc/lldap/ldap/ldap.sock` | LDAP (`ldapi://`) for consumers |
| `/run/rootless-ipc/lldap/http/http.sock` | web UI and GraphQL admin API |

Each socket's directory ACL decides which host users may connect; nobody
but `ldap-svc` (and root) can by default.

lldap keeps no state outside PostgreSQL: there is no config file and no
`/data` volume.  Its server key is derived from `secrets/lldap_key_seed`.

## Setup

Do the [base host setup](https://github.com/rootless-stack/base#host-setup) first.

**As root:**

```sh
git clone https://github.com/rootless-stack/ldap /opt/rootless-stack/ldap
useradd --create-home --home-dir /srv/ldap-svc --shell /usr/sbin/nologin ldap-svc
grep ldap-svc /etc/subuid /etc/subgid       # both must list a range
loginctl enable-linger ldap-svc
install -m 0644 /opt/rootless-stack/ldap/tmpfiles.d/rootless-ipc-ldap.conf /etc/tmpfiles.d/
systemd-tmpfiles --create
sudo -u ldap-svc -H -s
```

**As `ldap-svc`**, in the shell that opened:

```sh
export XDG_RUNTIME_DIR=/run/user/$(id -u)
cd ~ && umask 077
R=/opt/rootless-stack/ldap
mkdir -p config secrets data/postgres .config/containers/systemd .config/systemd/user
cp $R/quadlet/* .config/containers/systemd/
cp $R/systemd/ldap.service .config/systemd/user/
cp $R/config/haproxy.cfg config/
cp $R/config/env.example config/env          # then set LLDAP_LDAP_BASE_DN

# Secrets: one file each, no trailing newline.  The containers read them
# as other uids through a file mount, hence 0444 inside the 0700 dir.
pw=$(openssl rand -hex 32)
printf %s "$pw" > secrets/postgres_password
printf 'postgres://lldap:%s@postgres:5432/lldap' "$pw" > secrets/lldap_database_url
for s in lldap_jwt_secret lldap_key_seed lldap_admin_password; do
  printf %s "$(openssl rand -hex 32)" > secrets/$s
done
chmod 0444 secrets/*

# PostgreSQL runs as uid 70 inside the container.
podman unshare chown 70:70 data/postgres

systemctl --user daemon-reload
systemctl --user enable --now ldap.service
```

`start` returns once every container's start job has finished, which
waits for its healthcheck.  A container that fails does not fail the
anchor, so check them:

```sh
systemctl --user status 'ldap*'
curl -s --unix-socket /run/rootless-ipc/lldap/http/http.sock http://localhost/health
```

Log in as `admin` with the password in `secrets/lldap_admin_password`.
lldap stores it at the first start only: change it in the web UI later,
not in the file.

## Reaching the web UI

How the UI is exposed is your choice: a reverse proxy in front of
`http.sock` (with basic auth, an OAuth proxy, or nothing), or only an SSH
tunnel when needed:

```sh
ssh -L 17170:/run/rootless-ipc/lldap/http/http.sock root@host
# then open http://localhost:17170
```

## Granting access to a consumer

Each consumer stack ships its own tmpfiles file that appends (`a+`) its
grants to `/run/rootless-ipc/lldap` and the scope it uses; this stack's
file stays untouched.  Consumers mount the scope directory (not the
socket file, which is re-created at every relay start) and connect to
`ldapi://%2Frun%2Frootless-ipc%2Flldap%2Fldap%2Fldap.sock`.

Consumers bind with a read-only account: in the web UI, create a user
(for example `readonly`) and add it to the group
`lldap_strict_readonly`.

## Backup

Back up `data/postgres` (stop the stack, or dump with `pg_dump` through
`podman exec ldap-postgres`) **together with `secrets/`**.  Without
`lldap_key_seed`, no stored password can be verified.

## Updating

Copy the new units over the old ones, then:

```sh
systemctl --user daemon-reload
systemctl --user restart ldap.service
```
