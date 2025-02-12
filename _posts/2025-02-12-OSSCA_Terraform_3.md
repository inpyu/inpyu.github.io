---
title: "Terraform on Naver Cloud - Migrating NCP Infrastructure to Terraform"
excerpt: "We will go over how to set up Terraform, configure providers, and define resources such as VPCs, subnets, NAT gateways, and load balancers. This migration will make managing and scaling our infrastructure easier and more automated"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_3/

toc: true
toc_sticky: true

date: 2025-02-13
last_modified_at: 2025-02-13
---

In this post, I will walk you through the process of migrating an existing infrastructure on Naver Cloud Platform (NCP) to Terraform. We will go over how to set up Terraform, configure providers, and define resources such as VPCs, subnets, NAT gateways, and load balancers. This migration will make managing and scaling our infrastructure easier and more automated.

# Introduction

As cloud infrastructure grows, it becomes increasingly difficult to manually manage resources. One of the most efficient ways to handle infrastructure management is by using Infrastructure as Code (IaC) tools like Terraform. In this blog, I’ll show you how we can migrate a pre-existing NCP infrastructure to Terraform, which will allow us to automate deployments, manage resources efficiently, and maintain consistency across environments.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fm5FCU%2FbtsIHYlujVC%2Fqh3zp3TFYGo90EFJCzoonk%2Fimg.png)

# Installing Terraform

First things first, let's set up Terraform on Ubuntu. Here are the steps to get started with Terraform installation:

```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

# Setting Up Terraform Configuration

Once Terraform is installed, the next step is to configure it. The configuration file typically consists of two main blocks: the terraform block and the provider block.

## Terraform Block Configuration

The terraform block is where we define the providers and specify the version of Terraform we want to use.

```
terraform {
  required_providers {
    ncloud = {
      source = "NaverCloudPlatform/ncloud"
      version = ">= 2.1.2"
    }
  }
  required_version = ">= 0.13"
}
```
- required_providers: Here we specify the provider we are using, which is Naver Cloud Platform (ncloud). The version specifies the minimum version of the NCP provider.
- required_version: This defines the minimum version of Terraform required for this configuration.

# Configuring the Provider

The provider block is where we configure the NCP provider to allow Terraform to interact with Naver Cloud.

```
provider "ncloud" {
  support_vpc = true
  region = "KR"
}
```
- support_vpc: Setting this to true enables support for NCP’s Virtual Private Cloud (VPC) functionality.
- region: This is where we define the region in which our infrastructure will be deployed. For example, "KR" for Korea.

# Defining Resources

Now, let's define the resources for our infrastructure, starting with a VPC, followed by subnets, NAT gateways, and load balancers.

## VPC Configuration
The first step is to create a Virtual Private Cloud (VPC).

```
resource "ncloud_vpc" "ossca_vpc" {
  name            = "ossca-vpc"
  ipv4_cidr_block = "10.0.0.0/16"
}
```
- name: The name of the VPC.
- ipv4_cidr_block: The IP range assigned to the VPC. /16 gives a larger range of IP addresses.

## Subnet Configuration
Next, we configure subnets that will be part of this VPC.

```
#Subnet
resource "ncloud_subnet" "public_subnet" {
  vpc_no         = ncloud_vpc.ossca_vpc.id
  subnet         = "10.0.1.0/24"
  zone           = "KR-1"
  network_acl_no = ncloud_vpc.ossca_vpc.default_network_acl_no
  subnet_type    = "PUBLIC"
  name           = "tf-public-1"
  usage_type     = "GEN"
}

resource "ncloud_subnet" "private_subnet_kr1" {
  vpc_no         = ncloud_vpc.ossca_vpc.id
  subnet         = "10.0.2.0/24"
  zone           = "KR-1"
  network_acl_no = ncloud_vpc.ossca_vpc.default_network_acl_no
  subnet_type    = "PRIVATE"
  name           = "tf-private-1"
  usage_type     = "GEN"
}

resource "ncloud_subnet" "private_subnet_kr2" {
  vpc_no         = ncloud_vpc.ossca_vpc.id
  subnet         = "10.0.3.0/24"
  zone           = "KR-2"
  network_acl_no = ncloud_vpc.ossca_vpc.default_network_acl_no
  subnet_type    = "PRIVATE"
  name           = "tf-private-2"
  usage_type     = "GEN"
}

