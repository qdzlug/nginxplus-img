# Debian and Ubuntu NGINX Plus Images

## Image Description

These two images are both built to support an NGINX Plus deployment. Two versions have been provided:

- Debian 11 Bullseye
- Ubuntu 21.04 Focal

Additional details on the build process for the images are included below.

## Deployment Considerations

This is a very opinionated image that has been designed for a very specific use case. As such, there are a number of
configuration decisions that would not normally be made in a standard VM image. Specifically, this image has been built
specifically to be deployed in an OpenStack environment that does not provide correct metadata for interface
configuration. As this is not a normal configuration, several changes have needed to be made to the image in order to
prevent the network from being autoconfigured either by the standard boot process or by cloud-init.

### Nonstandard Configuration Items

- All networking configuration must be done via cloud-init; additional detail on how this has been accomplished in the
  image build is noted below.
- Root SSH keypair is present in the image and has been added to the `authorized_keys` file for the root user restricted
  to ip addresses 169.254.0.1 and 169.254.0.2. This has been done for the `keepalived` configuration.

### Deployment Requirements

The following configuration items will need to passed through to the image via the cloud-init process:

- Networking configuration.
- User accounts.
- Data cache management for cloud-init.
- Hostname and hostname management.
- NGINX Sync configuration.
- `keepalived` configuration.
- Service management.

### Networking

Since this image has been configured to not automatically plumb network interfaces the networking must be set as part of
the deployment via a cloud-init file. This includes adding the DNS resolver information.

These need to be set inside a write-files directive.

```
write_files:
  - path: /etc/network/interfaces.d/enp1s0
    permissions: '0644'
    defer: true
    content: |
      auto enp1s0
      iface enp1s0 inet static
      address 192.168.216.216
      netmask 255.255.255.0
      gateway 192.168.216.1
      dns-nameservers 192.168.216.1 192.168.213.2
```

#### Notes

- The interfaces for both the Debian and Ubuntu distributions are in the format of `enpXs0` where `X` is the interface
  number starting at `1`.
- If multiple interfaces are defined they should be put into files named with the name of the interface and placed in
  the `/etc/network/interfaces.d` directory.
- If multiple interfaces are defined, only one default gateway should be defined.

### User Accounts

By default, both Debian and Ubuntu lock user accounts from logging in from the console, although they are allowed to log
in via ssh. To address this they need to both have their password set and the their account unlocked. This can be
accomplished by following this example; note the `lock-passwd` and `passwd` directives.

```
users:
  - name: nplus
    groups: sudo
    shell: /bin/bash
    lock-passwd: false
    passwd: $6$jq4P5K1S2PAYxVt8$/jPIMbBf5jje6BLGoCRO9gZsIYGkYCEUeFzUQ2Hch4fdtlNCGbg0zTdYk/.8dZj/DqixpKUm5K9wEJtOOVdKi.
    sudo:
      - ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys: >-
      ssh-rsa
      AAAAB3NzaC1yc2EAAAADAQABAAABAQDh6ROxdnUrSAmjyqzlpvcSFlXcSwD7VMp7PvCTzAtDePSluBiQq3njWW88Pcxgmhsqhsm/ZjRKTdFO5RWRt2YM3BsZQqIMlsulIKK426RavgtnMYpJuUhTkyVm1QQAaoOH4NvkBOk35VOWylzxSZFa2v+LExjOQzQM5CfXB2GX7KerNNvEMNuTnFQ5upuV8YOEeeeomfLmt/I8VMxFJiSQWlELkS2NBVbhWKHcRaE2T2X2eASaruqlDhSMgeE0K/8bRuLquvv5j0F3rQ6slbVi0zjdIMRUlwD4gsZOQaSiFrQceItR+slp3/2FT/o6uxW/lJu3sW5RkHNHMxubSFpl
      nplus@nginx
```

#### Notes

