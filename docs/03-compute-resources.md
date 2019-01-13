# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single datacenter [jed1](http://jed1-vdc.bluvalt.com/).


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
80e37cda-1762-4d98-8e55-df3e33710295 | admin_floating_net      | 96368227-27ff-4b66-b9bf-ba8c4118e2c2 |
```
### Kubernetes Public IP Address

Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:
```
Public_IP=$(openstack floating ip create admin_floating_net -f value -c floating_ip_address)
```
### Virtual Network 

The virtual network is connected to the public network via a router during the network setup. This allows each OpenStack  instance attached to the virtual network the ability to request a floating IP from the public network for public access.

1.  Create the router kthw-router:
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
2. Connect the router to the gateway of the external network:
```
openstack router set --external-gateway $(openstack network show admin_floating_net -f value -c id) $(openstack router show Router1 -f value -c id)
```
3.  Create the kthw-virtual-network private network and subnet kthw-virtual-subnet in this network with 10.240.0.0/24 range. 
    This range to assign a private IP address to each node in the Kubernetes cluster.
    `P.S: Kubernetes nodes: 10.180.0.0/24 Services: 10.200.0.1.0/24 Pods: 10.34.128.0/17`
```
openstack network create kthw-virtual-network 
NETWORK=$(openstack network create kthw-virtual-network -f value -c id)

openstack subnet create --subnet-range 10.240.0.0/24 --gateway auto --dhcp --network kthw-virtual-network --dns-nameserver 46.49.140.155 --dns-nameserver 8.8.8.8 kthw-subnet
SUBNET=$(openstack subnet show kthw-subnet -f value -c id)
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
| name              | kthw-subnet                          |
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
4. Attach network `kthw-subnet` interface to the router `kthw-router`

```
openstack router add subnet kthw-router $(openstack subnet show kthw-subnet -f value -c id)
```

### Security Groups

1. Create a security group that allows internal communication across all protocols:
```
openstack security group create kthw-allow-internal
```
`Output`
```
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| created_at      | None                                 |
| description     | kthw-allow-internal                  |
| id              | 76901ae5-2245-4bb9-bbc4-b4dbc5bcf383 |
| location        | None                                 |
| name            | kthw-allow-internal                  |
| project_id      | a3a92951eb3b4e8a8746236ef7949cf6     |
| revision_number | None                                 |
| rules           |                                      |
| tags            | []                                   |
| updated_at      | None                                 |
+-----------------+--------------------------------------+
```
Add rules to security group
```
for i in tcp udp icmp
do
  openstack security group rule create \
      --ingress \
      --protocol ${i} \
      --remote-ip 10.240.0.0/24 \
      kthw-allow-internal
done
```
```
for i in tcp udp icmp
do
  openstack security group rule create \
      --ingress \
      --protocol ${i} \
      --remote-ip 10.200.0.0/16 \
      kthw-allow-internal
done
```
2. Create a security group that allows external SSH, ICMP, and HTTPS:
```
openstack security group create kthw-allow-external
```
Add rules for external SSH, ICMP, and HTTPS to security group 
```
openstack security group rule create \
    --ingress \
    --protocol icmp \
    kthw-allow-external
```
Add rules for external SSH, 6443 to security group 
```
for port in 22 6443
do
  openstack security group rule create \
      --ingress \
      --protocol tcp \
      --dst-port ${port} \
      kthw-allow-external
done
```
### Key pairs

This public key of an OpenSSH key pair to be used for access to created servers.
First generate SSH keys:
```
ssh-keygen
```
Then create SSH keypairs using the public key
```
openstack help keypair  create --public-key <file> id_rsa_public
```

## Compute Instances

The compute instances in this lab will be provisioned using Ubuntu Server 16.04, which has good support for the containerd container runtime. Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

```
openstack image list | grep Ubuntu-16.04.1-LTS | awk '{print $2}'
```
`Output`
```
f99bb6f7-658d-4f7c-840d-40228ee22581 | Ubuntu-16.04.1-LTS                        | active |
```
```
IMAGE=$(openstack image list | grep Ubuntu-16.04.1-LTS | awk '{print $2}')
```

### Flavor list

