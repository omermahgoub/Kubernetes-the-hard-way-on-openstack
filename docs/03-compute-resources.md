# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single datacenter [jed1] (http://jed1-vdc.bluvalt.com/).


## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Public Network
The public network is a network that contains external access and can be reached by the outside world is already created on Bluvalt cloud. 
```
openstack network list
```

`Output is `

```
| 80e37cda-1762-4d98-8e55-df3e33710295 | admin_floating_net      | 96368227-27ff-4b66-b9bf-ba8c4118e2c2 |
```

### Virtual Network 

The virtual network is connected to the public network via a router during the network setup. This allows each OpenStack  instance attached to the virtual network the ability to request a floating IP from the public network for public access.

1.  Create the router kthw-router and set gateway on the public network:
```
openstack router create kthw-router
```

`Output`

```
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints | None                                 |
| availability_zones      | None                                 |
| created_at              | None                                 |
| description             | None                                 |
| distributed             | None                                 |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | None                                 |
| id                      | 3f1f9a37-5a4b-4fff-9325-7cd2f1990b87 |
| location                | None                                 |
| name                    | kthw-router                           |
| project_id              | a3a92951eb3b4e8a8746236ef7949cf6     |
| revision_number         | None                                 |
| routes                  | None                                 |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | None                                 |
+-------------------------+--------------------------------------+
```
```
openstack router set --external-gateway $(openstack network show admin_floating_net -f value -c id) $(openstack router show Router1 -f value -c id)
```
2.  Create the kthw-virtual-network private network and subnet kthw-virtual-subnet in this network with 10.240.0.0/24 range. 
    This range to assign a private IP address to each node in the Kubernetes cluster.
```
openstack network create kthw-virtual-network | openstack subnet create --subnet-range 10.240.0.0/24 --gateway auto --dhcp --network kthw-virtual-network --dns-nameserver 46.49.140.155 --dns-nameserver 8.8.8.8 kthw-subnet
```

`Output`

```
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.240.0.2-10.240.0.254              |
| cidr              | 10.240.0.0/24                        |
| created_at        | None                                 |
| description       | None                                 |
| dns_nameservers   | 46.49.140.155, 8.8.8.8               |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.240.0.1                           |
| host_routes       |                                      |
| id                | dac88cc2-3c83-4d65-97ce-8e54cdfbfdd7 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| location          | None                                 |
| name              | kthw-subnet                  |
| network_id        | 944ba535-f9ab-43cd-8909-e7093823232f |
| project_id        | a3a92951eb3b4e8a8746236ef7949cf6     |
| revision_number   | None                                 |
| segment_id        | None                                 |
| service_types     | None                                 |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | None                                 |
+-------------------+--------------------------------------+
```
3. Add kthw-subnet interface to the kthw-router 

```
openstack router add subnet kthw-router $(openstack subnet show kthw-subnet -f value -c id)
```

Public_IP=$(openstack floating ip create admin_floating_net -f value -c floating_ip_address)

### Public IP Address
Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:
```
Public_IP=$(openstack floating ip create admin_floating_net -f value -c floating_ip_address)
```

### Security Groups
Create a security group that allows internal communication across all protocols:
```
openstack security group create kthw-allow-internal
```
Add rules to security group
```
for i in tcp udp icmp
do
  openstack security group rule create \
      --ingress \
      --protocol ${i} \
      --remote-group kthw-allow-internal \
      kthw-allow-internal
done
```
Create a security group that allows external SSH, ICMP, and HTTPS:
```
openstack security group create kubernetes-the-hard-way-allow-external
```
Add rules for external SSH, ICMP, and HTTPS to security group 
```
openstack security group rule create \
    --ingress \
    --protocol icmp \
    kubernetes-the-hard-way-allow-external
```
Add rules for external SSH, 6443 to security group 
```
for port in 22 6443
do
  openstack security group rule create \
      --ingress \
      --protocol tcp \
      --dst-port ${port} \
      kubernetes-the-hard-way-allow-external
done
```
### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as describe in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1006-gcp x86_64)

...

Last login: Sun May 13 14:34:27 2018 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
