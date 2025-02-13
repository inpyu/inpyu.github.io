---
title: "Terraform on Naver Cloud - Setting Up Infrastructure in NCP Environment"
excerpt: "This lab demonstrates setting up a secure, scalable network infrastructure with VPCs, subnets, NAT Gateways, and Load Balancers in the NCP environment for high availability and security."

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_2/

toc: true
toc_sticky: true
published: true

date: 2025-02-12
last_modified_at: 2025-02-12
---

In this week’s lab, the goal is to set up the infrastructure environment as provided in the NCP (Network Cloud Platform) setup. I'll walk through the flow of the assignment and the architecture of the setup.

# Architecture
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Flceqd%2FbtsIGr9vsUv%2FKu1LgFdnjCuw8PzzfhKGOk%2Fimg.png)
The architecture for the lab consists of the following structure:
Internet Gateway: The Internet Gateway is connected to the Public Subnet, which allows direct communication with the internet.
Private Subnet: The resources in the Private Subnet communicate with the outside world via a NAT Gateway.
NAT Gateway: The NAT Gateway enables outbound internet access for instances in the Private Subnet while keeping them inaccessible from external sources.
Load Balancer: The Load Balancer distributes traffic across multiple servers to ensure load balancing and high availability.
Before we proceed with building this infrastructure, let’s review some key concepts and terms.

# Key Terms
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FD7wrL%2FbtsIHLezquZ%2F38VDbpMYP3xDWt4iNIK7Tk%2Fimg.png)

## VPC (Virtual Private Cloud)
A VPC is a service that allows users to create logically isolated networks within a cloud environment. With VPC, users can control IP address ranges, subnets, routing tables, and network gateways. In this lab, we’ll divide the network into Private and Public Subnets and control network traffic via security groups.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FebQ4Uw%2FbtsIIi36pvm%2FuPK4v0orA2RIKbe9J8wNK1%2Fimg.jpg)
## Subnet
A subnet is a smaller subdivision of a VPC’s IP address range. Subnets can be public or private. The Public Subnet communicates directly with the internet through the Internet Gateway, while the Private Subnet accesses the internet via a NAT Gateway. Subnets allow fine-grained control over network traffic and security policies.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FArDPQ%2FbtsIGVWw3qq%2Fw4y8mG4z6SA7fTvyatQiL1%2Fimg.png)
## Internet Gateway
An Internet Gateway is a router that allows communication between a VPC and the internet. It routes traffic between resources in the Public Subnet and the internet. Resources in the Public Subnet can access the internet through the Internet Gateway, while resources in the Private Subnet access the internet through a NAT Gateway.

## NAT Gateway
The NAT Gateway enables instances in the Private Subnet to access the internet, but it prevents external sources from directly accessing those instances. The NAT Gateway performs Network Address Translation (NAT), converting private IP addresses to public ones for outgoing traffic.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FdOZDfu%2FbtsIGMrSFZU%2FGaUEXvpKuOGIkak0P4dcI1%2Fimg.png)
## Load Balancer
A Load Balancer distributes incoming network traffic across multiple server instances to ensure no single server becomes overloaded. It also improves application availability and fault tolerance by redirecting traffic to healthy servers in case of failure.

# Lab Steps
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcoyAKh%2FbtsIH1uGxU1%2FbI7KJypBHhUB3h9BkOKjz0%2Fimg.png)
1. Setting Up the VPC
Navigate to Service → Networking → VPC 

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fcm2lwi%2FbtsIJd82wem%2FPzxhRHVE6KaioZErl4VJg1%2Fimg.png)
and create a new VPC with a specified IP range.

2. Creating Subnets
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbC1Dzx%2FbtsIGGL9JwH%2Fv24VJnV1SLr9yNMvfZJ1e0%2Fimg.png)
Within the VPC, create the necessary Subnets (Public and Private) based on the required architecture. These subnets will be used for resources such as servers and databases.

3. Network Access Control
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbBYfpl%2FbtsIHccE8Ht%2FC773TETRpzNpnx6xFYDQY1%2Fimg.png)
Use the default Access Control Groups (ACGs) for network access settings. 
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FGDD1E%2FbtsIInK2nzV%2FjYth07G8gEnNxD6h5PSk6K%2Fimg.png)
These default settings help manage traffic and enforce security policies.

