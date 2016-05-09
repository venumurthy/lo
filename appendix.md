
To deploy Cinder

```bash
 mysql -u root -p
```

```sql
 CREATE DATABASE cinder;
```

```sql
 GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'password';
```
```sql
 GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'password';
```
```sql
 exit;
```

```bash
 source admin.opensrc.sh
```

```bash
 openstack user create --domain default --password-prompt cinder
```

```bash
 openstack role add --project service --user cinder admin
```

```bash
 openstack service create --name cinder --description "OpenStack Block Storage" volume
```

```bash
 openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
```

```bash
 openstack endpoint create --region RegionOne volume public http://control:8776/v1/%\(tenant_id\)s
```

```bash
 openstack endpoint create --region RegionOne volume internal http://control:8776/v1/%\(tenant_id\)s
```

```bash
 openstack endpoint create --region RegionOne volume admin http://control:8776/v1/%\(tenant_id\)s
```

```bash
openstack endpoint create --region RegionOne volumev2 public http://control:8776/v2/%\(tenant_id\)s
```


```bash
openstack endpoint create --region RegionOne volumev2 internal http://control:8776/v2/%\(tenant_id\)s
```

```bash
openstack endpoint create --region RegionOne volumev2 admin http://control:8776/v2/%\(tenant_id\)s
```

```bash
  apt-get install cinder-api cinder-scheduler
```

```bash
  cat /etc/cinder/cinder.conf | grep "^[^#$]" > cinder.conf.bkp
```

    [DEFAULT]
    
    my_ip = 10.10.10.2
    auth_strategy = keystone
    rpc_backend = rabbit
    ...
    
    [database]
    
    connection = mysql+pymysql://cinder:password@control/cinder
    ...
    
    [keystone_authtoken]
    
    auth_uri = http://control:5000
    auth_url = http://control:35357
    memcached_servers = control:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = cinder
    password = password
    ...
    
    [oslo_messaging_rabbit]
    
    rabbit_host = control
    rabbit_userid = openstack
    rabbit_password = password
    ...
    
    [oslo_concurrency]

    lock_path = /var/lib/cinder/tmp
    ...
    

```bash
  su -s /bin/sh -c "cinder-manage db sync" cinder
```

```bash
  vi /etc/nova/nova.conf 
```

    [cinder]
    os_region_name = RegionOne


```bash
 service nova-api restart
```

```bash
for service in scheduler api; do
service cinder-$service restart
done 
```

On Storage node

```bash
 apt-get install lvm2
```

```bash
 pvcreate /dev/sdb
```

```bash
 gcreate cinder-volumes /dev/sdb
```

```bash
 vgs -vvvv
```

```bash
 apt-get install cinder-volume
```

```bash
 cat /etc/cinder/cinder.conf | grep "^[^#$]" > cinder.conf.bkp
```

```bash
 vi /etc/cinder/cinder.conf
```

    [DEFAULT]
    
    enabled_backends = lvm
    glance_api_servers = http://control:9292
    ...
    
    [lvm]
    
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes
    iscsi_protocol = iscsi
    iscsi_helper = tgtadm
    ...
    
    
    [oslo_concurrency]
 
    lock_path = /var/lib/cinder/tmp
    ...

```bash
 service tgt restart
 service cinder-volume restart
```

```bash
 source admin.opensrc.sh
```

```bash
 cinder service-list
```

To deploy Swift


```bash
 source admin.opensrc.sh
```


```bash
 openstack user create --domain default --password-prompt swift
```

```bash
 openstack role add --project service --user swift admin
```


```bash
 openstack service create --name swift --description "Object Storage" object-store
```


```bash
 openstack endpoint create --region RegionOne object-store public http://control:8080/v1/AUTH_%\(tenant_id\)s
```


```bash
 openstack endpoint create --region RegionOne object-store internal http://control:8080/v1/AUTH_%\(tenant_id\)s
```


```bash
 openstack endpoint create --region RegionOne object-store admin http://control:8080/v1
```


```bash
 apt-get install swift swift-proxy python-swiftclient python-keystoneclient python-keystonemiddleware memcached
```


```bash
 mkdir -p /etc/swift
```

