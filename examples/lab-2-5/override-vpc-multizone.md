# VSI on VPC landing zone Deployable Architecture - Standard flavor starter

The examples shows how you can use the landing zone Deployable Architecture to configura and deploy a VPC based solution infrastructure with controlled network flows.
The resources are deployed via Cloud Projects using [VSI on VPC landing zone - Standard flavor DA](https://cloud.ibm.com/catalog/architecture/deploy-arch-ibm-slz-vsi-ef663980-4c71-4fac-af4f-4a510a9bcf68-global?format=terraform&kind=terraform&version=c0c1ed53-c224-416f-b5c1-f492058ff8c0-global) using an [override JSON](override-vpc-multizone.json) configuration.

The example deploys the following infrastructure:

- A `management` VPC with 3 subnets across the two availability zone to host a bastion, VPEs, and potentially other managment tools on VSIs. By default, the network flows are restricted for the outside access, SSH access to bastion is limited to Cloud Shell in us-south.
- A `workload` VPC with application node VSIs across two zones, incuding secondary storage volumes. The workload VSIs can only be accessed via the bastion host. Only workload VSIs have outbound access to the internet over HTTPS.

The example also creates the minimum encryption and audit infrastructure:

- A Key Protect instance and key that is used to encrypt the COS buckets VSI block storage volume.
- COS instance with buckets for audit data (VPC Flow Logs and ATracker events)
- COS instance for application data for workload instances.
- The Activity Tracker infrastructure: An Activity Tracker route to an encrypted COS bucket that stores audit events.

:exclamation: **Important:** This example shows an example of basic topology. The topology is not highly available or validated for the IBM Cloud Framework for Financial Services.

## Usage

- Add configuration using [VSI on VPC landing zone - Standard flavor DA](https://cloud.ibm.com/catalog/architecture/deploy-arch-ibm-slz-vsi-ef663980-4c71-4fac-af4f-4a510a9bcf68-global?format=terraform&kind=terraform&version=c0c1ed53-c224-416f-b5c1-f492058ff8c0-global) to a Cloud Project
- Set prefix input in the DA to `lab02`. If using a different prefix value, replace all occurences of `lab02` in the override JSON with the new value
- Enter your public SSH key value in `ssh_public_key` input in the configuration. Also select the region

## Configuration overview

<details>
<summary>Deployed landing zone diagram</summary>

![lab02-diagram]da-override-starter/landing-zone-starter-diagram.png)
  
</details>

:exclamation: **Important:** SSH access to the VSIs

The SSH access to workload VSIs is supposed to be done via the SSH jump host deployed in Management VPC. The SSH jump host is the only VSI with the floating IP, the workload VSIs are not getting floating IPs to ensure the network boundary protection for the workload compute resources.
The security group for SSH jump host only allows SSH traffic from the IP ranges in Cloud Shell environment. While this does not implement a complete lock down of the bastion hosts anyone on IBM Cloud still can reach the SSH jump host floating IPs on SSH port 22), it demonstrate the configuration needed to restrict access to the environment when public floating IPs are used for bastion / SSH jump hosts.

<details>
<summary>SSH access instructions</summary>
  
1. Make sure you are logged into an IBM Cloud account, e.g. go to https://cloud.ibm.com/resources
2. Start a Cloud Shell session with https://cloud.ibm.com/shell
3. Make sure you location at the top right is set to Dallas, change it if necessary.
4. Use **Upload file** button at the top right to upload a private SSH key matching the public key used for VSI provisioning
5. Assuming your private key was named `vpc-lab-id-rsa`, run the following commands in the Cloud Shell command prompt. Change the key file name if necessary. This will place the key in the correct directory and set the proper permissions.
```bash
mkdir -p .ssh
mv vpc-lab-id-rsa ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa
```
6. Get the SSH jump host **Floating IP** from the VSI list at https://cloud.ibm.com/infrastructure/compute/vs , also note the target Reserved IP (private address) of the VSI you are trying to access. Run the SSH command in the Cloud Shell command prompt:
```bash
ssh -J root@<SSH jump floating IP>  root@<target VSI private IP>
```
For example, for workload app node in zone 1 (public floating IP will be different):
```bash
ssh -J root@150.XX.XX.XXX  root@10.31.10.4
```