- The password supplied needs to be hashed; you can accomplish this by running the following command on a linux system
  and then cutting/pasting the resultant hash into the configuration. A plaintext password **will not work**.
  Additionally, this password should be changed by the user once they have logged into the instance.
  `mkpasswd --method=SHA-512 --rounds=4096`
- If desired, the `chpasswd` cloud-init construct can be used to change passwords (for example, the root password) for
  console login. As this requires a plaintext password it is recommended that this password be changed by the user once
  they have logged into the provisioned instance. Example provided below.

   ```
   chpasswd:
   list: |
    root:password
   expire: false
```

### Data Cache and Machine ID

By default, the cloud-init process checks the cloud-init cache and machine ID on each boot, and will re-run if the
machine ID is different. This means that if a VM is migrated the cloud-init process will re-run when it is started on a
new host machine. Since this is normally not the desired behavior the following stanza should be set in the cloud-init
file:

```
manual_cache_clean: true
```

If this is set and later it is decided that you wish to have cloud-init re-run, this can be accomplished by deleting
the `/var/lib/cloud` directory.

### Hostname Management

Although it is possible for an image to receive it's hostname from the hypervisor, in the case where we want to manage
the hostname ourselves we can accomplish this by setting two configuration lines in the cloud-init file as shown bwloe.
The `manage_etc_hosts` directive instructs cloud-init to update the hostfile appropriately for the hostname.

```
manage_etc_hosts: true
hostname: nginix-test
```

### Service Management

To ensure that the networking and resolver configuration are applied correctly, the last step that the cloud-init
process should initate is a restart of these services:

```
runcmd:
  - - systemctl
    - restart
    - networking
  - - systemctl
    - enable
    - '--now'
    - resolvconf
```

Additionally, if you are deploying in an HA configuration you will want ensure that you have started the `keepalived`
service by adding the following into the `runcmd` stanza:

```
	- - systemctl
		- enable
		- '--now'
		- keepalived
```

#### Notes

- With `runcmd` Care should be taken with any characters that are significant in yaml - for example, colons. In this
  case, they should be quoted or quoted and passed in an array.

### Root SSH Keys

Keys for root are automatically generated at image creation time, and they are added to the `authorized_keys` file for
the root user.

These are intended for use for HA setup, and to further this they have been configured for use only between the agreed
upon IP addresses used for this configuration:

- 169.254.0.1
- 169.254.0.2

#### Notes

It is recommended that the end user replace these with their own keys.

### Configuration for `keepalived`

The following template file is installed in the image as `/etc/keepalived/keepalived.conf.template`

```
     global_defs {
          vrrp_version 3
      }
      vrrp_script chk_manual_failover {
          script   /usr/lib/keepalived/nginx-ha-manual-failover
          interval 10
          weight   50
      }
      vrrp_script chk_nginx_service {
          script   /usr/lib/keepalived/nginx-ha-check
          interval 3
          weight   50
      }
      vrrp_instance VI_1 {
          state                      VI_1_STATE
          interface                VI_1_INTERFACE
          nopreempt
          priority                   125
          virtual_router_id          52
          use_vmac
          vmac_xmit_base
          advert_int                 1
          accept
          garp_master_refresh        5
          garp_master_refresh_repeat 1
          unicast_src_ip             VI_1_UNICAST_SRC_IP

          unicast_peer {
             VI_1_UNICAST_PEER
          }

          virtual_ipaddress {
              VI_1_VIRTUAL_IP_ADDRESS
          }

          track_script {
              chk_nginx_service
              chk_manual_failover
          }

          notify /usr/lib/keepalived/nginx-ha-notify
      }


      vrrp_instance VI_2 {
          state                    VI_2_STATE
          interface                VI_2_INTERFACE
          nopreempt
          priority                   125
          virtual_router_id          52
          advert_int                 1
          accept
          garp_master_refresh        5
          garp_master_refresh_repeat 1
          unicast_src_ip             VI_2_UNICAST_SRC_IP

          unicast_peer {
             VI_2_UNICAST_PEER
          }

          virtual_ipaddress {
              VI_2_VIRTUAL_IPADDRESS
          }

          track_script {
              chk_nginx_service
              chk_manual_failover
          }

          notify /usr/lib/keepalived/nginx-ha-notify
      }
```

