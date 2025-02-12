---
title: "Terraform on Naver Cloud - Open Source Contribution Academy 2024"
excerpt: "본문의 주요 내용을 여기에 입력하세요"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_1/

toc: true
toc_sticky: true

date: 2025-02-12
last_modified_at: 2025-02-12
---
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FqcbUW%2FbtsIsARDxv2%2FDHx3axOSkn9agKDMAwj8lK%2Fimg.webp)

Before diving into the hands-on practice, let's briefly summarize what Terraform is.

# Infrastructure as Code (IaC)

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FchYxhZ%2FbtsIsMqMdXF%2FNqhZwZmNwKQB5z1e2P3j30%2Fimg.png)

Before exploring Terraform in detail, we first need to understand the concept of Infrastructure as Code (IaC). IaC, as the name suggests, refers to defining and managing infrastructure through code. This means that instead of manually managing infrastructure through a console, we can provision and manage it using code.

# Benefits of IaC

## Cost Reduction

Traditional infrastructure provisioning involves a lot of manual work, leading to high labor costs. Additionally, fixing errors in manually configured infrastructure incurs extra costs. However, with IaC, infrastructure can be defined in code and deployed in an automated manner, reducing labor costs. Moreover, automation improves efficiency and minimizes resource waste, thereby reducing operational expenses.

## Faster Deployment

In traditional methods, setting up a new server or service requires manually navigating the console and making configurations, which can be time-consuming. However, with IaC, infrastructure can be deployed automatically by writing code, enabling quick setup of new environments and modifications to existing ones with minimal effort.

## Error Reduction

Manually setting up infrastructure introduces a high probability of human errors, such as typos, incorrect configurations, or missing settings. IaC, on the other hand, ensures that infrastructure is configured through code, reducing human errors and ensuring consistency. Additionally, code reviews and testing can help identify and fix errors in advance.

## Improved Consistency

IaC allows the same code to be used across multiple environments, ensuring consistent infrastructure deployment across development, testing, and production environments. This consistency reduces bugs and prevents issues caused by differences between environments. Additionally, IaC enables version control of infrastructure configurations, making it easy to track changes and restore previous configurations when necessary.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcVv7pf%2FbtsIrvXOQbn%2FkBqpvo2x4Bkq8ON4STySbk%2Fimg.png)
# What is Terraform?

Terraform is an open-source Infrastructure as Code (IaC) tool developed by HashiCorp. It allows developers to safely and efficiently build, manage, and version infrastructure.

## Go Language and HCL

Terraform is developed using the Go programming language and uses HCL (HashiCorp Configuration Language) to define infrastructure declaratively. Unlike procedural methods that execute API calls step by step, HCL allows resources to be defined declaratively, ensuring that multiple executions of Terraform do not result in duplicate resource creation.



# reference
- https://learn.microsoft.com/ko-kr/devops/deliver/what-is-infrastructure-as-code
- https://yozm.wishket.com/magazine/detail/2464/
- https://www.redhat.com/ko/topics/automation/what-is-infrastructure-as-code-iac