Note that the Cloud Shell instances are recycled after sessions expire, so the steps above will need to be repeated on each new Cloud Shell session.

  
</details>


### Default configuration elements
- Key Protect
  - VSI on VPC landing zone provisions KeyProtect instance `<prefix>-slz-kms` by default. The override example does not modify this configuration
  - Keys created
    - `<prefix>-atracker-key`
    - `<prefix>-slz-key`
    - `<prefix>-vsi-volume-key`
- SSH public key
  - `<prefix>-ssh-key` is created by default from the configuration input value. Override is not used for creating SSH key to minimize changes required for deployment.

## Override configuration structure
<details>
<summary>Activity Tracker</summary>

### Activity Tracker [configuration section](override-vpc-multizone.json#L2)
  - Bucket name for ATracker logs (`collector_bucket_name`) needs to match [one](override-vpc-multizone.json#L41) in the COS section.
  - Change `add_route` to `false` to skip ATracker provisioning

  <details><summary>JSON</summary>
    
```json
"atracker": {
    "collector_bucket_name": "atracker-bucket",
    "receive_global_events": true,
    "resource_group": "lab02-service-rg",
    "add_route": true
  }     
```

</details>
  
---
  
</details>

<details>
<summary>Cloud Object Storage</summary>
  
### COS [configuration section](override-vpc-multizone.json#L8)

  - Default [KMS key](override-vpc-multizone.json#L13) is used for all buckets. If a different key needs to be configrued, KMS section with key definitions should be added to the override.
  - Audit COS buckets (ATracker, Flow Logs) are set with [30 days expiration](override-vpc-multizone.json#L33).
  - Same instance [`audit-flowlogs-cos`](override-vpc-multizone.json#L63) is used for ATracker and FlowLogs buckets
  - Separate COS instance for workload data
    - Bucket workload-data with two HMAC keys for reader and writer access

  <details><summary>JSON</summary>
    
```json
  "cos": [
    {
      "buckets": [
        {
          "endpoint_type": "public",
          "kms_key": "lab02-slz-key",
          "force_delete": true,
          "name": "management-flowlogs",
          "storage_class": "standard",
          "expire_rule": {
            "rule_id": "fl-bucket-expire-30",
            "enable": true,
            "days": 30,
            "prefix": "ibm_vpc_flowlogs_v1/"
          }            
        },
        {
          "endpoint_type": "public",
          "kms_key": "lab02-slz-key",
          "force_delete": true,
          "name": "workload-flowlogs",
          "storage_class": "standard",
          "expire_rule": {
            "rule_id": "fl-bucket-expire-30",
            "enable": true,
            "days": 30,
            "prefix": "ibm_vpc_flowlogs_v1/"
          }            
        },
        {
          "endpoint_type": "public",
          "force_delete": true,
          "kms_key": "lab02-atracker-key",
          "name": "atracker-bucket",
          "storage_class": "standard",
          "expire_rule": {
            "rule_id": "a-bucket-expire-rule-30",
            "enable": true,
            "days": 30,
            "prefix": "logs/"
          }
        }        
      ],
      "keys": [
        {
          "name": "flowlogs-cos-bind-key",
          "role": "Writer",
          "enable_HMAC": true
        },
        {
          "name": "flowlogs-cos-bind-reader-key",
          "role": "Reader",
          "enable_HMAC": true
        }
      ],
      "name": "audit-flowlogs-cos",
      "plan": "standard",
      "random_suffix": true,
      "resource_group": "lab02-service-rg",
      "use_data": false
    },
    {
      "buckets": [
        {
          "endpoint_type": "public",
          "kms_key": "lab02-slz-key",
          "force_delete": true,
          "name": "workload-data",
          "storage_class": "standard"
        }
      ],
      "keys": [
        {
          "name": "workload-data-cos-bind-key",
          "role": "Writer",
          "enable_HMAC": true
        },
        {
          "name": "workload-data-cos-bind-reader-key",
          "role": "Reader",
          "enable_HMAC": true
        }
      ],
      "name": "workload-data-cos",
      "plan": "standard",
      "random_suffix": true,
      "resource_group": "lab02-workload-rg",
      "use_data": false
    }
  ]    
```
    
  </details>

---
  
</details>
<details>
<summary>Resource groups</summary>
  
### Resource groups [configuration section](override-vpc-multizone.json#L98)

  - Three new resource groups created
  - Change `create` to `false` to refer to an existing resource group. The resource group name needs to be updated throughout the override.

  <details><summary>JSON</summary>
    
```json
  "resource_groups": [
    {
      "create": true,
      "name": "lab02-workload-rg"
    },
    {
      "create": true,
      "name": "lab02-management-rg"
    },
    {
      "create": true,
      "name": "lab02-service-rg"
    }
  ]     
```

</details>
  
---
  
</details>

<details>
<summary>Security groups</summary>

### Security groups [configuration section](override-vpc-multizone.json#L112)

- Common rules in all security groups
  - Allow inbound and outbound traffic within workload and management VPCs and between them (`10.0.0.0/8` range)
  - Allow outbound to Cloud Service Endpoints (CSE) range `166.9.0.0/16`
  - Allow inbound from and outbound to IBM Cloud networking services range `161.26.0.0/16` - needed for DNS and NTP traffic
  
The access to CSE and networking services may be required for the VSIs to start up successfully.
  
- Workload VPC
  - `<prefix>-workload-vpe-sg` [rules](override-vpc-multizone.json#L114) : VPE access - common rules
  - `<prefix>-workload-vsi` [rules](override-vpc-multizone.json#L146): VSI in the workload VPC. In addition to common rules, allows SSH access from bastion subnet in Management VPC and outbound HTTPS traffic.
- Management VPC
  - `<prefix>-management-bastion` [rules](override-vpc-multizone.json#L191): jumphost access - inbound SSH access is allowed only from these sources:
    - [Inbound SSH](override-vpc-multizone.json#L200) from Workload and Management VPCs - `10.0.0.0/8` range
    - [Inbound SSH](override-vpc-multizone.json#L209) from Cloud Shell Dallas location - 3 ranges that is a larger superset of the [cloud shell IP ranges](https://cloud.ibm.com/docs/cloud-shell?topic=cloud-shell-cs-ip-ranges).
  - `<prefix>-management-c2svpn` [rules](override-vpc-multizone.json#L254): Security group that can be used for Client-to-Site VPN service. Allows incoming UDP traffic on port 443 from outside. 
  - `<prefix>-management-vpe-sg` [rules](override-vpc-multizone.json#L285): VPE access - common rules

---
  
</details>

<details>
<summary>Transit gateway</summary>

### Transit gateway [configuration section](override-vpc-multizone.json#L318)

  - Transit gateway can be skipped by setting `enable_transit_gateway` to `false`, e.g. when later connecting to another existing gateway
  - If adding more VPCs to the override configuration, they can be connected to the Transit Gateway by listing their names (without a prefix) in `transit_gateway_connections`
  - You can make the Transit Gateway instance global by adding `"transit_gateway_global": true,` to the configuration
  
  
  <details><summary>JSON</summary>
    
```json
  "enable_transit_gateway": true,
  "transit_gateway_resource_group": "lab02-workload-rg",
  "transit_gateway_global": false,
  "transit_gateway_connections": [
    "workload",
    "management"
  ],    
```

</details>
  
---
  
</details>

<details>
<summary>Virtual private endpoints</summary>

### VPE [configuration section](override-vpc-multizone.json#L324)

  This example configures two VPEs for Cloud Object Storage instances:
  - `<prefix>-audit-flowlogs-cos` in Management VPC
  - `<prefix>-workload-data-cos` in Workload VPC
  
  Note that the `service_name` attribute must match the COS instance name in corresponding section for [COS configuraton](override-vpc-multizone.json#L63).
  The name of the VPE will include prefix, VPC name and the COS instance name, e.g. `<prefix>-management-audit-flowlogs-cos`.
  
  <details><summary>JSON</summary>
    
```json
  "virtual_private_endpoints": [
    {
      "service_name": "audit-flowlogs-cos",
      "service_type": "cloud-object-storage",
      "resource_group": "lab02-management-rg",
      "vpcs": [
        {
          "name": "management",
          "security_group_name": "lab02-management-vpe-sg",
          "subnets": [
            "vpe-zone-1",
            "vpe-zone-2"
          ]
        }
      ]
    },
    {
      "service_name": "workload-data-cos",
      "service_type": "cloud-object-storage",
      "resource_group": "lab02-workload-rg",
      "vpcs": [
        {
          "name": "workload",
          "security_group_name": "lab02-workload-vpe-sg",
          "subnets": [
            "vpe-zone-1",
            "vpe-zone-2"
          ]
        }
      ]
    }    
  ],    
```

</details>
  
---
  
</details>

<details>
<summary>VPC</summary>

### VPC [configuration section](override-vpc-multizone.json#L356)

- Management [VPC](override-vpc-multizone.json#L358)
  - The VPC name is constructed from the global prefix and the `prefix` [attribute](override-vpc-multizone.json#L566).
  - Access Control Lists use a set of common rules similar to [Security Groups](#security-groups-configuration-section) for cross-VPC connections and Cloud service endpoints. Note that since ACL is stateles, traffic flow has to be explicitly allowed in both directions (i.e. outbound for request and inbount for response).
    - `management-acl` [rules](override-vpc-multizone.json#L371): common ACL, no access to or from public network outsive VPC and Cloud is allowed.
    - `management-bastion-acl` [rules](override-vpc-multizone.json#L419): common ACL rule set plus rules allowing traffic on port 22 without restriction on source. The source IP range restriction is implemented in the corresponding [security group](override-vpc-multizone.json#L209).
    - `management-c2svpn-acl` [rules](override-vpc-multizone.json#L493): common ACL plus rules allowing UDP traffic on port 443 without restriction on source. This ACL can be used for Client-to-Site VPN instance deployed to `management-vpn-zone-1` subnet.
  - [Subnets](override-vpc-multizone.json#L568)
    - `<prefix>-management-vsi-zone-1,2` : IP ranges `10.21.10.0/24` and `10.21.11.0/24` respectively, ACL `management-acl`. Could be used for deploying management tools and instrumentation besides bastions.
    - `<prefix>-management-vpe-zone-1,2` : IP ranges `10.21.20.0/24` and `10.21.21.0/24` respectively, ACL `management-acl`
    - `<prefix>-management-vpn-zone-1` : IP ranges `10.21.40.0/24`, zone 1 only, ACL `management-c2svpn-acl`, could be used for deploying Client-to-Site VPN instance
    - `<prefix>-management-bastion-zone-1` : IP ranges `10.21.30.0/24`, zone 1 only, ACL `management-bastion-acl`, dedicated subnet for bastion/jumphost VSIs.
  - No [public gateways](override-vpc-multizone.json#L611) configured for management VPC.
  
- Workload [VPC](override-vpc-multizone.json#L618)
  - The VPC name is constructed from the global prefix and the `prefix` [attribute](override-vpc-multizone.json#L704).
  - Access Control Lists use a set of common rules similar to [Security Groups](#security-groups-configuration-section) for cross-VPC connections and Cloud service endpoints. Note that since ACL is stateles, traffic flow has to be explicitly allowed in both directions (i.e. outbound for request and inbount for response).
     - `workload-acl` [rules](override-vpc-multizone.json#L631): common ACL rule set plus rules allowing outbound TCP traffic on port 443 (HTTPS) without restriction on target.
  - [Subnets](override-vpc-multizone.json#L706)
    - `<prefix>-workload-vsi-zone-1,2` : IP ranges `10.31.10.0/24` and `10.31.11.0/24` respectively, ACL `workload-acl`. These subnets host workload VSIs and allow outbound HTTPS connections via public gateways.
    - `<prefix>-workload-vpe-zone-1,2` : IP ranges `10.31.20.0/24` and `10.31.21.0/24` respectively, ACL `workload-acl`     - [Public gateways](override-vpc-multizone.json#L737) configured for zones 1 and 2 in workload VPC.  
  
The subnet IP ranges are selected to make it easier to configure network masks that cover same subnets in multiple zones (e.g. `/23`) or all subnets in a VPC (`/16`).  

---
  
</details>

<details>
<summary>VPN gateway</summary>

### VPN gateway [configuration section](override-vpc-multizone.json#L744)
  
  The VPC Landing Zone provisions a Site-to-Site VPN gateway instance, however the VPN connections have to be configured outside of DA deployment. This examples places the policy based VPN gateway in `<prefix>-management-vpn-zone-1` subnet in Management VPC.
  
  To skip VPN gateway deployment, replace the `"vpn_gateways": [ ..... ],` element with empty array `"vpn_gateways": [],`

  <details><summary>JSON</summary>
    
```json
  "vpn_gateways": [
      {
          "name": "mgmt-gateway-zone-1",
          "resource_group": "lab02-management-rg",
          "subnet_name": "vpn-zone-1",
          "vpc_name": "management",
          "mode": "policy"
      }
  ],    
```

</details>
  
---
  
</details>

<details>
<summary>VSI</summary>

### VSI [configuration section](override-vpc-multizone.json#L753)
  
  - All VSIs are deployed with SSH key provisioned from the DA input parameter `ssh_public_key`.
  - All block storage volumes are encrypted with the KeyProtect key `<prefix>-slz-key` defined in the default landing zone configuration
  - This example [uses profile](override-vpc-multizone.json#L757) `cx2-2x4`, this can be changed for each VSI group.
  - One VSI per subnet is provisioned, to increase the count, change the `vsi_per_subnet` [attribute](override-vpc-multizone.json#L805).
  - Management VPC: [bastion VSI](override-vpc-multizone.json#L755)
    - Ubuntu 24.04 image
    - Security group `<prefix>-management-bastion`
    - Floating IP enabled for SSH access
    - Deployed only in zone 1 [subnet](override-vpc-multizone.json#L767) `<prefix>-management-bastion-zone-1`. Note that in VSI configruation subnets are referenced without global prefix or VPC name.
    - Bastion VSI does not have any additional storage volumes besides boot volume.
  - Workload VPC: [application node VSI](override-vpc-multizone.json#L774)
    - RHEL 9.6 image
    - Security group `<prefix>-workload-vsi`
    - Floating IP disabled, SSH access should be done via the bastion / jumphost.
    - One VSI is deployed in zone 1 and 2 each, [subnets](override-vpc-multizone.json#L786) `<prefix>-workload-vsi-zone-1,2`. Note that in VSI configruation subnets are referenced without global prefix or VPC name.
    - Workload VSIs are provisioned with additional [storage volumes](override-vpc-multizone.json#L790) encrypted with the same KeyProtect key. Each volume size is set to 10GB but can be changed with the `capacity` [attribute](override-vpc-multizone.json#L794).

  <details><summary>JSON</summary>
    
```json
  "vsi": [
    {
      "boot_volume_encryption_key_name": "lab02-slz-key",
      "image_name": "ibm-ubuntu-24-04-3-minimal-amd64-1",
      "machine_type": "cx2-2x4",
      "name": "management-ssh-jump",
      "resource_group": "lab02-management-rg",
      "security_groups": [
        "lab02-management-bastion"
      ],
      "enable_floating_ip": true,
      "ssh_keys": [
        "ssh-key"
      ],
      "subnet_names": [
        "bastion-zone-1"
      ],
      "vpc_name": "management",
      "vsi_per_subnet": 1
    },
    {
      "boot_volume_encryption_key_name": "lab02-slz-key",
      "image_name": "ibm-redhat-9-6-minimal-amd64-3",
      "machine_type": "cx2-2x4",
      "name": "workload-app-node",
      "resource_group": "lab02-workload-rg",
      "security_groups": [
        "lab02-workload-vsi"
      ],
      "enable_floating_ip": false,
      "ssh_keys": [
        "ssh-key"
      ],
      "subnet_names": [
        "vsi-zone-1",
        "vsi-zone-2"
      ],
      "block_storage_volumes": [
        {
          "name": "app-packages",
          "profile": "general-purpose",
          "capacity": 10,
          "encryption_key": "lab02-slz-key"
        },
        {
          "name": "app-data",
          "profile": "5iops-tier",
          "capacity": 10,
          "encryption_key": "lab02-slz-key"
        }
      ],
      "vpc_name": "workload",
      "vsi_per_subnet": 1
    }
  ]    
```

</details>
  
---
  
</details>








