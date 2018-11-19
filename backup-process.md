# Backup and Restore procedure (First Iteration) - INCOMPLETE, DOES NOT WORK

Scenario: Full cluster backup and restore

# Backup:

## Velum:
Run two commands:

Ssh into the admin node.
```
docker exec $(docker ps -qf name="velum-mariadb") mysqldump --password=$(cat /var/lib/misc/infra-secrets/mariadb-root-password) velum_production | gzip > /var/lib/misc/infra-secrets/velum-backup.sql.gz
```

```
tar czf - -C / var/lib/misc etc/salt/pki etc/pki usr/share/caasp-container-manifests usr/share/salt/kubernetes > backup-CaaSP-admin-node.tgz
```

Logout from admin node and run (where $CaaSP-admin-node is the address of your admin node)
```
ssh -T $CaaSP-admin-node tar czf - -C / var/lib/misc etc/salt/pki etc/pki usr/share/caasp-container-manifests usr/share/salt/kubernetes > backup-CaaSP-admin-node.tgz
```
These two zip files can then be removed from the admin node and secured using your standard best practices.

Now backup-CaaSP-admin-node.tgz is on your host and you can store it whereever you like.

## etcd:
Further documentation on etcd backup can be found on [etcd recovery documentation](https://coreos.com/etcd/docs/latest/op-guide/recovery.html).

Ssh into a master node running etcd.

Work the notes from Rafael in here. Those may make this work on V3 as well.

Run:

```
set -a
source /etc/sysconfig/etcd
source /etc/sysconfig/etcdctl
set +a

etcdctl --endpoints $ETCDCTL_ENDPOINT snapshot save snapshot.db
```
Note `<$ETCDCTL_ENDPOINT>` is the address of the node you want to take the snapshot of. Any node should be fine. Alternativly `etcdctl snapshot save snapshot.db` can be used to capture a snapshot of the node you are currently on.

Copy the snapshopt to a safe location.

# Restore:

## Velum:
First copy the desired backup files onto the admin node and then run the following commands.

```
tar xzf - --exclude='mariadb-*-password' --exclude='usr/share/salt' --exclude='etc/salt/' --exclude='var/lib/misc/salt-master-credentials' --exclude='var/lib/misc/salt' --exclude='var/lib/misc/ssh-public-key' -C / < backup-CaaSP-admin-node.tgz 2>/dev/null

zcat velum-backup.sql.gz | docker exec -i $(docker ps -qf name="velum-mariadb") mysql --password=$(cat /var/lib/misc/infra-secrets/mariadb-root-password) velum_production

reboot
```

Wait for admin to fully boot and all docker containers to start. ssh back onto admin and run:

TODO: give a command to check if the admin node is ready to do the next step
```
docker exec -it $(docker ps | grep salt-master | awk '{print $1}') salt-run state.orchestrate orch.kubernetes
```
This is the faulty result ...
```
caasp-192-168-122-112:~ # docker exec -it $(docker ps | grep salt-master | awk '{print $1}') salt-run state.orchestrate orch.kubernetes
[CRITICAL] Specified ext_pillar interface velum is unavailable
[WARNING ] [CaaS]: etcd member count too low (0), increasing to 0
[ERROR   ] [CaaS]: get_additional_etcd_members: cannot satisfy the 1 members missing
[ERROR   ] Rendering exception occurred: Jinja variable No first item, sequence was empty.
[CRITICAL] Rendering SLS 'base:orch.kubernetes' failed: Jinja variable No first item, sequence was empty.
caasp-192-168-122-112_master:
    Data failed to compile:
    ----------
        Rendering SLS 'base:orch.kubernetes' failed: Jinja variable No first item, sequence was empty.
```

## etcd:

Bring up a new cluster as you would normally.

To find all the nodes you can ssh onto a master node running etcd and run:

```
set -a
source /etc/sysconfig/etcd
source /etc/sysconfig/etcdctl
set +a

etcdctl endpoint status --cluster
```

Find all instances of etcd and shut them down `systemctl stop etcd`
Copy backup file onto master
Ssh onto master

Run:

```
set -a
source /etc/sysconfig/etcd
source /etc/sysconfig/etcdctl
set +a
```

copy the snapshot.db backup file onto node
In the directory the snapshot file is run:
```
etcdctl snapshot restore snapshot.db --name $ETCD_NAME --initial-cluster $ETCD_NAME=https://<FQDN>:2380 --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls https://<FQDN>:2380
```

This will create a directory named `$ETCD_NAME.etcd` in your current working directory.

Inside this directory is a `member` directory, replace the `member` directory found in `$ETCD_DATA_DIR` with the directory created from the backup.

Update permissions `chown -R etcd:etcd /var/lib/etcd`

Start etcd `systemctl start etcd`

At this point you should be able to use kubectl and ask about your cluster (aka. `kubectl get pods`)

### Restore etcd cluster nodes

More infromation on cluster node addition can be found in [etcd runtime configuration documentation](https://coreos.com/etcd/docs/latest/op-guide/runtime-configuration.html).

Add nodes too your etcd cluster to a minimum of 3 nodes ssh to a worker node that you stopped etcd previously running on.

```
set -a
source /etc/sysconfig/etcd
source /etc/sysconfig/etcdctl
set +a
echo $ETCD_NAME
```
copy the etcd name

On the first etcd node that is now running:

```
ETCDCTL_API=2 etcdctl member add <NEW_ETCD_NAME> https://<FQDN>:2380
```

Copy the output that looks like this
```
ETCD_NAME="<NEW_ETCD_NAME>"
ETCD_INITIAL_CLUSTER="<ORGIN_ETCD_NAME>=https://master-0:2380,<NEW_ETCD_NAME>=https://<LOCAL-DOMAIN>:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"

```

Back on the new node run:
```
set -a
[paste the coppied lines]
ETCD_NAME="<NEW_ETCD_NAME>"
ETCD_INITIAL_CLUSTER="<ORGIN_ETCD_NAME>=https://master-0:2380,<NEW_ETCD_NAME>=https://<LOCAL-DOMAIN>:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
set +a

rm -R $ETCD_DATA_DIR/member

etcd
```

once you are up too 3 etcd nodes stop it and start it using `systemctl` on one node at a time:
```
chown -R etcd:etcd /var/lib/etcd
systemctl start etcd
```
Note, do this one at a time to maintain quorum.
