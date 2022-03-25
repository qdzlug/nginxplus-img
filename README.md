## NGINX Plus VM Image Build

This repository will walk you through the process of building a qcow2 image file that will enable you to deploy NGINX
Plus in a VM. Note that this is an exceptionally opinionated build; please see the [Usage Notes](./docs/Usage.md)
for full details on how to use this image, as well as information on the build decisions.

## Important Caveats

- You need to have a supported linux build environment. These builds were done on Ubuntu 21.04; your mileage may vary on
  other platforms.

- This process uses [diskimage-builder](https://docs.openstack.org/diskimage-builder/latest/index.html) to create the
  image.

- This process requires that you have a valid cert/key pair for NGINX Plus.

- The cert and key are used in the build, but are not contained in the repository. They are injected at build time from
  Jenkins; if you are not using Jenkins you can replicate this by copying them to the appropriate directories under
  the `static` part of the build hierarchy.

- This process will build a bare-bones deployment that exposes the API and nginx-dashboard as part of the deployment;
  this is only for development and testing purposes, with the configuration being passed in a heredoc. For actual usage,
  the data should be passed using the disk-image-builder `extra-data.d` element.

- The image built is can be based off Debian 11 (Bullseye) or Ubuntu 20.04 (Focal). There is a directory for a CentOS
  build but this option is deprecated and not maintained.

- Sample cloud configurations are included.

## Setup

It is recommended that you install the diskimage builder in a virtualenv:

- `sudo apt install virtualenvwrapper`
- `virtualenv dib-elements`
- `source dib-elements/bin/activate`
- ` pip install diskimage-builder`

## Key Files

A full discussion of the diskimage build system is out of scope for this document, and instead we will focus on the key
configuration files for this project. It is *highly* recommended that you review the documentation on the build process
prior to attempting any major changes in order to fully understand the possible effects of those changes.

The file/directory structure for the build process looks like this:

```
nginx-plus
├── centos
│   └── elements
│       ├── dib-elements
│       │   ├── bin
│       │   ├── lib
│       │   └── share
│       └── nginx-plus
│           ├── environment.d
│           ├── extra-data.d
│           ├── install.d
│           ├── nginxplus.d
│           └── post-install.d
├── debian
    └── elements
        ├── dib-elements
        │   ├── bin
        │   ├── lib
        │   └── share
        └── nginx-plus
            ├── environment.d
            ├── extra-data.d
            ├── install.d
            ├── post-install.d
            └── static
└── ubuntu
    └── elements
        ├── dib-elements
        │   ├── bin
        │   ├── lib
        │   └── share
        └── nginx-plus
            ├── environment.d
            ├── extra-data.d
            ├── install.d
            ├── nginxplus.d
            ├── post-install.d
            └── static
```

| File/Directory            | Purpose                                                                                                                                            |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| element-deps              | List of the key dependencies for the image buid; includes the base image to be used and what tasks to complete (such as enabling a serial console) |
| environment.d             | Files that define the environment                                                                                                                  |
| 10-cloud-init-datasources | Configures cloud-init for various clouds / environments. For example, OpenStack and Ec2.                                                           |
| extra-data.d              | Currently empty; can be used to pass files into the build environment.                                                                             |
| install.d                 | Files used during the install phase of the build.                                                                                                  |
| 15-install-nginx-plus     | Customized script to install nginx-plus; uses an external webserver for certs and has an inline configuration.                                     |
| package-installs.yaml     | List of packages to be installed                                                                                                                   |
| pkg-map                   | Mapping between package names between releases and families                                                                                        |
| post-install.d            | Currently empty; scripts / steps to be run following installation                                                                                  |
| static                    | Static files to be copied directly into the image                                                                                                  |

Of these, the most likely files you will need to change are:

- The cloud init datasources, in order to build images for different clouds and environments.
- The install script, for a number of reasons:
    - Change the location of where the scripts are pulled from.
    - Change the configuration that is passed through.
    - Make use of the extra-data.d process to bring different content into the image.
    - Add files into the static directory to build bespoke files into the image.
- Package installs, to add/remove packages from the final image.

## Cert Management

This process is designed to stamp out preconfigured images w/ NGINX Plus installed without containing the cert/key pair.
These are passed through using the static directory; please see the relevant configuration directory for full details 
on how to accomplish this.

## Build Process

The build process involves two steps:

1. Export the path to `elements` directory you created for the build type you wish (Debian or CentOS); this can be
   absolute or relative. For example, `export ELEMENTS_PATH=./elements`
   or `export ELEMENTS_PATH=$HOME/repos/nginxplus-img/debian/elements`
2. Set your version variable. Currently, has been tested with `DIB_RELEASE=bullseye` for
   Debian and `DIB_RELEASE=focal` for Ubuntu.
2. Run the disk image creation process. This requires that you provide:
    1. The architecture (in this case amd64)
    2. The output file, including the file extension indicating the format.
    2. The partition format being used; in this case we are using a block device with an MBR.
    3. The name of the project being built, in this case nginx-plus.
4. Command line: `disk-image-create -a amd64 -o nginxplus.qcow2 block-device-mbr nginx-plus`

## Example

 ```
$ disk-image-create -a amd64 -o nginxplus.qcow2 block-device-mbr nginx-plus
2021-05-26 22:20:08.313 | diskimage-builder version 3.11.0
2021-05-26 22:20:08.314 | Building elements: base block-device-mbr nginx-plus
2021-05-26 22:20:08.372 | Expanded element dependencies to: growroot runtime-ssh-host-keys install-static cloud-init-datasources openssh-server sysprep modprobe block-device-mbr base pkg-map install-bin vm debootstrap nginx-plus manifests package-installs dhcp-all-interfaces enable-serial-console dpkg install-types dib-init-system debian-minimal bootloader
2021-05-26 22:20:12.557 | Building in /tmp/jschmidt/dib_build.j0WKmm8c
2021-05-26 22:20:12.564 | Searching elements for block-device[-amd64].yaml ...
2021-05-26 22:20:12.566 | Using block-device config: /home/jschmidt/.local/lib/python3.8/site-packages/diskimage_builder/elements/block-device-mbr/block-device-default.yaml
2021-05-26 22:20:12.754 | INFO diskimage_builder.block_device.blockdevice [-] Wrote final block device config to [/tmp/jschmidt/dib_build.j0WKmm8c/states/block-device/config.json]
2021-05-26 22:20:12.950 | INFO diskimage_builder.block_device.blockdevice [-] Getting value for [root-label]
2021-05-26 22:20:13.145 | INFO diskimage_builder.block_device.blockdevice [-] Getting value for [root-fstype]
2021-05-26 22:20:13.337 | INFO diskimage_builder.block_device.blockdevice [-] Getting value for [mount-points]
<--- SNIP --->
2021-05-26 22:22:09.761 | INFO diskimage_builder.block_device.utils [-] Calling [sudo fstrim --verbose /tmp/jschmidt/dib_build.j0WKmm8c/mnt/]
2021-05-26 22:22:09.771 | INFO diskimage_builder.block_device.utils [-] Calling [sudo umount /tmp/jschmidt/dib_build.j0WKmm8c/mnt/]
2021-05-26 22:22:09.916 | INFO diskimage_builder.block_device.utils [-] Calling [sudo kpartx -d /dev/loop6]
2021-05-26 22:22:09.997 | INFO diskimage_builder.block_device.level0.localloop [-] loopdev detach
2021-05-26 22:22:09.997 | INFO diskimage_builder.block_device.utils [-] Calling [sudo losetup -d /dev/loop6]
2021-05-26 22:22:10.264 | INFO diskimage_builder.block_device.blockdevice [-] Removing temporary state dir [/tmp/jschmidt/dib_build.j0WKmm8c/states/block-device]
2021-05-26 22:22:10.342 | Converting image using qemu-img convert
2021-05-26 22:22:45.421 | Old image found. Renaming it to nginxplus-2021.05.26-16.22.45.qcow2
2021-05-26 22:22:45.423 | Image file nginxplus.qcow2 created...
2021-05-26 22:22:45.548 | Build completed successfully
```

### Notes

- If there are any issues with the build you will need to remediate them; the most common issues are around packages
  that need to be installed on the build machine.
- This process may require sudo permissions depending on your build environment.

## Post Build Editing

It is possible to make changes to the image after it is built without re-running the build process. This requires that
you use a utility such as [virt-customize](https://libguestfs.org/virt-customize.1.html). For example, the following
code can be used to add a user, set the password for the user, add the user to the sudoers file, and inject an ssh key
for the user:

```
 virt-customize -a theimage.qcow2 --run-command "adduser theuser"
 virt-customize -a theimage.qcow2 --run-command "echo 'theuser  ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers.d/theuser-sudo"
 virt-customize -a theimage.qcow2 --password theuser:password:thepass
 virt-customize -a theimage.qcow2 --ssh-inject theuser:string:"thesshkey"
 ````

You will need to substitute your own values for _theimage_, _theuser_, _thepass_, and _thesshkey_.

 

