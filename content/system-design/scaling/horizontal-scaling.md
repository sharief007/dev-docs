---
title: 'Horizontal Scaling'
type: docs
toc: true
sidebar:
  open: true
prev: scaling
next:
params:
  editURL:
---

Horizontal scaling, also known as scaling out, involves increasing the capacity of a system by adding more machines to a network. This approach is widely used to handle increased workloads and ensure high availability.

### Pros and Cons of Horizontal Scaling:

**Pros:**
1. **Improved Performance:** Horizontal scaling allows for the distribution of the load across multiple machines, which can significantly improve the overall performance and response times of a system.

2. **High Availability:** In the event of a hardware failure or maintenance, the system remains operational as traffic is distributed across multiple servers. This ensures high availability and reduces downtime.

3. **Cost-Efficiency:** Scaling horizontally is often more cost-effective as it involves adding commodity hardware. This can be less expensive than upgrading to more powerful, and therefore more expensive, individual machines.

**Cons:**
1. **Complexity in Data Management:** Managing data consistency across multiple machines can be more challenging in a horizontally scaled system compared to vertical scaling (scaling up).

2. **Network Overheads:** There may be additional overhead for network communication between the different nodes. This can lead to increased latency in some scenarios.

### Load Balancers:

Load balancing is a critical component of horizontal scaling.  It involves distributing incoming network traffic or requests across multiple servers to ensure that no single server becomes overwhelmed. They serve as the entry point for clients.

There are hardware-based load balancers and software-based load balancers. Hardware load balancers are dedicated devices, while software load balancers are implemented using software on standard servers.

Here are some key points about load balancing:

#### Algorithms for Load Balancing:

Load balancers use various algorithms to distribute traffic. Common ones include:
- Round Robin: Requests are distributed in a circular sequence to each server.
- Least Connections: Traffic is sent to the server with the fewest active connections.
- IP Hash: Based on the client's IP address, requests are sent to a specific server, ensuring that a client's requests always go to the same server.

#### Health Checks 

Load balancers should regularly check the health of the servers to ensure that they are responsive. Unresponsive servers should be temporarily taken out of rotation.

#### Scalability

The load balancer itself should be scalable. This can be achieved by using a combination of techniques such as DNS-based load balancing or deploying multiple load balancers in an active-passive or active-active configuration.

#### SSL Termination 

If SSL encryption is used, the load balancer can handle SSL termination to offload the decryption process from the application servers, improving performance.


### Handling Deployments:

Deploying updates in a horizontally scaled environment requires careful planning to ensure minimal downtime and service disruption.

Imagine you have a horizontally scaled e-commerce platform with multiple servers handling incoming customer requests. You've developed a new feature that enhances the user experience and want to deploy it. However, deploying updates to a live system can be tricky, especially in a horizontally scaled environment.

#### Problems

**Downtime Risk**
Deploying the update all at once could result in significant downtime. If all servers are updated simultaneously, the entire system could be offline until the deployment is complete.

**User Impact**
For instance, if a simultaneous update is performed on all servers, users accessing the system might experience disruptions or errors because all servers are undergoing changes at the same time. This can lead to a poor user experience and potential loss of sales or data.

**Risk of Widespread Impact**
Deploying a new feature directly to all users at once poses a risk. If the feature contains an unexpected bug or issue, it could lead to a widespread disruption and potentially damage the user experience.

**Rollback Complexity**
If a problem is discovered after the update, rolling back the entire deployment can be complex and time-consuming.

#### Rolling Deployments 

Rolling deployments involve updating servers one at a time, rather than all at once. This means that a portion of the servers remain operational while the others are being updated. This is known as **Gradual update**.

The update process typically follows these steps:
- **Take Node Out of Service:** The server to be updated is taken out of the rotation, meaning it no longer receives incoming requests. This can be achieved by removing it from the load balancer's pool of available servers.

- **Apply Update:** The update (which could be a new version of the software, a configuration change, or a patch) is applied to the server.

- **Health Checks:** After the update is applied, the server undergoes a series of health checks to ensure it is functioning correctly.

- **Return to Service:** Once the server passes the health checks, it is reintroduced into the pool of available servers, and it starts receiving incoming requests again.

This gives us the following benefits:
- **Reduced Downtime:** At any given time, only a fraction of the servers are out of service.

- **Continuous Availability:** The system remains operational throughout the deployment process. 

- **Easier Rollback:** If a problem is detected after a server is updated, rolling back the change is straightforward. The updated server can be taken out of rotation, and the previous version can be reintroduced.

#### Canary Releases

A canary release is a deployment strategy that allows you to test a new feature or update on a small, representative subset of your user base before rolling it out to the entire audience. 

**Limited Rollout**
Instead of deploying the new feature to all users at once, you initially release it to a small percentage of users (the "canary group"). This group represents a cross-section of your user base.

**Early Issue Detection**
Canary releases provide an early warning system. If there are any unexpected problems with the new feature, they're discovered in a controlled environment with a limited number of users.

In our e-commerce platform example, you decide to use a canary release for the new feature. You deploy it to 5% of your user base, carefully selected to represent different demographics and usage patterns. Over the next few days, you monitor user interactions and gather feedback. Everything goes smoothly, and users are happy with the new feature. With this confidence, you gradually expand the release to more users until it's eventually deployed to everyone.

#### Blue-Green Deployments 
Blue-Green deployment is a deployment strategy that involves running two identical production environments, known as "Blue" and "Green." At any given time, only one of these environments is actively serving user traffic while the other remains inactive.

**Blue Environment (Active):** The "Blue" environment is the currently active production environment that serves user traffic.
**Green Environment (Inactive):** The "Green" environment is a completely identical environment that is currently inactive and not receiving any user traffic.

When a new version or update is ready for deployment, it is deployed to the inactive "Green" environment. The team thoroughly tests and validates the new version in the "Green" environment. This includes functional testing, performance testing, and any other necessary validations. Once the new version in the "Green" environment is deemed stable and ready for production, a traffic switch is performed. This involves redirecting user traffic from the "Blue" environment to the now-active "Green" environment. If any issues are discovered after the switch, the traffic can quickly be redirected back to the "Blue" environment, ensuring a fast rollback.



In conclusion, horizontal scaling offers numerous benefits, including improved performance, high availability, and cost-efficiency. However, it also introduces complexities in data management and application design. Load balancers are crucial for evenly distributing traffic, and proper session management is essential for maintaining user state. Thoughtful deployment strategies help minimize downtime and ensure a smooth update process. Adhering to these best practices can help organizations effectively scale their systems horizontally.