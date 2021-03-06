GCP Networking
==============

https://cloud.google.com/solutions/best-practices-vpc-design

3 VPC Network Types
-------------------

* Default

  * Every Project
  * One subnet per region
  * Default firewall rule


* Auto mode

  * Default network
  * One subnet per region
  * Regional IP allocation
  * Fixed /20 subnetwork perregion
  * Expandable up to /16


* Custom mode

  * No default subnets
  * Full control of IP ranges
  * Regional IP allocation
  * Expandable to any RFC 1918 size


Subnet
------

* cross zones ( us-east1-a, us-east1-b ... ) in a region
* 4 IPs are reserved in Subnets
* RFC 1918 address spaces
* Can expand without downtime, but not shrink
* Aut-mode can be expanded from /20 to /16
* Avoid large subnets
 

VM Instance IP
--------------

* OS of VM doesn't know external IP regardless ephemeral or static IP. ( ifconfig only show internal IP )
* The external IP is mapped to VMs internal IP transparently by VPC.


DNS resolution for internal address
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

* Each VM Instance has a **hostname** that can be resolved to an internal IP.

  * The hostname is the same as the instance name.
  * FQDN is [hostname].c.[project-id].internal
  
* Name resolution is handled by **internal DNS**

  * Provided as part of Compute Engine
  * Configured via DHCP
  * Provides answer for internal and external IP


DNS resolution for external address
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

* DNS zone can be hosted using Cloud DNS


Routes / Firewall
-----------------

A route is a mapping of an IP range to a destination
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

Every VPC network has

* Routes that let VM Instance in a VPC network send traffic directly to each other.
* A default route that directs packets to destinations that are outside the network.

Firewall rules must also allow the packet.


Routes map traffic to destination networks
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

* Destinations in CIDR notation
* Applies to traffic egressing a VM
* Forwards traffic to most specific route
* Traffic is delivered only if it also matches a firewall rule
* Created when a subnet is created
* Enables VMs on same network to communicate 


Firewall rules protects VM Instance from unapproved connections
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

* VPC network functions as a distributed firewall
* Firewall rules are applied to the network as a whole
* Connections are allowed or denied at the VM instance level
* Firewall rules are staeful
* Implied deny all ingress and allow all egress


Multiple Network Interfaces on VM Instance
------------------------------------------

VPC Networks are isolated ( by default )

* Commnicate within networks using **internal IP**
* Commnicate across networks using **external IP**


Mutiple Network Interfaces

* Network Interface controllers (NICs)
* Each NIC is attached to a VPN network
* Communicate across networks using **internal IP**

.. image:: images/multiple_network_interfaces.png

https://cloud.google.com/vpc/docs/create-use-multiple-interfaces#limitations


VM Instance with multiple Network Interfaces
--------------------------------------------

.. code-block:: ip route

  default via 172.16.0.1 dev eth0
  10.128.0.0/20 via 10.128.0.1 dev eth2
  10.128.0.1 dev eth2 scope link
  10.130.0.0/20 via 10.130.0.1 dev eth1
  10.130.0.1 dev eth1 scope link
  172.16.0.0/24 via 172.16.0.1 dev eth0
  172.16.0.1 dev eth0 scope link


The primary interface eth0 gets the default route (default via 172.16.0.1 dev eth0), and all three interfaces, eth0, eth1, and eth2, get routes for their respective subnets. Because the subnet of mynet-eu-vm (10.132.0.0/20) is not included in this routing table, the ping to that instance leaves vm-appliance on eth0 (which is on a different VPC network). You could change this behavior by configuring policy routing as documented here.

Shared VPC
----------

* https://www.youtube.com/watch?v=4MtfyViH9t0
* https://cloud.google.com/vpc/docs/provisioning-shared-vpc


.. image:: images/shared_vpc_vs_peering.png


VPC Peering
-----------