#### Value List

| Variable | Replace With | Valid Values |
|----------|--------------|-------|
|VI_1_STATE|State of the VI1 Instance|PRIMARY or BACKUP|
|VI_1_INTERFACE|Interface for VI1 Instance|Network Device|
|VI_1_UNICAST_SRC_IP|Source IP for VI1|IP Address|
|VI_1_UNICAST_PEER|Unicast Peer for VI1|IP Address|
|VI_1_VIRTUAL_IP_ADDRESS|Floating IP for VI1||
|VI_2_STATE |State of the VI2 Instance|PRIMARY or BACKUP|
|VI_2_INTERFACE|Interface for the VI2 Instance|Network Device|
|VI_2_UNICAST_SRC_IP|Source IP for the VI2|IP Address|
|VI_2_UNICAST_PEER|Unicast Peer for VI2|IP Address|
|VI_2_VIRTUAL_IPADDRESS|Floating IP for VI2||

#### Notes

- Placeholder variables such as VI_1_UNICAST_PEER will need to be updated; this can be done at provision time or later
  if desired.
- This file will need to be renamed to `keepalived.conf` before use.
- This file will need to be configured for each member of the HA pair separately; an identical configuration on both
  nodes will cause issues.

### Configuration for NGINX Sync

Similar to the `keepalived` file, this file has been included in the image in template form and is installed
as `/etc/nginx-sync.conf.template`.

```
NODES="NGINX_SYNC_NODES"
CONFPATHS="/etc/nginx/nginx.conf /etc/nginx/conf.d"
EXCLUDE="default.conf"
```

#### Value List

| Variable | Replace With | Valid Values |
|----------|--------------|-------|
|NGINX_SYNC_NODES|IP Address of Peer Nodes|Space delimited list of IP addresses in quotes|

#### Notes

- Placeholder variables such as NGINX_SYNC_NODES will need to be updated with the actual values. This can be done at
  provision time or later if desired.
- This file will need to be renamed to `nginx-sync.conf` before use.

## Build Process

