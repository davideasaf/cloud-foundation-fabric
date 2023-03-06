# Network Connectivity Center Module

This module allows the creation and management of an NCC-based hub-and-spoke architecture. It focuses in site-to-cloud connectivity with network virtual appliances (NVAs) as the _backing resource_ for spokes. This allows to connect an external network to Google Cloud by using a SD-WAN router or another appliance with BGP capabilities. It does not handle site-to-site data transfer which is not available in all regions, in particular in EMEA.

The module can manage a hub, multiple spokes, and corresponding Cloud Routers and BGP sessions to network virtual appliances. The NVAs themselves, VPCs, and other Google Cloud resources should be managed externally.

## Examples

### Connect a site to a VPC network

In this example a router appliance connects with a peer router in an on-premises network, and also peers with a Cloud Router.

<p align="center"> <img src="images/site-to-vpc.png" width="600"> </p>

```hcl
module "vpc" {
  source     = "./fabric/modules/net-vpc"
  project_id = "my-project"
  name       = "network-a"
  subnets = [
    {
      name          = "subnet-a"
      ip_cidr_range = "10.1.3.0/24"
      region        = "us-central1"
    }
  ]
}

module "nva1" {
  source     = "./fabric/modules/compute-vm"
  project_id = "my-project"
  zone       = "us-central1-a"
  name       = "router-app-a"
  network_interfaces = [{
    network    = module.vpc.self_link
    subnetwork = module.vpc.subnet_self_links["us-central1/subnet-a"]
    addresses  = { external = null, internal = "10.1.3.8" }
  }]
  can_ip_forward = true
}

module "ncc" {
  source     = "./fabric/modules/net-ncc"
  asn        = 65000
  name       = "ncc-hub"
  project_id = "my-project"
  spokes = {
    spoke-a = {
      vpc        = module.vpc.name
      region     = "us-central1"
      subnetwork = module.vpc.subnet_self_links["us-central1/subnet-a"]
      nvas = [
        {
          vm = module.nva1.self_link
          ip = module.nva1.internal_ip
        }
      ]
      router = {
        ip1      = "10.1.3.14"
        ip2      = "10.1.3.15"
        peer_asn = 65001
      }
    }
  }
}
# tftest modules=3 resources=10
```

### Connect a site to two VPC networks

In the following topology, a router appliance instance has interfaces in two VPC networks. Each interface has been used to create a Router appliance spoke.

<p align="center"> <img src="images/site-to-two-vpcs.png" width="600"> </p>

```hcl
module "vpc-a" {
  source     = "./fabric/modules/net-vpc"
  project_id = "my-project"
  name       = "network-a"
  subnets = [
    {
      name          = "subnet-a"
      ip_cidr_range = "10.1.3.0/24"
      region        = "us-central1"
    }
  ]
}

module "vpc-b" {
  source     = "./fabric/modules/net-vpc"
  project_id = "my-project"
  name       = "network-b"
  subnets = [
    {
      name          = "subnet-b"
      ip_cidr_range = "192.168.10.0/24"
      region        = "us-central1"
    }
  ]
}

module "nva1" {
  source     = "./fabric/modules/compute-vm"
  project_id = "my-project"
  zone       = "us-central1-a"
  name       = "router-app-a"
  network_interfaces = [
    {
      network    = module.vpc-a.self_link
      subnetwork = module.vpc-a.subnet_self_links["us-central1/subnet-a"]
      addresses  = { external = null, internal = "10.1.3.8" }
    },
    {
      network    = module.vpc-b.self_link
      subnetwork = module.vpc-b.subnet_self_links["us-central1/subnet-b"]
      addresses  = { external = null, internal = "192.168.10.3" }
    }
  ]
  can_ip_forward = true
}

module "ncc" {
  source     = "./fabric/modules/net-ncc"
  asn        = 65000
  name       = "ncc-hub"
  project_id = "my-project"
  spokes = {
    spoke-a = {
      vpc        = module.vpc-a.name
      region     = "us-central1"
      subnetwork = module.vpc-a.subnet_self_links["us-central1/subnet-a"]
      nvas = [
        {
          vm = module.nva1.self_link
          ip = module.nva1.internal_ips[0]
        }
      ]
      router = {
        ip1      = "10.1.3.14"
        ip2      = "10.1.3.15"
        peer_asn = 65001
      }
    },
    spoke-b = {
      vpc        = module.vpc-b.name
      region     = "us-central1"
      subnetwork = module.vpc-b.subnet_self_links["us-central1/subnet-b"]
      nvas = [
        {
          vm = module.nva1.self_link
          ip = module.nva1.internal_ips[1]
        }
      ]
      router = {
        ip1      = "192.168.10.14"
        ip2      = "192.168.10.15"
        peer_asn = 65001
      }
    }
  }
}
# tftest modules=4 resources=18
```

### Using load-balanced router appliances

The following topology shows a site that uses load-balanced router appliance instances to connect to Google Cloud. Both router appliance instances are backing resources for the same spoke.

<p align="center"> <img src="images/load-balanced-router-appliances.png" width="600"> </p>