* Peered VPC networks reamin administratively seprate
* Each side of a peering assocation is set up independently ( one side can disconnect )
* No Subnet IP overlapping
* Transitive peering is not supported. ( A-B-C doesn't make A-C communication )
* Compute engines' internal DNS names are created in a VPC network and those are not accessible to peer VPC networks.
* VPC Peering generats Peering Routes.


Load Balancer
-------------

Global
>>>>>>

* HTTP(S)
* SSL Proxy
* TCP Proxy


Regional
>>>>>>>>

* Internal TCP/UDP
* Network TCP/UDP


Managed Instance groups autoscaling
-----------------------------------

* Dynamicall add/remove instances

  * Increase/Decrease in load
  
* Autoscaling policy

  * CPU Utilization
  * Load balancing capacity
  * Monitoring metrics
  * Queue based workloads


HTTPS Load Balancing
--------------------

* Global Load balancing
* Anycast IP address
* HTTP on port 80 or 8080
* HTTs on port 443
* IPv6 and IPv4 Clients
* Autoscaling
* URL maps

Architecture of an HTTP(S) load balancer
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

.. image:: images/achitecture_of_http_loadbalancing.png


Backend Services
>>>>>>>>>>>>>>>>

* Health Check
* Session affinity (optional): attempts to send all requests from the same client to the same virtual machine instance
* Time setting (30sec default)
* One or more backends

  * An instance group ( managed or unmanaged )
  * A balancing mode ( CPU utilization or RPS )
  * A capacity scaler ( celiling % of CPU/Rate targets )
  


Configuring an HTTP Load Balancer for Practice
----------------------------------------------

1. Create HTTP and health check firewall rules
2. Configure two instance templates
3. Create two managed instance groups
4. Configure an HTTP load balancer with IPv4 and IPv6
5. Stress test an HTTP load balancer
6. Blacklist an IP address to restrict access to an HTTP load balancer


Metadata ( For testing )

* startup-script-url:  gs://cloud-training/gcpnet/httplb/startup.sh


Configure the HTTP load balancer
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

Health checks determine which instances can receive new connections. This HTTP health check polls instances every 5 seconds, waits up to 5 seconds for a response, and treats 2 successful or 2 failed attempts as healthy or unhealthy, respectively.

HTTP(S) load balancing supports both IPv4 and IPv6 addresses for client traffic. Client IPv6 requests are terminated at the global load balancing layer and then proxied over IPv4 to your backends.


Stress Test
>>>>>>>>>>>

.. code-block:: bash

  sudo apt-get -y install siege
  siege -c 250 http://35.227.238.156



Cloud Armor - Blacklist IP
--------------------------

* Cloud Armor enables you to restrict or allow access to the HTTPS load balancer at the edge of the GCP network.
* A DDos attack can be blocked directly at the edge without consuming resources or entering your VPC network.

.. image:: images/how_cloud_armor_works.png


Configure Cloud Armor
>>>>>>>>>>>>>>>>>>>>>

* Default rule action: Allow
* Add rule

  * Action: Deny
  * Deny status: 403 (Forbidden)
  * Priority: 1000

It might take a couple of minutes for the security policy to take affect. If you are able to access the backends, keep trying until you get the 403 Forbidden error.


Cloud CDN
---------

* https://cloud.google.com/cdn/docs/locations


Configure Cloud CDN
>>>>>>>>>>>>>>>>>>>

* Create and populate a Cloud Storage bucket
* Create an HTTP load balancer with Cloud CDN
* Verify the caching of your bucket's content ( Logging )


SSL Proxy Load-Balancing
------------------------

* Global load-balancing for encrypted, non-HTTP traffic
* Terminates SSL sessions at balancing layer - And balance the connections across instances using SSL or TCP ( closest one having capacity )
* IPv4 or IPv6 clients
* Benefits:

  * Intelligent Routing
  * Certificate management
  * Security patching
  
* https://cloud.google.com/load-balancing/docs/ssl/#overview

.. image:: images/ssl_proxy_load_balancing.png


TCP Load-Balancing
------------------

* Global load-balancing for unencrypted, non-HTTP traffic
* Terminates SSL sessions at balancing layer
* IPv4 or IPv6 clients
* Benefits:

  * Intelligent Routing
  * Security patching

* https://cloud.google.com/load-balancing/docs/tcp/#overview


Network Load-Balancing
----------------------

* Regional, non-proxied load-balancer ( traffic is passed through the load-balancer instead of being proxied )
* Forwarding rules(IP protocol data)
* Traffic

  * UDP
  * TCP/SSL ports

* Backend

  * Instance group
  * Target pool


Internal Load-Balancing
-----------------------

* Regional, private load balancing

  * VM instances in same region
  * RFC 1918 IP Address

* TCP/UDP Traffic
* Reduced latency(since staging internal in Google), simpler configuration
* software-defined, fully distributed load balancing ( built on top of Andromeda )

* https://cloudplatform.googleblog.com/2014/04/enter-andromeda-zone-google-cloud-platforms-latest-networking-stack.html


.. image:: images/internal_load_balancing.png

Configure Internal Load-Balancing
---------------------------------

.. image:: images/configure_internal_load_balacing.png


Choose a load balancer
----------------------

.. image:: images/ipv6_termination_for_load_balancing.png

.. image:: images/choosing_a_load_balancer.png



Hybrid connectivity
-------------------

* Services

  * Layer 3 - Dedicated: Direct Peering
  * Layer 2 - Dedicated: Dedicated Interconnect
  * Layer 3 - Shared: Carrier Peering
  * Layer 2 - Shared: Partner


* Dedicated / Shared

  * Direct connection to Google's Network
  * Shared connection to Google's Network through a partner


* Layer2 / Layer3

  * Layer2 connections use a VLAN that pipes directly into your GCP environment, providing connectivity to internal IP addresses in the RFC 1918 address space.


Pricing
-------


Network Service Tiers
---------------------

* Premium Tier: Less Hop Points since the traffic Google's Global Network

Premium Tier delivers traffic over Google’s well-provisioned, low-latency, highly reliable global network. This network consists of an extensive global private fiber network with over 100 points of presence (POPs) across the globe.

* Standard Tier: More Hop Points since the traffic goes through Public ISP.

Standard Tier is a new, lower-cost offering. The network quality of this tier is comparable to the quality of other public cloud providers and regional network services, such as regional load balancing with one VIP per region, but lower than the quality of Premium Tier.


Configure a Cloud NAT gateway
-----------------------------

Cloud NAT is a regional resource. You can configure it to allow traffic from all ranges of all subnets in a region, from specific subnets in the region only, or from specific primary and secondary CIDR ranges only.

Network services > Cloud NAT

* Gateway name: nat-config
* VPC network: privatenet
* Region: us-central1
* Create new router

  * Name: nat-router


Automating the Deployment of Networks Using Deployment Manager
--------------------------------------------------------------

gcloud deployment-manager types list | grep network


.. code-block:: ymal

  # autonetwork-template.jinja
  resources:
  - name: {{ env["name"] }}
    type: compute.v1.network
    properties:
      # automatically creates a subnetwork
      autoCreateSubnetworks: true


.. code-block:: ymal

  # customnetwork-template.jinja
  resources:
  - name: {{ env["name"] }}
    type: compute.v1.network
    properties:
      autoCreateSubnetworks: false


.. code-block:: yaml

  # subnetwork-template.jinja
  resources:
  - name: {{ env["name"] }}
    type: compute.v1.subnetwork
    properties:
      ipCidrRange: {{ properties["ipCidrRange"] }}
      network: {{ properties["network"] }}
      region: {{ properties["region"] }}


.. code-block:: yaml

  # firewall-template.jinja
  resources:
  - name: {{ env["name"] }}
    type: compute.v1.firewall
    properties:
      network: {{ properties["network"] }}
      sourceRanges: ["0.0.0.0/0"]
      allowed:
      - IPProtocol: {{ properties["IPProtocol"] }}
        ports: {{ properties["Port"] }}


.. code-block:: yaml

  # instance-template.jinja
  resources:
  - name: {{ env["name"] }}
    type: compute.v1.instance  
    properties:
       machineType: zones/{{ properties["zone"] }}/machineTypes/{{ properties["machineType"] }}
       zone: {{ properties["zone"] }}
       networkInterfaces:
        - network: {{ properties["network"] }}
          subnetwork: {{ properties["subnetwork"] }}
          accessConfigs:
          - name: External NAT
            type: ONE_TO_ONE_NAT
       disks:
        - deviceName: {{ env["name"] }}
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/family/debian-9


.. code-block:: yaml

  # config.yaml
  imports:
  - path: autonetwork-template.jinja
  - path: customnetwork-template.jinja
  - path: subnetwork-template.jinja
  - path: firewall-template.jinja
  - path: instance-template.jinja

  # mynetwork setting
  resources:
  - name: mynetwork
    type: autonetwork-template.jinja

  - name: mynetwork-allow-http-ssh-rdp
    type: firewall-template.jinja
    properties:
      network: $(ref.mynetwork.selfLink)
      IPProtocol: TCP
      Port: [22, 80, 3389]

  - name: mynetwork-allow-icmp
    type: firewall-template.jinja
    properties:
      network: $(ref.mynetwork.selfLink)
      IPProtocol: ICMP
      Port: []

  # managementnet setting
  - name: managementnet
    type: customnetwork-template.jinja

  - name: managementsubnet-us
    type: subnetwork-template.jinja
    properties:
      ipCidrRange: 10.130.0.0/20
      network: $(ref.managementnet.selfLink)
      region: us-central1

  - name: managementnet-allow-http-ssh-rdp
    type: firewall-template.jinja
    properties:
      network: $(ref.managementnet.selfLink)
      IPProtocol: TCP
      Port: [22, 80, 3389]

  - name: managementnet-allow-icmp
    type: firewall-template.jinja
    properties:
      network: $(ref.managementnet.selfLink)
      IPProtocol: ICMP
      Port: []

  # privatenet setting
  - name: privatenet
    type: customnetwork-template.jinja

  - name: privatesubnet-us
    type: subnetwork-template.jinja
    properties:
      ipCidrRange: 172.16.0.0/24
      network: $(ref.privatenet.selfLink)
      region: us-central1

  - name: privatesubnet-eu
    type: subnetwork-template.jinja
    properties:
      ipCidrRange: 172.20.0.0/24
      network: $(ref.privatenet.selfLink)
      region: europe-west1

  - name: privatenet-allow-http-ssh-rdp
    type: firewall-template.jinja
    properties:
      network: $(ref.privatenet.selfLink)
      IPProtocol: TCP
      Port: [22, 80, 3389]

  - name: privatenet-allow-icmp
    type: firewall-template.jinja
    properties:
      network: $(ref.privatenet.selfLink)
      IPProtocol: ICMP
      Port: []

  # instances
  - name: mynet-us-vm
    type: instance-template.jinja
    properties:
      zone: us-central1-a
      machineType: n1-standard-1
      network: $(ref.mynetwork.selfLink)
      subnetwork: regions/us-central1/subnetworks/mynetwork

  - name: mynet-eu-vm
    type: instance-template.jinja
    properties:
      zone: europe-west1-d
      machineType: n1-standard-1
      network: $(ref.mynetwork.selfLink)  
      subnetwork: regions/europe-west1/subnetworks/mynetwork

  - name: privatenet-us-vm
    type: instance-template.jinja
    properties:
      zone: us-central1-a
      machineType: n1-standard-1
      network: $(ref.privatenet.selfLink)
      subnetwork: $(ref.privatesubnet-us.selfLink)

  - name: managementnet-us-vm
    type: instance-template.jinja
    properties:
      zone: us-central1-a
      machineType: n1-standard-1
      network: $(ref.managementnet.selfLink)
      subnetwork: $(ref.managementsubnet-us.selfLink)


.. code-block:: bash

  gcloud deployment-manager deployments create gcpnet --config=config.yaml


Automating the Deployment of Networks Using Terraform
-----------------------------------------------------

sample: https://registry.terraform.io/browse/modules?provider=google&verified=true

.. code-block:: bah

  ├── instance
  │   └── main.tf
  ├── managementnet.tf
  ├── privatenet.tf
  ├── mynetwork.tf
  └── provider.tf


.. code-block:: tf

  # provider.tf
  provider "google" {}


.. code-block:: tf

  # managementnet.tf

  # Create the managementnet network
  resource "google_compute_network" "managementnet" {
    name                    = "managementnet"
    auto_create_subnetworks = "false"
  }

  # Create managementsubnet-us subnetwork
  resource "google_compute_subnetwork" "managementsubnet-us" {
    name          = "managementsubnet-us"
    region        = "us-central1"
    network       = google_compute_network.managementnet.self_link
    ip_cidr_range = "10.130.0.0/20"
  }

  # Add a firewall rule to allow HTTP, SSH, and RDP traffic on managementnet
  resource "google_compute_firewall" "managementnet-allow-http-ssh-rdp-icmp" {
    name    = "managementnet-allow-http-ssh-rdp-icmp"
    network = google_compute_network.managementnet.self_link
    allow {
      protocol = "tcp"
      ports    = ["22", "80", "3389"]
    }
    allow {
      protocol = "icmp"
    }
  }

  # Add the managementnet-us-vm instance
  module "managementnet-us-vm" {
    source              = "./instance"
    instance_name       = "managementnet-us-vm"
    instance_zone       = "us-central1-a"
    instance_subnetwork = google_compute_subnetwork.managementsubnet-us.self_link
  }
  
  
.. code-block:: tf

  # instance/main.tf

  variable "instance_name" {}
  variable "instance_zone" {}
  variable "instance_type" {
    default = "n1-standard-1"
    }
  variable "instance_subnetwork" {}

  resource "google_compute_instance" "vm_instance" {
    name         = "${var.instance_name}"
    zone         = "${var.instance_zone}"
    machine_type = "${var.instance_type}"
    boot_disk {
      initialize_params {
        image = "debian-cloud/debian-9"
        }
    }
    network_interface {
      subnetwork = "${var.instance_subnetwork}"
      access_config {
        # Allocate a one-to-one NAT IP to the instance
      }
    }
  }


.. code-block:: tf

  # privatenet.tf

  # Create privatenet network
  resource "google_compute_network" "privatenet" {
    name                    = "privatenet"
    auto_create_subnetworks = false
  }

  # Create privatesubnet-us subnetwork
  resource "google_compute_subnetwork" "privatesubnet-us" {
    name          = "privatesubnet-us"
    region        = "us-central1"
    network       = google_compute_network.privatenet.self_link
    ip_cidr_range = "172.16.0.0/24"
  }

  # Create privatesubnet-eu subnetwork
  resource "google_compute_subnetwork" "privatesubnet-eu" {
    name          = "privatesubnet-eu"
    region        = "europe-west1"
    network       = google_compute_network.privatenet.self_link
    ip_cidr_range = "172.20.0.0/24"
  }

  # Create a firewall rule to allow HTTP, SSH, RDP and ICMP traffic on privatenet
  resource "google_compute_firewall" "privatenet-allow-http-ssh-rdp-icmp" {
    name    = "privatenet-allow-http-ssh-rdp-icmp"
    network = google_compute_network.privatenet.self_link
    allow {
      protocol = "tcp"
      ports    = ["22", "80", "3389"]
    }
    allow {
      protocol = "icmp"
    }
  }

  # Add the privatenet-us-vm instance
  module "privatenet-us-vm" {
    source              = "./instance"
    instance_name       = "privatenet-us-vm"
    instance_zone       = "us-central1-a"
    instance_subnetwork = google_compute_subnetwork.privatesubnet-us.self_link
  }


.. code-block:: tf

  # mynetwork.tf

  # Create the mynetwork network
  resource "google_compute_network" "mynetwork" {
    name                    = "mynetwork"
    auto_create_subnetworks = true
  }

  # Create a firewall rule to allow HTTP, SSH, RDP and ICMP traffic on mynetwork
  resource "google_compute_firewall" "mynetwork_allow_http_ssh_rdp_icmp" {
    name    = "mynetwork-allow-http-ssh-rdp-icmp"
    network = google_compute_network.mynetwork.self_link
    allow {
      protocol = "tcp"
      ports    = ["22", "80", "3389"]
    }
    allow {
      protocol = "icmp"
    }
  }

  # Create the mynet-us-vm instance
  module "mynet-us-vm" {
    source              = "./instance"
    instance_name       = "mynet-us-vm"
    instance_zone       = "us-central1-a"
    instance_subnetwork = google_compute_network.mynetwork.self_link
  }

  # Create the mynet-eu-vm" instance
  module "mynet-eu-vm" {
    source              = "./instance"
    instance_name       = "mynet-eu-vm"
    instance_zone       = "europe-west1-d"
    instance_subnetwork = google_compute_network.mynetwork.self_link
  }

.. code-block:: bash

  # Initialize Terraform
  terraform init

  # Rewrite the Terraform configurations files to a canonical format and style by running the following command:
  terraform fmt

  # Create an execution plan by running the following command:
  terraform plan

  # Apply the desired changes by running the following command:
  terraform apply


Dynamic VPN gateways with Cloud Routers
---------------------------------------

* Create VPC Network (gcp-vpc)

  * Subnet Name: subnet-a
  * Region: us-central1
  * IP address range: 10.5.4.0/24

* Create VPC Network (on-prem)

  * Subnet Name: subnet-b
  * Region: europe-west1
  * IP address range: 10.1.3.0/24


* Create Routers(gcp-vpc) - (Hybrid Connectivity > Cloud Routers)

  * Name:	gcp-vpc-cr
  * Network:	gcp-vpc
  * Region:	us-central1
  * Google ASN:	65470

* Create Routers(on-prem)

  * Name: on-prem-cr
  * Network: on-prem
  * Region: europe-west1
  * Google ASN: 65503

* Reserve static IP - 1

  * Name: gcp-vpc-ip
  * Type: Regional
  * Region: us-central1

* Reserve static IP - 2

  * Name: on-prem-ip
  * Type: Regional
  * Region: europe-west1
 
* Create the first VPN (Hybrid Connectivity > VPN)

  * Name: vpn-1
  * Network: gcp-vpc
  * Region: us-central1
  * IP address:	gcp-vpc-ip
  * Remote peer IP address: <Enter the on-prem-ip-address>
  * IKE version: IKEv2
  * Shared secret: gcprocks
  * Routing options	Dynamic (BGP)
  * Cloud router: gcp-vpc-cr
  * BGP Session
  
    * Name: bgp1to2
    * Peer ASN: 65503
    * Cloud Router BGP IP: 169.254.0.1
    * BGP peer IP: 169.254.0.2

* Create the second VPN

  * Name: vpn-2
  * Network: on-prem
  * Region: europe-west1
  * IP address: on-prem-ip
  * Remote peer IP address: <Enter the gcp-vpc-ip-address>
  * IKE version: IKEv2
  * Shared secret: gcprocks
  * Routing options	Dynamic (BGP)
  * Cloud router: on-prem-cr
  * BGP Session
  
    * Name: bgp2to1
    * Peer ASN: 65470
    * Cloud Router BGP IP: 169.254.0.2
    * BGP peer IP: 169.254.0.1


Network Monitoring
------------------


Network Logging
---------------

* VPC Subnet can be created with feature called, Log Flow.
* Log Flow goes to Monitoring > Logging.
* This Logging info can be queried through BigQuery ( It looks streaming data From GCE > Logging > BigQuery )