This image is built from the [nginxplus-img](https://github.com/qdzlug/nginxplus-img) github repository using the
Jenkins CI system and the OpenStack [diskimage-builder](https://docs.openstack.org/diskimage-builder/latest/) program.

There are three main subdirectories:

- `debian`: Debian image build files.
- `ubuntu`: Ubuntu image build files.
- `centos`: Centos image build files (deprecated).

Each subdirectory contains a `Jenkinsfile` with the pipeline logic for the build.

### Image Changes

- The creation of `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` with the contents:
  ```
  network:
config: disabled
disable_network_activation: true
```
- Setting the `CONFIGURE_INTERFACES` variable to `NO` in `/etc/default/networking`.
- Setting the `DIB_DHCP_NETWORK_MANAGER_AUTO` variable to `true` for the build process.

## Image Details

The images are currently being built out of the [nginxplus-img](https://github.com/qdzlug/nginxplus-img) repository.
Note that there are certain values that are being supplied at build time; these are documented in the Jenkinsfiles.

### Ubuntu

The following configuration options were used for the Ubuntu build.

#### Cloud-Init Datasources

- NoCloud
- ConfigDrive
- OpenStack

#### Image Options

- ubuntu-minimal
- base
- cloud-init-datasources
- dhcp-all-interfaces
- enable-serial-console
- growroot
- openssh-server
- package-installs
- pkg-map
- vm
- journal-to-console

#### Packages

- curl
- dnsutils
- man
- netcat
- ntp
- ping
- resolvconf
- vim
- cloud-init
- ipcalc
- sshpass
- syslog
- tcpdump
- ethtool

### Debian

The following configuration options were used for the Debian build.

#### Cloud-Init Datasources

- NoCloud
- ConfigDrive
- OpenStack

#### Image Options

- debian
- base
- cloud-init-datasources
- dhcp-all-interfaces
- enable-serial-console
- growroot
- openssh-server
- package-installs
- pkg-map
- vm
- journal-to-console

#### Packages

- curl
- dnsutils
- man
- netcat
- ntp
- ping
- resolvconf
- vim
- cloud-init
- ipcalc
- sshpass
- syslog
- tcpdump
- ethtool

## Sample `cloud-init.yaml` Files

### Standalone

```
#cloud-config
#
# This is a standalone configuration for one NGINX Plus instance.
#
# Note that this is for example purposes only; users will need to add all information
# that is enclosed in angle brackets <>
#
# The images built by this pipeline will have NICs in the format of enpXs0, where X is 
# the number of the NIC. NIC numbering starts at 1.
#
manual_cache_clean: true
manage_etc_hosts: true
hostname: nplus-standalone
users:
  - name: nplus
    groups: sudo
    shell: /bin/bash
    lock-passwd: false
    passwd: <HASHED-PASSWORD>
    sudo:
      - ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys: >-
      <YOURKEY>
write_files:
  - path: /etc/network/interfaces.d/<NIC1>
    permissions: '0644'
    defer: true
    content: |
      auto <NIC1>
      iface <NIC1> inet static
      address <NIC1_IP_ADDRESS>
      netmask <NIC1_NETMASK>
      gateway <NIC1_GATEWAY>
      dns-nameservers <NAMESERVERS SPACE DELIMITED>
runcmd:
  - - systemctl
    - restart
    - networking
  - - systemctl
    - enable
    - '--now'
    - resolvconf
ssh_authorized_keys:
  - >
    YOURKEY

```

### HA Primary

```
#cloud-config
#
# This is a primary HA configuration for one NGINX Plus instance.
#
# Note that this is for example purposes only; users will need to add all information
# that is enclosed in angle brackets <>
#
# The images built by this pipeline will have NICs in the format of enpXs0, where X is
# the number of the NIC. NIC numbering starts at 1.
#
manual_cache_clean: true
manage_etc_hosts: true
hostname: nginix-primary
  users:
      - name: nplus
        groups: sudo
        shell: /bin/bash
        lock-passwd: false
        passwd: <HASHED-PASSWORD>
        sudo:
          - ALL=(ALL) NOPASSWD:ALL
        ssh-authorized-keys: >-
          <YOURKEY>
  write_files:
    - path: /etc/network/interfaces.d/<NIC1>
      permissions: '0644'
      defer: true
      content: |
        auto <NIC1>
        iface <NIC1> inet static
        address <NIC1_IP_ADDRESS>
        netmask <NIC1_NETMASK>
        gateway <NIC1_GATEWAY>
        dns-nameservers <NAMESERVERS SPACE DELIMITED>
    - path: /etc/network/interfaces.d/<NIC2>
      permissions: '0644'
      defer: true
      content: |
        auto <NIC2>
        iface <NIC2> inet static
        address <NIC2>
        netmask <NIC2>
  - path: /usr/local/bin/build-configs
    permissions: '0755'
    content: |
      #!/bin/bash
      cp /etc/keepalived/keepalived.conf.template /etc/keepalived/keepalived.conf.template.hold
      sed -i 's/VI_1_STATE/PRIMARY/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_INTERFACE/<NIC2>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_UNICAST_SRC_IP/<NIC2_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_UNICAST_PEER/<HA_VI1_SECONDARY_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_VIRTUAL_IP_ADDRESS/<HA_VI1_VIP>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_STATE/PRIMARY/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_INTERFACE/<NIC1>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_UNICAST_SRC_IP/<NIC1_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_UNICAST_PEER/<HA_VI2_SECONDARY_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_VIRTUAL_IPADDRESS/<HA_VI2_VIP>/g' /etc/keepalived/keepalived.conf.template
      cp /etc/keepalived/keepalived.conf.template /etc/keepalived/keepalived.conf
      sed -i 's/NGINX_SYNC_NODES/<HA_VI1_SECONDARY_IP_ADDRESS/g' /etc/nginx-sync.conf.template
      cp /etc/nginx-sync.conf.template /etc/nginx-sync.conf

runcmd:
  - - '/bin/bash'
    - '-v'
    - '/usr/local/bin/build-configs'
  - - systemctl
    - restart
    - networking
  - - systemctl
    - enable
    - '--now'
    - resolvconf
  - - systemctl
    - enable
    - '--now'
    - keepalived
ssh_authorized_keys:
  - >
    <YOURKEY>

```

### HA Secondary

```
#cloud-config
#
# This is a secondary HA configuration for one NGINX Plus instance.
#
# Note that this is for example purposes only; users will need to add all information
# that is enclosed in angle brackets <>
#
# The images built by this pipeline will have NICs in the format of enpXs0, where X is
# the number of the NIC. NIC numbering starts at 1.
#
manual_cache_clean: true
manage_etc_hosts: true
hostname: nginix-secondary
  users:
      - name: nplus
        groups: sudo
        shell: /bin/bash
        lock-passwd: false
        passwd: <HASHED-PASSWORD>
        sudo:
          - ALL=(ALL) NOPASSWD:ALL
        ssh-authorized-keys: >-
          <YOURKEY>
  write_files:
    - path: /etc/network/interfaces.d/<NIC1>
      permissions: '0644'
      defer: true
      content: |
        auto <NIC1>
        iface <NIC1> inet static
        address <NIC1_IP_ADDRESS>
        netmask <NIC1_NETMASK>
        gateway <NIC1_GATEWAY>
        dns-nameservers <NAMESERVERS SPACE DELIMITED>
    - path: /etc/network/interfaces.d/<NIC2>
      permissions: '0644'
      defer: true
      content: |
        auto <NIC2>
        iface <NIC2> inet static
        address <NIC2>
        netmask <NIC2>
  - path: /usr/local/bin/build-configs
    permissions: '0755'
    content: |
      #!/bin/bash
      sed -i 's/VI_1_STATE/PRIMARY/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_INTERFACE/<NIC2>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_UNICAST_SRC_IP/<NIC2_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_UNICAST_PEER/<HA_VI1_PRIMARY_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_1_VIRTUAL_IP_ADDRESS/<HA_VI1_VIP>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_STATE/PRIMARY/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_INTERFACE/<NIC1>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_UNICAST_SRC_IP/<NIC1_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_UNICAST_PEER/<HA_VI2_PRIMARY_IP_ADDRESS>/g' /etc/keepalived/keepalived.conf.template
      sed -i 's/VI_2_VIRTUAL_IPADDRESS/<HA_VI2_VIP>/g' /etc/keepalived/keepalived.conf.template
      cp /etc/keepalived/keepalived.conf.template /etc/keepalived/keepalived.conf

runcmd:
  - - '/bin/bash'
    - '-v'
    - '/usr/local/bin/build-configs'
  - - systemctl
    - restart
    - networking
  - - systemctl
    - enable
    - '--now'
    - resolvconf
  - - systemctl
    - enable
    - '--now'
    - keepalived
ssh_authorized_keys:
  - >
    <YOURKEY>

```

## References

- [NGINX Plus Documentation](https://docs.nginx.com/nginx/) - Documentation landing page for NGINX Plus
- [NGINX Plus HA](https://www.nginx.com/products/nginx/high-availability/) - Overview of the NGINX Plus HA Solution
- [High Availability Support for NGINX Plus in the NGINX Plus Admin Guide](https://docs.nginx.com/nginx/admin-guide/high-availability/ha-keepalived/)
  – Full instructions for configuring NGINX Plus for high availability in on‑premises deployments
- [Virtual Router Redundancy Protocol on Wikipedia](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol) –
  Overview of VRRP
- [keepalived home page](http://www.keepalived.org/) – Details about extending and customizing keepalived‘s
  configuration