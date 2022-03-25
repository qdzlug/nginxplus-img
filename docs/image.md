# Release Notes

## Image Description
These two images are both built to support an NGINX Plus deployment. Two versions have been provided:
- Debian 11 Bullseye
- Ubuntu 21.04 Focal

Additional details on the installation are provided below.

## Build Process
This image is built from the [nginxplus-img](https://github.com/qdzlug/nginxplus-img) github repository using the Jenkins CI system and the OpenStack [diskimage-builder](https://docs.openstack.org/diskimage-builder/latest/) program.

There are three main subdirectories:
- `debian`: Debian image build files.
- `ubuntu`: Ubuntu image build files.
- `centos`: Centos image build files (deprecated).

Each subdirectory contains a `Jenkinsfile` with the pipeline logic for the build.


## Build Considerations

This is a custom image that has been built specifically to be deployed in an OpenStack environment that does not provide correct metadata for interface configuration. As this is not a normal configuration, several changes have needed to be made to the image in order to prevent the network from being autoconfigured either by the standard boot process or by cloud-init. 

### Image Changes
- The creation of `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` with the contents:
	```
	network:
  config: disabled
  disable_network_activation: true
  ```
- Setting the `CONFIGURE_INTERFACES` variable to `NO` in `/etc/default/networking`.
- Setting the `DIB_DHCP_NETWORK_MANAGER_AUTO` variable to `true` for the build process.

### Deployment Considerations
Since this image has been configured to not automatically plumb network interfaces the networking must be set as part of the deployment via a cloud-init file. This includes adding the DNS resolver information.

### Networking Stanzas
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
- The interfaces for both the Debian and Ubuntu distributions are in the format of `enpXs0` where `X` is the interface number starting at `1`. 
- If multiple interfaces are defined they should be put into files named with the name of the interface and placed in the `/etc/network/interfaces.d` directory.
- If multiple interfaces are defined, only one default gateway should be defined.

### User Account Stanzas
By default, both Debian and Ubuntu lock user accounts from logging in from the console, although they are allowed to log in via ssh. To address this they need to both have their password set and the their account unlocked. This can be accomplished by following this example; note the `lock-passwd` and `passwd` directives. 

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
      jschmidt@jack.virington.com
```

#### Notes
- The password supplied needs to be hashed; you can accomplish this by running the following command on a linux system and then cutting/pasting the resultant hash into the configuration.
	`mkpasswd --method=SHA-512 --rounds=4096`
	
### Data Cache and Machine ID
By default, the cloud-init process checks the cloud-init cache and machine ID on each boot, and will re-run if the machine ID is different. This means that if a VM is migrated the cloud-init process will re-run when it is started on a new host machine. Since this is normally not the desired behavior the following stanza should be set in the cloud-init file:
	`manual_cache_clean: true`
	
If this is set and later it is decided that you wish to have cloud-init re-run, this can be accomplished by deleting the `/var/lib/cloud` directory.

### Hostname Management
Although it is possible for an image to receive it's hostname from the hypervisor, in the case where we want to manage the hostname ourselves we can accomplish this by setting two configuration lines in the cloud-init file:

```
manage_etc_hosts: true
hostname: nginix-test
```

### Service Management
To ensure that the networking and resolver configuration are applied correctly, the last step that the cloud-init process should initate is a restart of these services:

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

Note the configuration of the above; since cloud-init is yaml based it is expecting arrays of commands. Care should be taken with any characters that are significant in yaml - for example, colons. In this case, they should be quoted or quoted and passed in an array.


## Image Details

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



