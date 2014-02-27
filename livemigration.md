# Enabling live migration without shared storage

---

## Configure Nova

Set the following in `/etc/nova/nova.conf` on your compute nodes:

    vncserver_listen = 0.0.0.0
    live_migration_flag = VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE

...and restart all nova services (`openstack-service restart nova`).

---

## Configure libvirt

Set the following in `/etc/libvirt/libvirtd.conf`:

    listen_tcp=1
    listen_tls=0
    dynamic_ownership=0
    auth_tcp="none"
    listen_addr="0.0.0.0"

---

## Enable libvirtd networking

In `/etc/sysconfig/libvirtd`, set:

    LIBVIRTD_ARGS="--listen"

...and restart `libvirtd`.

---

## Migrate an instance

You will only be able to migrate instances started *after* making
these changes.  Then:

    nova live-migration --block-migrate <instance> <compute host>