resource "ncloud_subnet" "nat_subnet" {
  vpc_no         = ncloud_vpc.ossca_vpc.id
  subnet         = "10.0.4.0/24"
  zone           = "KR-1"
  network_acl_no = ncloud_vpc.ossca_vpc.default_network_acl_no
  subnet_type    = "PUBLIC"
  usage_type     = "NATGW"
}

resource "ncloud_subnet" "lb_subnet" {
  vpc_no         = ncloud_vpc.ossca_vpc.id
  subnet         = "10.0.5.0/24"
  zone           = "KR-1"
  network_acl_no = ncloud_vpc.ossca_vpc.default_network_acl_no
  subnet_type    = "PUBLIC" // PUBLIC(Public) | PRIVATE(Private)
  usage_type     = "LOADB"  // GEN(General) | LOADB(For load balancer)
}

```
Here, we define a public subnet with a /24 subnet mask, sufficient for 256 IP addresses.

- vpc_no: Specifies the VPC to which each subnet belongs. This is a mandatory element for connecting a subnet to a VPC.
- subnet: Defines the IP address range of the subnet. A /24 block provides 256 IP addresses, ensuring sufficient address space for each subnet.
- zone: Specifies the availability zone where the subnet is located. This helps distribute resources physically to enhance availability.
- network_acl_no: Assigns the default network ACL, automatically applying security settings to the subnet.
- subnet_type: Defines the subnet type to separate public and private traffic. Public subnets are suitable for resources requiring external communication, while private subnets are ideal for internal resources.
- name: Assigns a name to the subnet for easier management and identification.
- usage_type: Specifies the subnet’s purpose to manage network traffic. For example, NATGW is used for a NAT gateway, and LOADB is used for a load balancer.


## Configuring NAT Gateway and Route Tables

To allow the private subnets to access the internet, we’ll configure a NAT Gateway and link it with a route table.

```
resource "ncloud_nat_gateway" "nat_gateway" {
  vpc_no    = ncloud_vpc.ossca_vpc.id
  subnet_no = ncloud_subnet.nat_subnet.id
  zone      = "KR-1"
}
```
And a route table that directs traffic to the NAT gateway:

```
resource "ncloud_route_table" "route_table" {
  vpc_no                = ncloud_vpc.ossca_vpc.id
  supported_subnet_type = "PRIVATE"
}

resource "ncloud_route_table_association" "route_table_as" {
  route_table_no = ncloud_route_table.route_table.id
  subnet_no      = ncloud_subnet.private_subnet_kr1.id
}