- For controllers, we will be using R1-Generic-4 which has 4 CPU and 8GB of RAM
- For workers, we will be using R1-Generic-8 which has 8 CPU and 16GB of RAM
```
openstack flavor list
```
`Output`
```
+--------------------------------------+---------------+--------+------+-----------+-------+-----------+
| ID                                   | Name          |    RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+---------------+--------+------+-----------+-------+-----------+
| 2e7f5353-cff7-4723-bbec-b35b12999d83 | R1-Generic-1  |   2048 |   30 |         0 |     1 | True      |
| 42386296-18b7-4f4d-8b9b-7f4a4c2bc469 | R1-Generic-8  |  16384 |   30 |         0 |     8 | True      |
| 496d049b-60b0-4b80-aa3e-570f7c3a5bb2 | R1-Network-16 |  32768 |   30 |         0 |    16 | True      |
| 61329a7d-ba26-49cc-bea7-fb58ae567d05 | R1-Network-2  |   4096 |   30 |         0 |     2 | True      |
| 661f0512-61e2-45dc-8c9c-cb4cb41d0a59 | R1-CPU-32     | 131072 |   30 |         0 |    32 | True      |
| 7f8fc7d3-4bf0-4e2a-b81b-030d3ab0e5d0 | R1-Generic-4  |   8192 |   30 |         0 |     4 | True      |
| 80ab04df-1b95-4555-aa5c-1c9427e3b3b2 | R1-Memory-4   |  32768 |   30 |         0 |     4 | True      |
| 963b2fe8-a36f-4d8c-b995-2e1b6007fe78 | R1-Generic-16 |  32768 |   30 |         0 |    16 | True      |
| 97bdad00-cb8a-4c61-b71f-8858e58545e2 | R1-CPU-48     | 262144 |   30 |         0 |    48 | False     |
| 9b8b3367-d80f-4f81-8f0c-ef1647444a7d | R1-Generic-2  |   4096 |   30 |         0 |     2 | True      |
| b375bb76-b7c0-4056-95d1-2499d089d02e | R1-Memory-32  | 262144 |   30 |         0 |    32 | True      |
| c640a747-7606-48f7-82a9-27c52f4153d7 | R1-Network-8  |  16384 |   30 |         0 |     8 | True      |
| c70c6310-13c4-4463-9196-f2328aef6140 | R1-Memory-16  | 131072 |   30 |         0 |    16 | True      |
| ea407ca0-3cdf-4b2b-acc2-6ca121f1f79c | R1-Memory-8   |  65536 |   30 |         0 |     8 | True      |
+--------------------------------------+---------------+--------+------+-----------+-------+-----------+
```
### Kubernetes Controllers
Create three compute instances which will host the Kubernetes control plane:
```
for i in 0 1 2; do
nova boot --block-device \
source=image,id=$IMAGE,dest=volume,size=200,bootindex=0,shutdown=preserve \
--nic net-id=$NETWORK,v4-fixed-ip=10.240.0.1${i} \
--flavor R1-Generic-4 \
--key-name id_rsa_public --security-groups kthw-allow-internal,kthw-allow-external controller-${i}
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The pod-cidr instance metadata will be used to expose pod subnet allocations to compute instances at runtime.
```
for i in 0 1 2; do
    nova boot --block-device source=image,id=$IMAGE,dest=volume,size=200,bootindex=0,shutdown=preserve \
    --nic net-id=$NETWORK,v4-fixed-ip=10.240.0.2${i} \
    --flavor R1-Generic-8 \
    --key-name id_rsa_public --security-groups kthw-allow-internal,kthw-allow-external worker-${i}
done
```

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets

### Proxy and Loadbalancer VMs
This instance will be using as a bastion (Jumphost) and a load balancer for kubernetes APIs.

```
nova boot --block-device source=image,id=$IMAGE,dest=volume,size=100,bootindex=0,shutdown=preserve \
--nic net-id=$NETWORK,v4-fixed-ip=10.240.0.200 \
--flavor R1-Generic-4 \
--key-name id_rsa_public --security-groups kthw-allow-internal,kthw-allow-external prx-1
```

We can now associate a floatingip with pry to access the kubenet:

```
openstack server add floating ip prx-1 $Public_IP
```

### Verification

List the compute instances in your default compute zone:

```
openstack server list | grep -E "work|cont|prx"
```

> output
```
| ID                                   | Name          | Status | Networks                             | Image | Flavor        |
| 1ee2432b-8740-41f6-bc8d-f876c2550596 | prx-1         | ACTIVE | kthw-virtual-network=10.240.0.200    |       | R1-Generic-4  |
| 199e46ad-115a-48d1-9d4b-4a2e193a8573 | worker-2      | ACTIVE | kthw-virtual-network=10.240.0.22     |       | R1-Generic-8  |
| d391b9a3-da05-47fe-9035-342c29819c80 | worker-1      | ACTIVE | kthw-virtual-network=10.240.0.21     |       | R1-Generic-8  |
| d99fb78b-9893-4c26-98c3-7c0cafae635e | worker-0      | ACTIVE | kthw-virtual-network=10.240.0.20     |       | R1-Generic-8  |
| 558aeb3f-5237-4e96-9a3b-9d43edd8d0b7 | controller-2  | ACTIVE | kthw-virtual-network=10.240.0.12     |       | R1-Generic-4  |
| 8064de88-c780-4c80-b522-e6cc0505c975 | controller-1  | ACTIVE | kthw-virtual-network=10.240.0.11     |       | R1-Generic-4  |
| 1e4546a7-5404-4f83-8c1a-837849080dea | controller-0  | ACTIVE | kthw-virtual-network=10.240.0.10     |       | R1-Generic-4  |
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. We will use prx-1 as bastion host to connect to controller and worker instances.

Test SSH access to the `prx-1` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
ssh ubuntu@95.177.214.110
The authenticity of host '95.177.214.110 (95.177.214.110)' can't be established.
ECDSA key fingerprint is SHA256:rT1oted6q4UDHFPerlhh0zd+XUZjAlxbU3cRompqNPs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '95.177.214.110' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-98-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```

As all instances can only be accessed via private key, I'll upload my private key to `prx-1` to be able to access controller and worker instances from the bastion host.
```
scp .ssh/id_rsa ubuntu@95.177.214.110:/home/ubuntu/.ssh/
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