```hcl
module "vpc" {
  source     = "./fabric/modules/net-vpc"
  project_id = "my-project"
  name       = "network-a"
  subnets = [
    {
      name          = "subnet-a-1"
      ip_cidr_range = "10.0.1.0/24"
      region        = "us-west1"
    }
  ]
}

module "nva1" {
  source     = "./fabric/modules/compute-vm"
  project_id = "my-project"
  zone       = "us-west1-a"
  name       = "router-app-a"
  network_interfaces = [{
    network    = module.vpc.self_link
    subnetwork = module.vpc.subnet_self_links["us-west1/subnet-a-1"]
    addresses  = { external = null, internal = "10.0.1.10" }
  }]
  can_ip_forward = true
}

module "nva2" {
  source     = "./fabric/modules/compute-vm"
  project_id = "my-project"
  zone       = "us-west1-b"
  name       = "router-app-b"
  network_interfaces = [{
    network    = module.vpc.self_link
    subnetwork = module.vpc.subnet_self_links["us-west1/subnet-a-1"]
    addresses  = { external = null, internal = "10.0.1.11" }
  }]
  can_ip_forward = true
}

module "ncc" {
  source     = "./fabric/modules/net-ncc"
  asn        = 65000
  name       = "ncc-hub"
  project_id = "my-project"
  spokes = {
    spoke-a = {
      vpc        = module.vpc.name
      region     = "us-west1"
      subnetwork = module.vpc.subnet_self_links["us-west1/subnet-a-1"]
      nvas = [
        {
          vm = module.nva1.self_link
          ip = module.nva1.internal_ip
        },
        {
          vm = module.nva2.self_link
          ip = module.nva2.internal_ip
        }
      ]
      router = {
        ip1      = "10.0.1.5"
        ip2      = "10.0.1.6"
        peer_asn = 65001
      }
    }
  }
}
# tftest modules=4 resources=13
```

It is possible to add custom route advertisements. For example, suppose the VPC network-a is peered to another VPC network-b using the CIDR range 10.10.0.0/24. If you want to reach this VPC network-b from the on-premises network you should advertise its range to the router appliances:

```hcl
module "vpc" {
  source     = "./fabric/modules/net-vpc"
  project_id = "my-project"
  name       = "network-a"
  subnets = [
    {
      name          = "subnet-a-1"
      ip_cidr_range = "10.0.1.0/24"
      region        = "us-west1"
    }
  ]
}

module "nva1" {
  source     = "./fabric/modules/compute-vm"
  project_id = "my-project"
  zone       = "us-west1-a"
  name       = "router-app-a"
  network_interfaces = [{
    network    = module.vpc.self_link
    subnetwork = module.vpc.subnet_self_links["us-west1/subnet-a-1"]
    addresses  = { external = null, internal = "10.0.1.10" }
  }]
  can_ip_forward = true
}

module "nva2" {
  source     = "./fabric/modules/compute-vm"
  project_id = "my-project"
  zone       = "us-west1-b"
  name       = "router-app-b"
  network_interfaces = [{
    network    = module.vpc.self_link
    subnetwork = module.vpc.subnet_self_links["us-west1/subnet-a-1"]
    addresses  = { external = null, internal = "10.0.1.11" }
  }]
  can_ip_forward = true
}

module "ncc" {
  source     = "./fabric/modules/net-ncc"
  asn        = 65000
  name       = "ncc-hub"
  project_id = "my-project"
  spokes = {
    spoke-a = {
      vpc        = module.vpc.name
      region     = "us-west1"
      subnetwork = module.vpc.subnet_self_links["us-west1/subnet-a-1"]
      nvas = [
        {
          vm = module.nva1.self_link
          ip = module.nva1.internal_ip
        },
        {
          vm = module.nva2.self_link
          ip = module.nva2.internal_ip
        }
      ]
      router = {
        custom_advertise = {
          all_subnets = true
          ip_ranges = {
            "peered-vpc-b" = "10.10.0.0/24"
          }
        }
        ip1      = "10.0.1.5"
        ip2      = "10.0.1.6"
        peer_asn = 65001
      }
    }
  }
}
# tftest modules=4 resources=13
```
<!-- BEGIN TFDOC -->

## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [asn](variables.tf#L17) | ASN for all CRs in the hub. | <code>number</code> | ✓ |  |
| [name](variables.tf#L28) | The name of the NCC hub being created. | <code>string</code> | ✓ |  |
| [project_id](variables.tf#L33) | The ID of the project where the NCC hub & spokes will be created. | <code>string</code> | ✓ |  |
| [spokes](variables.tf#L38) | List of NCC spokes. | <code title="map&#40;object&#40;&#123;&#10;  vpc        &#61; string&#10;  region     &#61; string&#10;  subnetwork &#61; string &#35; URI&#10;  nvas &#61; list&#40;object&#40;&#123;&#10;    vm &#61; string &#35; URI&#10;    ip &#61; string&#10;  &#125;&#41;&#41;&#10;  router &#61; object&#40;&#123;&#10;    custom_advertise &#61; optional&#40;object&#40;&#123;&#10;      all_subnets &#61; bool&#10;      ip_ranges   &#61; map&#40;string&#41; &#35; map of descriptions and address ranges&#10;    &#125;&#41;&#41;&#10;    ip1       &#61; string&#10;    ip2       &#61; string&#10;    keepalive &#61; optional&#40;number&#41;&#10;    peer_asn  &#61; number&#10;  &#125;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> | ✓ |  |
| [description](variables.tf#L22) | An optional description of the NCC hub. | <code>string</code> |  | <code>&#34;Terraform-managed.&#34;</code> |

<!-- END TFDOC -->