resource "ncloud_route" "route" {
  route_table_no         = ncloud_route_table.route_table.id
  destination_cidr_block = "0.0.0.0/0"
  target_type            = "NATGW"
  target_name            = ncloud_nat_gateway.nat_gateway.name
  target_no              = ncloud_nat_gateway.nat_gateway.id
}
```
- vpc_no: Specifies the VPC to which the NAT gateway belongs.
- subnet_no: Specifies the subnet where the NAT gateway will be located. This must be a public subnet.
- zone: Specifies the availability zone where the NAT gateway is located to distribute resources physically.


## Setting Up Load Balancer

Finally, we configure a load balancer to distribute traffic across servers.

```
resource "ncloud_lb" "lb" {
  name = "tf-lb-test"
  network_type = "PUBLIC"
  type = "APPLICATION"
  subnet_no_list = [ ncloud_subnet.lb_subnet.subnet_no ]
}
```
The load balancer will handle HTTP/HTTPS traffic and distribute it across multiple servers.

- name: Assigns a name to the load balancer for easy identification.
- network_type: Specifies the public network type to handle external traffic.
- type: Uses the application load balancer type to process HTTP/HTTPS traffic.
- subnet_no_list: Specifies the subnets to which the load balancer belongs.

## Configuring the Load Balancer Target Group
```
resource "ncloud_lb_target_group" "ossca_tg" {
  vpc_no   = ncloud_vpc.ossca_vpc.vpc_no
  protocol = "HTTP"
  target_type = "VSVR"
  port        = 8080
  description = "for test"
  health_check {
    protocol = "HTTP"
    http_method = "GET"
    port           = 80
    url_path       = "/"
    cycle          = 30
    up_threshold   = 2
    down_threshold = 2
  }
  algorithm_type = "RR"
}
```
- vpc_no: Specifies the VPC to which the target group belongs.
- protocol: Uses the HTTP protocol to handle traffic.
- target_type: Specifies "VSVR" (Virtual Server) as the target type to distribute traffic efficiently.
- port: Defines the port on which the load balancer will receive traffic.
- health_check: Configures health checks to monitor the target servers' status. The cycle and thresholds are set to ensure stable service operation.
- algorithm_type: Uses the Round Robin (RR) algorithm to distribute traffic evenly among targets.

## Attaching Targets to the Target Group
```
resource "ncloud_lb_target_group_attachment" "test" {
  target_group_no = ncloud_lb_target_group.ossca_tg.target_group_no
  target_no_list = [ncloud_server.private_server1.instance_no,ncloud_server.private_server2.instance_no]
}
```
- target_group_no: Specifies the target group to which the attachment is being made.
- target_no_list: Lists the instances to be added as targets in the target group.

# Creating servers
## Public Server
```
resource "ncloud_server" "public_server1" {
  subnet_no                 = ncloud_subnet.public_subnet.id
  name                      = "ossca-pubserver"
  server_image_product_code = data.ncloud_server_image.server_image.product_code
  login_key_name            = ncloud_login_key.loginkey.id
}
```
## Private Server (KR-1)
```
resource "ncloud_server" "private_server1" {
  subnet_no                 = ncloud_subnet.private_subnet_kr1.id
  name                      = "ossca-priv1server"
  server_image_product_code = data.ncloud_server_image.server_image.product_code
  login_key_name            = ncloud_login_key.loginkey.id
}
```
## Private Server (KR-2)
```
resource "ncloud_server" "private_server2" {
  subnet_no                 = ncloud_subnet.private_subnet_kr2.id
  name                      = "ossca-priv2server"
  server_image_product_code = data.ncloud_server_image.server_image.product_code
  login_key_name            = ncloud_login_key.loginkey.id
}
```
- subnet_no: Specifies the subnet where the server will be placed, ensuring proper network configuration.
- name: Assigns a name to the server for easy identification and management.
- server_image_product_code: Specifies the server image product code to create a server with the desired OS and configuration.
- login_key_name: Defines the login key for SSH access to the server.

# Configuring the Login key
```
resource "ncloud_login_key" "loginkey" {
  key_name = "osscakey"
}
```
- key_name: Assigns a name to the login key, which is used for managing and identifying the SSH key for remote access.

# Assigning a Public IP to the Public Instance
```
resource "ncloud_public_ip" "public_ip" {
  server_instance_no = ncloud_server.public_server1.id
}
```
- server_instance_no: Specifies the server instance to which the public IP will be assigned, enabling external access. Public IPs are essential for resources that require communication with external networks.

# Troubleshooting: Using Environment Variables
```
provider "ncloud" {
    support_vpc = true
    region = "KR"
    access_key = '${var.NCP_ACCESS_KEY}'
    secret_key = '${var.NCP_SECRET_KEY}'
}
```
## Using TF_VAR for Environment Variables
To use environment variables in Terraform, export them in the following format:
```
export TF_VAR_NCP_ACCESS_KEY="your-access-key"
export TF_VAR_NCP_SECRET_KEY="your-secret-key"
```
Important Note:
When using environment variables in Terraform, you must declare them within a variable block. Initially, I assumed that using environment variables meant no need for a variable block, which led to some difficulties. Keep this in mind to avoid issues! 😅

# Final Result
```
{
  "version": 4,
  "terraform_version": "1.8.1",
  "serial": 92,
  "lineage": "db110ac3-cc96-c4bb-496d-4d12aba1201b",
  "outputs": {},
  "resources": [
    {
      "mode": "data",
      "type": "ncloud_server_image",
      "name": "server_image",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "base_block_storage_size": "50GB",
            "exclusion_product_code": null,
            "filter": [
              {
                "name": "product_name",
                "regex": false,
                "values": [
                  "Rocky Linux 8.8"
                ]
              }
            ],
            "id": "SW.VSVR.OS.LNX64.ROCKY.0808.B050",
            "infra_resource_detail_type_code": null,
            "infra_resource_type": "SW",
            "os_information": "Rocky Linux 8.8",
            "platform_type": "LNX64",
            "platform_type_code_list": null,
            "product_code": "SW.VSVR.OS.LNX64.ROCKY.0808.B050",
            "product_description": "Rocky Linux 8.8",
            "product_name": "Rocky Linux 8.8",
            "product_name_regex": null,
            "product_type": "LINUX"
          },
          "sensitive_attributes": []
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_lb",
      "name": "lb",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "description": null,
            "domain": "tf-lb-test-23743510-2be92e35bf02.kr.lb.naverncp.com",
            "id": "23743510",
            "idle_timeout": 60,
            "ip_list": [
              "223.130.153.68"
            ],
            "listener_no_list": [],
            "load_balancer_no": "23743510",
            "name": "tf-lb-test",
            "network_type": "PUBLIC",
            "subnet_no_list": [
              "145856"
            ],
            "throughput_type": "SMALL",
            "timeouts": null,
            "type": "APPLICATION",
            "vpc_no": "61298"
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjozNjAwMDAwMDAwMDAwLCJkZWxldGUiOjMwMDAwMDAwMDAwMCwidXBkYXRlIjo2MDAwMDAwMDAwMDB9fQ==",
          "dependencies": [
            "ncloud_subnet.lb_subnet",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_lb_target_group",
      "name": "ossca_tg",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "algorithm_type": "RR",
            "description": "for test",
            "health_check": [
              {
                "cycle": 30,
                "down_threshold": 2,
                "http_method": "GET",
                "port": 80,
                "protocol": "HTTP",
                "up_threshold": 2,
                "url_path": "/"
              }
            ],
            "id": "1489953",
            "load_balancer_instance_no": "",
            "name": "tg18f137da939",
            "port": 8080,
            "protocol": "HTTP",
            "target_group_no": "1489953",
            "target_no_list": [],
            "target_type": "VSVR",
            "use_proxy_protocol": false,
            "use_sticky_session": false,
            "vpc_no": "61298"
          },
          "sensitive_attributes": [],
          "private": "bnVsbA==",
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_lb_target_group_attachment",
      "name": "test",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "2024-04-25 04:22:33.801627357 +0000 UTC",
            "target_group_no": "1489953",
            "target_no_list": [
              "23780374",
              "23780380"
            ],
            "timeouts": null
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjozNjAwMDAwMDAwMDAwLCJkZWxldGUiOjMwMDAwMDAwMDAwMH19",
          "dependencies": [
            "data.ncloud_server_image.server_image",
            "ncloud_lb_target_group.ossca_tg",
            "ncloud_login_key.loginkey",
            "ncloud_server.private_server1",
            "ncloud_server.private_server2",
            "ncloud_subnet.private_subnet_kr1",
            "ncloud_subnet.private_subnet_kr2",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_login_key",
      "name": "loginkey",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "fingerprint": "6a:65:09:fb:ca:38:8a:9f:c5:69:ec:8f:76:d7:3b:5b",
            "id": "osscakey",
            "key_name": "osscakey",
            "private_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAslmTFcZ1WlAAKnD91DVxZunqKzYFXV6Rrb0L1DpIysKfla5R\nSizExrvducJTqVX6Bc6THTGl61aFWtAL2vr+4VFzaAGiRCOYTDkFIUrhNKaKxb9N\nR7lqnz5bR3hnM5zZwCdFUwFTHbC0V6rF7dnn2QwwCLNRSRuTpHrPabvT3+TikJAz\nQ2EQXaKC2iIZPziRyQbWfhgtRimFI3IC6/FyuwYc5cEoXw81rfAfDUHmml99WlzX\n3smHah7/4KiNy8AQoUX3A0RrfMC4C0e5rOrsTC/KuZzSVJdbm11+is82wmO00zCJ\nnWLwnVABTQElhFF1EBAYT1hPuTQhY56Al+PyXwIDAQABAoIBAFoXN15LhqIdQUgv\nFXkpmeQjit9TBXi5uZrqoNwOqRCLKXPBv1xZqvi8k28vQ3WJcaeXRub7WlW7udc6\nupJeMXv92e8SzDXhSSBPuVCs83/WFMl5Lf9qIPrZ0+ARaQhAVhpje/hG9gZMaXzT\nTfItHZmdN3JdqlTksjrmsnk1oPw6V5JgjU7t6nl3ZGwxtunmeaLNAgUi6hCLtOCU\n6KSFMdW0UKh9//CA5Zkr2zxAWVR9cIcVNPkHeOluhhNjFubSbtK37SQPPe1qPveq\ncwG+zngh+lmcrG49EeZGcWAXqEXCCIi2aPT4QjwLrQxKne6IaXTeYz/T8AXh7mQf\nz/5uV1kCgYEA9Bd7NfV3b7/p2ZkL2cpbMP/tk8dmvYRkh43fY62AH5OkpGpryrL5\n7RqGPTqvwSZB7aebS5sDNI3+9/tIjOUHW2kf1ny8yT1Y/qNkwPQRDY+5ETI+SoGa\nIbmVWvC2Y2NwnnIx9lKwWqGsz/EXIOkuxnXfLP3nGUgIuqzK1EgS8YsCgYEAuw0H\nOiq13BpW1QqhWQRW2+B4cHvmXATQ7U/WpbKX2QUjU19rim0/DGms3Knv9U02ciWD\n8FvGPy4hQad8nP/rsbEiGWlJ2uzerXWEQMDRRPN5AJEotXEQDA2/soa7vbciTXxq\nfdRxfvGvkpQE+TDjuGPcRfBsRb879o/9g+mxNP0CgYANRx23sLOfi5P/9zhSz5Qo\nVTOqP0WSd5o0WX5WYMDAdvqUywk0DIpV4IR+3itjWV5qvBxRf4wsFrFQ8gVfTLIa\nwdwugbiPRdwKdf7sFBq9Xx0VF2OWD/i/buX1/XQecfFVXSbknFjlhTfuU9ILQ0P9\nHbpXKzSgBnAbH30lEQqewwKBgFsBtbh5O05BimnQ6Du1PsVv62le/u9acIRlyduI\njxTJySwxStNo37ocWDxsehFxZcIXup/hJw1qVkfpQ1nnsjccJakTbxmTEax3dsdC\niQ7xHrhF5/aPce1Lay9jGkjtp0Tn+bALAsVutautVNYhEUqPW4azuRoeNwB5gjEC\nLHPJAoGBALqHHN6xFmvnNMgFyXZbuzDALmBKo2AJFOrIbAQopKePzvADosMSIIsI\n8eYOVrjUMyFpw3XoOPj9AfkxZjanuGwDYGxj624Na7nFU8xGDZR5rGkBHlshUKek\nfDOpNhGiKKkM/JMe+wSElTqN5lq7C5s4dubkaqIOXIzDvGupEvOj\n-----END RSA PRIVATE KEY-----"
          },
          "sensitive_attributes": [
            [
              {
                "type": "get_attr",
                "value": "private_key"
              }
            ]
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_nat_gateway",
      "name": "nat_gateway",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "description": null,
            "id": "23743493",
            "name": "ng18f1372df9c",
            "nat_gateway_no": "23743493",
            "private_ip": "10.0.4.6",
            "public_ip": "223.130.153.149",
            "public_ip_no": "23743511",
            "subnet_name": "sn18f1372b0d0",
            "subnet_no": "145859",
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "dependencies": [
            "ncloud_subnet.nat_subnet",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_public_ip",
      "name": "public_ip",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "description": "",
            "id": "23743576",
            "instance_no": "23743576",
            "internet_line_type": null,
            "kind_type": null,
            "public_ip": "101.79.10.78",
            "public_ip_no": "23743576",
            "server_instance_no": "23780377",
            "zone": null
          },
          "sensitive_attributes": [],
          "private": "bnVsbA==",
          "dependencies": [
            "data.ncloud_server_image.server_image",
            "ncloud_login_key.loginkey",
            "ncloud_server.public_server1",
            "ncloud_subnet.public_subnet",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_route",
      "name": "route",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "destination_cidr_block": "0.0.0.0/0",
            "id": "route-2827765935",
            "is_default": false,
            "route_table_no": "130035",
            "target_name": "ng18f1372df9c",
            "target_no": "23743493",
            "target_type": "NATGW",
            "vpc_no": "61298"
          },
          "sensitive_attributes": [],
          "private": "bnVsbA==",
          "dependencies": [
            "ncloud_nat_gateway.nat_gateway",
            "ncloud_route_table.route_table",
            "ncloud_subnet.nat_subnet",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_route_table",
      "name": "route_table",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "description": "",
            "id": "130035",
            "is_default": false,
            "name": "rt18f1372b0a7",
            "route_table_no": "130035",
            "supported_subnet_type": "PRIVATE",
            "vpc_no": "61298"
          },
          "sensitive_attributes": [],
          "private": "bnVsbA==",
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_route_table_association",
      "name": "route_table_as",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "130035:145855",
            "route_table_no": "130035",
            "subnet_no": "145855"
          },
          "sensitive_attributes": [],
          "private": "bnVsbA==",
          "dependencies": [
            "ncloud_route_table.route_table",
            "ncloud_subnet.private_subnet_kr1",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_server",
      "name": "private_server1",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "access_control_group_configuration_no_list": null,
            "base_block_storage_disk_detail_type": "SSD",
            "base_block_storage_disk_type": "NET",
            "base_block_storage_size": null,
            "cpu_count": 2,
            "description": "",
            "fee_system_type_code": null,
            "id": "23780374",
            "init_script_no": "",
            "instance_no": "23780374",
            "internet_line_type": null,
            "is_encrypted_base_block_storage_volume": null,
            "is_fee_charging_monitoring": null,
            "is_protect_server_termination": false,
            "login_key_name": "osscakey",
            "member_server_image_no": null,
            "memory_size": 4294967296,
            "name": "ossca-priv1server",
            "network_interface": [
              {
                "network_interface_no": "3974532",
                "order": 0,
                "private_ip": "10.0.2.6",
                "subnet_no": "145855"
              }
            ],
            "placement_group_no": "",
            "platform_type": "LNX64",
            "port_forwarding_external_port": null,
            "port_forwarding_internal_port": null,
            "port_forwarding_public_ip": null,
            "private_ip": null,
            "public_ip": "",
            "raid_type_name": null,
            "region": null,
            "server_image_name": null,
            "server_image_product_code": "SW.VSVR.OS.LNX64.ROCKY.0808.B050",
            "server_product_code": "SVR.VSVR.HICPU.C002.M004.NET.SSD.B050.G002",
            "subnet_no": "145855",
            "tag_list": [],
            "timeouts": null,
            "user_data": null,
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjozNjAwMDAwMDAwMDAwLCJkZWxldGUiOjMwMDAwMDAwMDAwMH19",
          "dependencies": [
            "data.ncloud_server_image.server_image",
            "ncloud_login_key.loginkey",
            "ncloud_subnet.private_subnet_kr1",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_server",
      "name": "private_server2",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "access_control_group_configuration_no_list": null,
            "base_block_storage_disk_detail_type": "SSD",
            "base_block_storage_disk_type": "NET",
            "base_block_storage_size": null,
            "cpu_count": 2,
            "description": "",
            "fee_system_type_code": null,
            "id": "23780380",
            "init_script_no": "",
            "instance_no": "23780380",
            "internet_line_type": null,
            "is_encrypted_base_block_storage_volume": null,
            "is_fee_charging_monitoring": null,
            "is_protect_server_termination": false,
            "login_key_name": "osscakey",
            "member_server_image_no": null,
            "memory_size": 4294967296,
            "name": "ossca-priv2server",
            "network_interface": [
              {
                "network_interface_no": "3974534",
                "order": 0,
                "private_ip": "10.0.3.6",
                "subnet_no": "145857"
              }
            ],
            "placement_group_no": "",
            "platform_type": "LNX64",
            "port_forwarding_external_port": null,
            "port_forwarding_internal_port": null,
            "port_forwarding_public_ip": null,
            "private_ip": null,
            "public_ip": "",
            "raid_type_name": null,
            "region": null,
            "server_image_name": null,
            "server_image_product_code": "SW.VSVR.OS.LNX64.ROCKY.0808.B050",
            "server_product_code": "SVR.VSVR.HICPU.C002.M004.NET.SSD.B050.G002",
            "subnet_no": "145857",
            "tag_list": [],
            "timeouts": null,
            "user_data": null,
            "vpc_no": "61298",
            "zone": "KR-2"
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjozNjAwMDAwMDAwMDAwLCJkZWxldGUiOjMwMDAwMDAwMDAwMH19",
          "dependencies": [
            "data.ncloud_server_image.server_image",
            "ncloud_login_key.loginkey",
            "ncloud_subnet.private_subnet_kr2",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_server",
      "name": "public_server1",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "access_control_group_configuration_no_list": null,
            "base_block_storage_disk_detail_type": "SSD",
            "base_block_storage_disk_type": "NET",
            "base_block_storage_size": null,
            "cpu_count": 2,
            "description": "",
            "fee_system_type_code": null,
            "id": "23780377",
            "init_script_no": "",
            "instance_no": "23780377",
            "internet_line_type": null,
            "is_encrypted_base_block_storage_volume": null,
            "is_fee_charging_monitoring": null,
            "is_protect_server_termination": false,
            "login_key_name": "osscakey",
            "member_server_image_no": null,
            "memory_size": 4294967296,
            "name": "ossca-pubserver",
            "network_interface": [
              {
                "network_interface_no": "3974533",
                "order": 0,
                "private_ip": "10.0.1.6",
                "subnet_no": "145858"
              }
            ],
            "placement_group_no": "",
            "platform_type": "LNX64",
            "port_forwarding_external_port": null,
            "port_forwarding_internal_port": null,
            "port_forwarding_public_ip": null,
            "private_ip": null,
            "public_ip": "101.79.10.78",
            "raid_type_name": null,
            "region": null,
            "server_image_name": null,
            "server_image_product_code": "SW.VSVR.OS.LNX64.ROCKY.0808.B050",
            "server_product_code": "SVR.VSVR.HICPU.C002.M004.NET.SSD.B050.G002",
            "subnet_no": "145858",
            "tag_list": [],
            "timeouts": null,
            "user_data": null,
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjozNjAwMDAwMDAwMDAwLCJkZWxldGUiOjMwMDAwMDAwMDAwMH19",
          "dependencies": [
            "data.ncloud_server_image.server_image",
            "ncloud_login_key.loginkey",
            "ncloud_subnet.public_subnet",
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_subnet",
      "name": "lb_subnet",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "145856",
            "name": "sn18f1372b0c2",
            "network_acl_no": "92652",
            "subnet": "10.0.5.0/24",
            "subnet_no": "145856",
            "subnet_type": "PUBLIC",
            "usage_type": "LOADB",
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_subnet",
      "name": "nat_subnet",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "145859",
            "name": "sn18f1372b0d0",
            "network_acl_no": "92652",
            "subnet": "10.0.4.0/24",
            "subnet_no": "145859",
            "subnet_type": "PUBLIC",
            "usage_type": "NATGW",
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_subnet",
      "name": "private_subnet_kr1",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "145855",
            "name": "tf-private-1",
            "network_acl_no": "92652",
            "subnet": "10.0.2.0/24",
            "subnet_no": "145855",
            "subnet_type": "PRIVATE",
            "usage_type": "GEN",
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_subnet",
      "name": "private_subnet_kr2",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "145857",
            "name": "tf-private-2",
            "network_acl_no": "92652",
            "subnet": "10.0.3.0/24",
            "subnet_no": "145857",
            "subnet_type": "PRIVATE",
            "usage_type": "GEN",
            "vpc_no": "61298",
            "zone": "KR-2"
          },
          "sensitive_attributes": [],
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_subnet",
      "name": "public_subnet",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "145858",
            "name": "tf-public-1",
            "network_acl_no": "92652",
            "subnet": "10.0.1.0/24",
            "subnet_no": "145858",
            "subnet_type": "PUBLIC",
            "usage_type": "GEN",
            "vpc_no": "61298",
            "zone": "KR-1"
          },
          "sensitive_attributes": [],
          "dependencies": [
            "ncloud_vpc.ossca_vpc"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "ncloud_vpc",
      "name": "ossca_vpc",
      "provider": "provider[\"registry.terraform.io/navercloudplatform/ncloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "default_access_control_group_no": "172647",
            "default_network_acl_no": "92652",
            "default_private_route_table_no": "130034",
            "default_public_route_table_no": "130033",
            "id": "61298",
            "ipv4_cidr_block": "10.0.0.0/16",
            "name": "ossca-vpc",
            "vpc_no": "61298"
          },
          "sensitive_attributes": []
        }
      ]
    }
  ],
  "check_results": null
}
```

Done!