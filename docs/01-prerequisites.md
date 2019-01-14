# Prerequisites

## Openstack Cloud Platform: Bluvalt Cloud

The guide leverages Openstack cloud platform to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Bluvalt Cloud](https://cloud.bluvalt.com/) is used to demonestrate the lab.

All configuration is going to be achieved using Openstack API. The guide helps on [Openstack API Access](https://github.com/omermahgoub/openstack/wiki/Openstack-API-access). 


## OpenStack command-line clients

### Install the Google Cloud SDK

Follow OpenStack command-line clients [documentation](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html) to install and configure Openstack command line clients.

Verify the Google Cloud SDK version is 218.0.0 or higher:
```
openstack --version
```
### Set environment variables using the OpenStack RC file

To set the required environment variables for the OpenStack command-line clients, you must create an environment file called an OpenStack rc file, or openrc.sh file. You can download the file from [here](https://github.com/omermahgoub/openstack/tree/master/API), make sure to download the one that correspond to your project datacenter. This project-specific environment file contains the credentials that all OpenStack services use.

In order to use the OpenRC file, run:
``` 
source jed1
```
### Basic commands

```
openstack server list

openstack image list

```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
