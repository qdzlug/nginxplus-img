## NGINX Plus VM Image Build
This repository will walk you through the process of building a qcow2 image file that will enable you to deploy NGINX Plus in a VM. 

## Important Caveats
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



## Cert Management