4. Creating Servers
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbyqFHU%2FbtsIGrItPwD%2F31KObKqYz7RPYeqR3GKj20%2Fimg.png)
Go to Compute → Server to create new servers, similar to EC2 instances in AWS. Use the previously created ACL and select the appropriate Gateway (Public for the Public Subnet).
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FvIlq9%2FbtsIIHP3zJd%2FCp5l5fmKKWVkCXbdyGXCu1%2Fimg.png)
Select the KR1 availability zone for the initial setup, and KR2 for redundancy.
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FvIlq9%2FbtsIIHP3zJd%2FCp5l5fmKKWVkCXbdyGXCu1%2Fimg.png)

5. Server Access
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbiFSEc%2FbtsIG8VBRAt%2FKfinkI1NeHfX2CP3hwVhgk%2Fimg.png)
After creating the servers, log in using the administrator password and test connectivity between the Public and Private servers. Ensure that you can access the private server from the public server.

6. Setting Up the NAT Gateway
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbiFSEc%2FbtsIG8VBRAt%2FKfinkI1NeHfX2CP3hwVhgk%2Fimg.png)
As seen in the previous concept, the NAT Gateway allows outbound internet access for instances in the Private Subnet. Start by creating a new subnet for the NAT Gateway and configure it accordingly.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbvqrzS%2FbtsIHV884CJ%2FXRjF2KIZqVs3TvHSVtiojK%2Fimg.png)
After setting up the NAT Gateway, ensure the route table is updated to route traffic through the NAT Gateway for outbound access.
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FHS4E4%2FbtsIGwiAnSu%2FBj4kgtnkgWKKetV6AkGBiK%2Fimg.png)
To test, you can try pinging an external destination like Google to confirm the NAT Gateway is working.

7. Load Balancer Setup in KR2
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fc1SUwf%2FbtsIHF6zaXY%2FHSzK11gqIhNNkr3JKU7H90%2Fimg.png)
For high availability, create an additional server in the KR2 zone and replicate the setup process.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcizO53%2FbtsIHVuxxqP%2FgoFWvfGobJ7QTRMYwse5aK%2Fimg.png)
→ You can create a new subnet in KR2 just like you did before.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcizO53%2FbtsIHVuxxqP%2FgoFWvfGobJ7QTRMYwse5aK%2Fimg.png)
Since you need to perform the same steps as the previous private server, it's possible to easily replicate the existing server by creating an image.
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcSZI5v%2FbtsIHpbKsOJ%2FHvDKTL0798bJeKlrK3Seyk%2Fimg.png)
After replicating the server setup, create a Load Balancer 
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FQjhDB%2FbtsIIn5lqgk%2FdkqhFqVgcIvxb5nmtzwDH0%2Fimg.png)
and set up the necessary Target Groups to manage traffic between the two private servers.

8. Health Check Configuration
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcpzWWk%2FbtsIHDOqkDC%2F1IVYoEZgqq9LyRk0rt2XQK%2Fimg.png)
Configure health checks to ensure the Load Balancer routes traffic only to healthy servers. 
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbtwFxs%2FbtsIHa0dKjy%2FUe6uhYsqYdgRy1TVntA5yK%2Fimg.png)
Create a load balancer like the one below and check the connection information. Currently, a 503 error is displayed. Let's add an inbound rule.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbkgKAf%2FbtsIHHQLT5h%2F5MCeHnV7muYSXQabv5Tpr1%2Fimg.png)
Set the success threshold to 2 successful checks and the failure threshold accordingly.

9. Inbound Rules Configuration
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FV5Snk%2FbtsIHnkIbey%2F713cUE7NotCvjWfgaRI7g0%2Fimg.png)
To ensure proper traffic flow, modify the inbound rules of the Load Balancer to allow traffic on port 80 (HTTP).
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcwldQM%2FbtsIGFT3iaD%2FNPyvX4qltFHvs3vpQ8R9K1%2Fimg.png)
After updating the rules, test the Load Balancer by accessing the server’s application (e.g., Nginx page) through the Load Balancer’s IP.

10. Verify Load Balancer Status
Check the status of the Load Balancer’s target group to ensure it is correctly routing traffic and that all instances are healthy.

# Conclusion
This lab walk-through demonstrates how to set up a secure and scalable network infrastructure using VPCs, subnets, NAT Gateways, and Load Balancers in the NCP environment. By following the steps, we can create a robust environment where internal servers in the Private Subnet are securely accessed, and external traffic is efficiently managed by the Load Balancer. This architecture ensures high availability, security, and scalability for web applications.