```bash
curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/mitaka
```

    [DEFAULT]
    
    bind_port = 8080
    user = swift
    swift_dir = /etc/swift
    ...

    [pipeline:main]
    
    pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server
    ...
    
    [app:proxy-server] 
    
    use = egg:swift#proxy
    ...
    account_autocreate = True
    
    [filter:keystoneauth]
    
    use = egg:swift#keystoneauth
    ...
    operator_roles = admin,user
    
    
    [filter:authtoken]
    
    paste.filter_factory = keystonemiddleware.auth_token:filter_factory
    ...
    auth_uri = http://control:5000
    auth_url = http://control:35357
    memcached_servers = control:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = swift
    password = SWIFT_PASS
    delay_auth_decision = True
    
 ```bash
  apt-get install xfsprogs rsync
 ```

```bash
 fdisk /dev/sdc
```

```bash
 mkfs.xfs /dev/sdc1
```

```bash
 vi /etc/fstab
```

    /dev/sdc1 /mnt/sdc1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 0
    
    ```bash
    mkdir /mnt/sdc1
    mount /mnt/sdc1
    mkdir /mnt/sdc1/1 /mnt/sdc1/2 /mnt/sdc1/3 /mnt/sdc1/4
    chown ${USER}:${USER} /mnt/sdc1/*
    sudo mkdir /srv
    for x in {1..4}; do sudo ln -s /mnt/sdc1/$x /srv/$x; done
    sudo mkdir -p /srv/1/node/sdc1 /srv/1/node/sdc5 \
                  /srv/2/node/sdc2 /srv/2/node/sdc6 \
                  /srv/3/node/sdc3 /srv/3/node/sdc7 \
                  /srv/4/node/sdc4 /srv/4/node/sdc8 \
                  /var/run/swift
    sudo chown -R ${USER}:${USER} /var/run/swift
    
    
    
    for x in {1..4}; do sudo chown -R ${USER}:${USER} /srv/$x/; done
    ```




```bash
 mount /srv/node/sdc
 mount /srv/node/sdd
```

```bash
 vi /etc/rsyncd.conf
```

    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /var/run/rsyncd.pid
    address = 10.10.10.2
    
    [account]
    max connections = 2
    path = /srv/node/
    read only = False
    lock file = /var/lock/account.lock
    
    [container]
    max connections = 2
    path = /srv/node/
    read only = False
    lock file = /var/lock/container.lock
    
    [object]
    max connections = 2
    path = /srv/node/
    read only = False
    lock file = /var/lock/object.lock

```bash
 vi /etc/default/rsync
```

    RSYNC_ENABLE=true

```bash
  service rsync start
```

```bash
 apt-get install swift swift-account swift-container swift-object
```

```bash
curl -o /etc/swift/account-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/mitaka
# curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/mitaka
# curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/mitaka
```

```bash
 vi /etc/swift/account-server.conf
```

    [DEFAULT]
    
    bind_ip = 10.10.10.2
    bind_port = 6002
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node
    mount_check = True
    ...
    
    [pipeline:main]
    
    pipeline = healthcheck recon account-server
    ...
    
    [filter:recon]
    
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift
    ...
    

```bash
 vi /etc/swift/container-server.conf 
```

    [DEFAULT]
    
    bind_ip = 10.10.10.2
    bind_port = 6001
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node
    mount_check = True
    ...
    
    [pipeline:main]
    
    pipeline = healthcheck recon container-server
    ...
    
    [filter:recon]
    
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift
    ...
    
    
```bash
 vi /etc/swift/object-server.conf
```
    
    [DEFAULT]

    bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
    bind_port = 6000
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node
    mount_check = True
    ...
    
    [pipeline:main]
    pipeline = healthcheck recon object-server
    ...
    
    [filter:recon]
    
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift
    recon_lock_path = /var/lock
    ...
    
```bash
 chown -R swift:swift /srv/node
```

```bash
# chown -R root:swift /var/cache/swift
# mkdir -p /var/cache/swift
# chmod -R 775 /var/cache/swift
```

    
    
    
    
    
    
    
    
    
    
    
    
