## NGINX Plus VM Image Build
This repository will walk you through the process of building a qcow2 image file that will enable you to deploy NGINX Plus in a VM. 

## Important Caveats
- You need to have a supported linux build environment. These builds were done on Ubuntu 20.10; your mileage may vary on other platforms.
- This process uses [diskimage-builder](https://docs.openstack.org/diskimage-builder/latest/index.html) to create the image.
* This process requires that you have a valid cert/key pair for NGINX Plus.
* The cert and key are used in the buld, but are not contained in the repository. They are pulled from a local http server (instructions below) and removed from the image prior to it being finalized. 
- This process will build a bare-bones deployment that exposes the API and nginx-dashboard as part of the deployment; this is only for development and teting purposes, with the configuration being passed in a heredoc. For actual usage, the data should be passed using the disk-image-builder `extra-data.d` element.
- The image built is based off Debian 10, with a selection of packages suitable for debugging. These can be adjusted as needed.

## Setup
It is recommended that you install the diskimage builder in a virtualenv:
- `sudo apt install virtualenvwrapper`
- `virtualenv dib-elements`
- `source dib-elements/bin/activate`
- ` pip install diskimage-builder`


## Key Files
A full discussion of the diskimage build system is out of scope for this document, and instead we will focus on the key configuration files for this project. It is *highly* recommended that you review the documentation on the build process prior to attempting any major changes in order to fully understand the possible effects of thsoe changes.

The file/directory structure for the bulding process looks like this:

```
nginx-plus
├── element-deps
├── environment.d
│   └── 10-cloud-init-datasources
├── extra-data.d
├── install.d
│   └── 15-install-nginx-plus
├── package-installs.yaml
├── pkg-map
└── post-install.d
```

| File/Directory | Purpose |
|----------------|---------|
| element-deps   | List of the key dependencies for the image buid; includes the base image to be used and what tasks to complete (such as enabling a serial console)    |
| environment.d | Files that define the environment |
| 10-cloud-init-datasources                | Configures cloud-init for various clouds / environments. For example, OpenStack and Ec2.         |
| extra-data.d                | Currently empty; can be used to pass files into the build environment.         |
| install.d                | Files used during the install phase of the build. |
| 15-install-nginx-plus | Customized script to install nginx-plus; uses an external webserver for certs and has an inline configuration. |
| package-installs.yaml | List of packages to be installed |
| pkg-map | Mapping between package names between releases and families |
| post-install.d | Currently empty; scripts / steps to be run following installation |

OF these, the most likely files you will need to change are:
- The cloud init datasources, in order to build images for different clouds and environments.
- The install script, for a number of reasons:
    - Change the location of where the scripts are pulled from.
    - Change the configurtion that is passed through.
    - Make use of the extra-data.d process to bring different content into the image.
- Package installs, to add/remove packages from the final image.


## Cert Management
This process is designed to stamp out preconfigured images w/ NGINX Plus installed without containing the cert/key pair. The way to accomplish this is to put the files into a tar archive called `certs.tar` and serve it from any location you wish. The testing has all been done with the node-httpserver module from localhost. This can be installed with a simple `npm install http-server`, and then run in a directory containing your `certs.tar` file via `http-server`:

```
http-server
Starting up http-server, serving ./
Available on:
  http://127.0.0.1:8081
  http://192.168.213.42:8081
  http://192.168.213.63:8081
  http://10.48.208.1:8081
  http://172.17.0.1:8081
  http://10.1.164.128:8081
Hit CTRL-C to stop the server
```

## Build Processs
The build process involves two steps:
1. Export the path to `elements` directory you created when you cloned the repo; this can be absolute or relative. For example, `export ELEMENTS_PATH=./elements` or `export ELEMENTS_PATH=$HOME/repos/nginxplus-img/elements`
2. Run the disk image creation process. This requires that you provide:
    1. The architecture (in this case amd64)
    2. The output file, including the file extention indicating the format.
    2. The partition format being used; in this case we are using a block device with an MBR.
    3. The name of the project being built, in this case nginx-plus.
 `disk-image-create -a amd64 -o nginxplus.qcow2 block-device-mbr nginx-plus`

 ## Example 

 ```
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
- If there are any issues with the build you will need to remediate them; the most common issues are around packages that need to be installed on the build machine.
- This process may rquire sudo permissions depneding on your bulid environment. 

