LSST's ceph-ansible
===================

This procedure was used to deploy Ceph to back the storage (through the S3 API) for our git-lfs server.

A quick and dirty way to create a small Ceph cluster on NCSA's Nebula.

The expected life of this cluster is six months and no more than 10 TB.

Additional docs can be found at the Ceph website: http://docs.ceph.com/docs/v0.86/start/

# Prerequisites #

Launch four m1.mediums on Nebula that use a recommended [OS and Linux Kernel](http://docs.ceph.com/docs/v0.80.5/start/os-recommendations/). One will be used an an admin-node, git-lfs server. The other three will be part of the ceph cluster and host the S3 API through radosgw service. I recommend naming is `git-lfs-snowflake`, `ceph0-snowflake`, `ceph1-snowflake`, `ceph2-snowflake`. Add public IPs (Add Floating IPs) to `git-lfs-snowflake` and `ceph1-snowflake`.

Follow the [preflight instructions](http://docs.ceph.com/docs/v0.80.5/start/quick-start-preflight/) for all nodes.

Consider imaging or snapshotting the OS and Kernel at this point so you have pre-flight nodes with the appropriate OS and Kernel. Optional: be a good lsst-sqre citizen and install emacs24-nox. I used the snapshot `ubuntu-14.04-kernel-3.19`.


Install an AWS CLI and configure lsst-sqre AWS credentials that have access to r53. Use r53 to associate private and public IPs to machines with related names. `git-lfs-snowflake.lsst.codes`, `ceph0-snowflake.lsst.codes`, `ceph1-snowflake.lsst.codes`, `ceph2-snowflake.lsst.codes`. Associate `s3-snowflake.lsst.codes` to `ceph1`. Make sure that the server hostname is still ceph1. This machine will host the radosgw service.

# Configure Ansible #

Package install Ansible. Clone this repository.

```bash
git clone https://github.com/lsst-sqre/ceph-ansible.git
```

Configure Ansible's hosts file in the repo `files/admin/etc/ansible/hosts`.

```conf
[mons]
ceph0-snowflake.lsst.codes
ceph1-snowflake.lsst.codes
ceph2-snowflake.lsst.codes

[osds]
ceph0-snowflake.lsst.codes
ceph1-snowflake.lsst.codes
ceph2-snowflake.lsst.codes

[rgws]
ceph1-snowflake.lsst.codes
```

All osd nodes should have cinder volumes attached to them. By OpenStack convention these will be attached as `/dev/vdb`. To change this edit `group_vars/osds`.

```
devices:
  - /dev/vdb
```

Run this Ansible playbook.

```bash
ansible-playbook -i ./files/admin/etc/ansible/hosts ./site.yml
```

## Configure radosgw and apache2 ##

The Radosgw service provides the S3 and Swift APIs for Ceph. I proxy this through apache2. Note: fcgi and fcgi-proxy are required.

### rgw ###

In `/etc/ceph/ceph.conf` configure the rgw.

```conf
[client.rgw.ceph1-snowflake]
  host = ceph1-snowflake
  keyring = /var/lib/ceph/radosgw/ceph-rgw.ceph1-snowflake/keyring
  rgw socket path = ""  # /tmp/radosgw-ceph1-snowflake.sock
  rgw swift url = https://s3.lsst.codes
  log file = /var/log/radosgw/radosgw-ceph1-snowflake.log
  rgw data = /var/lib/ceph/radosgw/ceph-rgw.ceph1
  rgw frontends = fastcgi socket_port=9000 socket_host=0.0.0.0
  rgw print continue = false
```

Create a user and credentials.

```
radosgw-admin user create --uid="gitlfs" --display-name="Git LFS"
```

Note: It is possible for the credentials that are created to be bad. If this happens recreate them. Bad credentials will contain a `\/` (though there may be other bad character combinations) in their access-key or secret.

```bash
radosgw-admin key create --uid="gitlfs" --gen-access-key --gen-secret
```

These are the credentials you should supply to s3s3, git-lfs-s3-server.

### apache2 ###

On `ceph1-snowflake` install apache2 and the related fcgi and fcgi-proxy apache2 modules. Configure lsst-sqre's *.lsst.codes certificates, key, create a dhparam.

Copy and link `files/rgw/s3.lsst.codes.conf` to `ceph1-snowflake`'s apache2 sites-* directories. Update certficate, key and dhparam file locations.

Restart the rgw and apache2 services.

```bash
service radosgw restart id=rgw.ceph1-snowflake
service apache2 restart
```
