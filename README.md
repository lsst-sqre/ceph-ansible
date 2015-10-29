ceph-ansible
============

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

Original README.md
==================

## What does it do?

General support for:

* Monitors
* OSDs
* MDSs
* RGW

More details:

* Authentication (cephx), this can be disabled.
* Supports cluster public and private network.
* Monitors deployment. You can easily start with one monitor and then progressively add new nodes. So can deploy one monitor for testing purpose. For production, I recommend to always use an odd number of monitors, 3 tends to be the standard.
* Object Storage Daemons. Like the monitors you can start with a certain amount of nodes and then grow this number. The playbook either supports a dedicated device for storing the journal or both journal and OSD data on the same device (using a tiny partition at the beginning of the device).
* Metadata daemons.
* Collocation. The playbook supports collocating Monitors, OSDs and MDSs on the same machine.
* The playbook was validated on Debian Wheezy, Ubuntu 12.04 LTS and CentOS 6.4.
* Tested on Ceph Dumpling and Emperor.
* A rolling upgrade playbook was written, an upgrade from Dumpling to Emperor was performed and worked.


## Setup with Vagrant

Run your virtual machines:

```bash
$ vagrant up
...
...
...
 ____________
< PLAY RECAP >
 ------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||


mon0                       : ok=16   changed=11   unreachable=0    failed=0
mon1                       : ok=16   changed=10   unreachable=0    failed=0
mon2                       : ok=16   changed=11   unreachable=0    failed=0
osd0                       : ok=19   changed=7    unreachable=0    failed=0
osd1                       : ok=19   changed=7    unreachable=0    failed=0
osd2                       : ok=19   changed=7    unreachable=0    failed=0
rgw                        : ok=20   changed=17   unreachable=0    failed=0
```

Check the status:

```bash
$ vagrant ssh mon0 -c "sudo ceph -s"
    cluster 4a158d27-f750-41d5-9e7f-26ce4c9d2d45
     health HEALTH_OK
     monmap e3: 3 mons at {ceph-mon0=192.168.0.10:6789/0,ceph-mon1=192.168.0.11:6789/0,ceph-mon2=192.168.0.12:6789/0}, election epoch 6, quorum 0,1,2 ceph-mon0,ceph-mon1,ceph-mon
     mdsmap e6: 1/1/1 up {0=ceph-osd0=up:active}, 2 up:standby
     osdmap e10: 6 osds: 6 up, 6 in
      pgmap v17: 192 pgs, 3 pools, 9470 bytes data, 21 objects
            205 MB used, 29728 MB / 29933 MB avail
                 192 active+clean
```

To re-run the Ansible provisioning scripts:

```bash
$ vagrant provision
```

## Specifying fsid and secret key in production

The Vagrantfile specifies an fsid for the cluster and a secret key for the
monitor. If using these playbooks in production, you must generate your own `fsid`
in `group_vars/all` and `monitor_secret` in `group_vars/mons`. Those files contain
information about how to generate appropriate values for these variables.

## Specifying package origin

By default, ceph-common installs from Ceph repository. However, you
can set `ceph_origin` to "distro" to install Ceph from your default repository.


### For Debian based systems

If you want to use "backports", you can set "true" to `ceph_use_distro_backports`.
Attention, ceph-common doesn't manage backports repository, you must add it yourself.


## Vagrant Demo

[![Ceph-ansible Vagrant Demo](http://img.youtube.com/vi/E8-96NamLDo/0.jpg)](https://youtu.be/E8-96NamLDo "Deploy Ceph with Ansible (Vagrant demo)")


## Bare metal demo

Deployment from scratch on bare metal machines:

[![Ceph-ansible bare metal demo](http://img.youtube.com/vi/dv_PEp9qAqg/0.jpg)](https://youtu.be/dv_PEp9qAqg "Deploy Ceph with Ansible (Bare metal demo)")
