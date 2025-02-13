---
title: "Terraform on Naver Cloud - Terraform Providers: A Guide to Understanding and Development"
excerpt: "Understand Terraform Provider principles and Development Steps"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_4/

toc: true
toc_sticky: true
published: true

date: 2025-02-13
last_modified_at: 2025-02-13
---

From the 4th week onwards, we will dive into the development of Terraform providers. Before we begin actual development, let's first understand how Terraform providers work and how we can start building them.

# Terraform Provider
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FwcPCw%2FbtsIH2OV15A%2FiXOOQH5JBwx9VamP0DV5B1%2Fimg.png)
## What is Terraform?
Terraform is a provisioning tool that allows infrastructure to be defined and managed as code. However, Terraform itself does not have direct relationships with platforms like AWS or NCP. Therefore, in order to use these platforms through Terraform, we need to use a provider for the specific platform.

## Terraform Providers
A provider in Terraform is a plugin that allows Terraform to interact with various cloud services. For example, to manage resources on Naver Cloud Platform (NCP) using Terraform, a provider that interacts with NCP's API is required. This provider enables Terraform to create, modify, or delete resources by communicating with the NCP API.

Major cloud service providers, such as AWS, Azure, and Google Cloud, offer Terraform providers. Users can install these providers and include them in their Terraform configuration files to manage resources on those cloud services. With this setup, users can manage resources across multiple cloud services in a consistent manner with just a line of code.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcBxd5A%2FbtsIIkaOgp6%2FdqgRa7Aia1RCKAk12fhuzK%2Fimg.png)
## Terraform Core
Terraform Core functions as the framework of Terraform. It provides the basic functionality of Terraform, interacting with providers to manage infrastructure. Terraform Core manages the state file, generates plans, and applies them to change the actual infrastructure state.

### How Terraform Core and Providers Work Together
Terraform operates around Terraform Core, which reads user-written configuration files and uses them to manage infrastructure. Terraform Core also manages a backend, such as a local backend that checks environment variables and issues Terraform commands.

After provisioning infrastructure, Terraform creates a state file to track the state of the resources. This file records the current state of each resource, allowing changes to be tracked and managed.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fb134ra%2FbtsIIKtwCnk%2FLY39dB25UfKQW1nOiKfK71%2Fimg.png)
Terraform Core creates a dependency graph, which helps Terraform create resources in the correct order based on dependencies. Terraform communicates with providers through Remote Procedure Calls (RPC) to manage infrastructure. When a user requests the creation of a resource (e.g., a VPC), Terraform Core sends an RPC request to the provider. The provider then interacts with the cloud's API to create the requested resource.

## Example of Terraform with VPC
Let’s say a user wants to create a Virtual Private Cloud (VPC). Terraform Core would send an RPC request to the provider (e.g., NCP) to create the VPC. If the provider is AWS, Terraform Core would call AWS's SDK to interact with AWS's API and create the VPC.

Once the VPC is created, the result is sent back to Terraform Core, which updates the state file with the new resource information.

# Authentication and Resource Definitions in Terraform Providers
### Authentication in Terraform Providers
Terraform providers also handle authentication. This often involves generating access keys or inserting authentication information into environment variables, which are then used when executing Terraform commands. These authentication processes are typically handled by the SDK (Software Development Kit) of the provider.

### Defining Resources and Data Sources
In Terraform configuration files, users define the resources they want to create, such as a VPC. For instance, to create a VPC on Naver Cloud Platform (NCP), users need to include the relevant NCP resource in the configuration file.

Resources available for use in Terraform are defined in the provider. If NCP adds new resources, the provider must be updated to support those resources so that users can manage them with Terraform.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FuDwYM%2FbtsIIGEFc74%2FKhckMfcmTgaQTNVSTK5Hc0%2Fimg.png)
# gRPC: Communication Between Terraform Core and Providers
Terraform Core and providers communicate through Remote Procedure Calls (RPC), specifically using gRPC (Google Remote Procedure Call). gRPC is an open-source RPC framework developed by Google that enables efficient communication across different platforms and languages.

[Terraform Development](https://developer.hashicorp.com/terraform/plugin/terraform-plugin-protocol)
Through gRPC, Terraform Core sends requests to the provider to create resources, such as a VPC. The provider then calls the respective cloud API (e.g., NCP or AWS) to provision the resource. After the resource is created, the result is sent back to Terraform Core, which updates the state file with the new resource information.

## Terraform Validate
When running the plan and apply commands in Terraform, the system automatically performs a series of checks. The first step involves fetching the provider's schema to validate the configuration file against the provider's specifications. For example, when defining the region or VPC in the configuration, Terraform validates whether the defined parameters conform to the provider's schema.

Additionally, Terraform checks that resources and data blocks are defined in the correct order and according to the provider's resource and data block specifications. Providers can also impose constraints on values (e.g., ensuring no special characters are used).

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fod6Sq%2FbtsIIIoWZUq%2FMcKRKgVBv097NKNaGbFhlk%2Fimg.png)
# Terraform Providers: SDK vs. Framework
Previously, providers were developed using an SDK, but HashiCorp has introduced a new framework to address limitations of the SDK. The new framework provides an improved version of the SDK, offering better functionality and flexibility. A separate post will explore the differences between the SDK and the new framework in more detail.