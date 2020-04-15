# Service Mesh相关topic

### 选择Service Mesh Proxy
Service mesh follows the side car design pattern where each service instance has its own sidecar proxy which handles the communication to other services. Service A does not need to be aware of the network or interconnections with other services. It will only need to know about the existence of the sidecar proxy and do the communication through that.

At a high level, service mesh can be considered as a dedicated software infrastructure for handling inter-microservice communication. The main responsibility of the service mesh is to deliver the requests from service X to service Y in a reliable, secure and timely manner. 

Service mesh is analogous to the TCP/IP network stack at a functional level. In TCP/IP, the bytes (network packets) are delivered from one computer to another computer via the underlying physical layer which consists of routers, switches and cables. It has the ability to absorb failures and make sure that messages are delivered properly. Similary, service mesh delivers requests from one microservice to another microservice on top of the service mesh network which runs on top of an unreliable microservices network.


名次复习
  - data plane 【Envoy】: Data plane touches every requests passing through the system and executes the functionalities like discovery, routing, load balancing, security and observability.
  - Control Plane 【Istio】: Control plane provides the policies and configurations for all the data planes and make them into a distributed network.
  - TLS handshake: TLS is encryption protocol designed to secure Internet communications. A TLS handshake takes place whenever a user navigates to a website over HTTPS. During a TLS handshake, the two communicating sides exchange messages to acknowledge each other, verify each other, establish the encryption algorithms they will use, and agree on session keys. TLS handshakes are a foundational part of HTTPS.

Service Mesh的好处如下

> Smarter, performant, and concurrent load balancing

> Platform and protocol agnostic routing, with HTTP and HTTP/2 (with focus on gRPC) as requirements

> Application independent routing and tracing metrics

> Traffic security


service discovery 可以告诉某一个pod，它的request可以分流给哪几个pod。service mesh proxy handles the load balancing using a list of available destinations acquired through service discovery.

两种proxy pattern
 - SideCar Pattern        : 每个pod都有一个proxy container。虽然有从resource方向来讲有overhead，但是 per-service authorization gives us the ability to maximize infrastructure security。It gives us the ability to maintain, update, and change SSL certificates for each service independent of each other.
 - DaemonSet Proxy Pattern: 一个cluster (node) 用一个daemonset agent.  From a resource allocation perspective, this also lowers the cost of running the proxies in larger clusters, significantly. This pattern makes maintaining and configuring these proxies easier by separating the lifecycle of proxies from microservices running in the same cluster.

| Pattern | Resource | lifeCycles | Network Distance | 
| ------ | ------ | ------ | ------ |
| *SideCar Pattern* | 更费机器资源，每个pod都有额外开销  | if the proxy (sidecar) needs to be updated, the entire pod must be restarted/recreated for that change to take effect| 快 |
| *DaemonSet Proxy Pattern* | 一个cluster用一个大的proxy，省资源 | updating one container won’t interrupt the others execution, and lowers the risk of inadvertent issues and downtime.| 慢，需要goes through a hostname resolution |


### Service Mesh 健康的检测和警报

In the service mesh architecture, services can discover each other using Istio, and route requests to one another using Envoy, In addition to load balancing and metrics.

最主要的是 identifying what monitoring means for each module in the system, and then the entire system as a whole. 

正常的架构是Envoy Talk to Istio, 但是访问量大了之后。可以把Istio replicated in a Kubernetes cluster。这样的话就需要加一个load balancer。每个pod去找这个 Istio的 load balancer，load balancer分发request到每个Istio pod上面去。把Istio 弄成k8s还可以有助 于better healing and monitoring。Each replica is watched by Kubernetes’ scheduler to ensure exactly N live replicas. Each replica is configured to get pinged using *Kubernetes’ health scheduler* over a short interval, e.g. every few seconds, to gauge responsiveness. If any of these checks are unsuccessful or fail to respond, the affected replica is restarted. As a result, the replica starts up fresh and re-binds with its configured control plane agent (istio agent).

几点基本要求：
> When possible, the system should self-heal.

> The monitoring system must be able to report both the internal and external health of each module in the system.

> Improve the system to self-heal as much as possible, and alert when self-healing is not applied or is not possible to implement.

首先所有的pod要被发现，Service discovery is an important part of a service mesh system, it is a crucial service in the infrastructure automation team, given how many services it discovers in each of our data centers. Any disruption in discovery can affect routing, and if the disruption is extended to minutes, it can bring down routing for all services that discover each other using service mesh. 

如果我们的整个Istio cluster都down了呢？？ 我们要确保我们可以handle all replicas are affected at once 这种情况。In such situations, we get notified about a full downtime for Istio by setting up a **higher level heartbeat**. This heartbeat is external to Kubernetes and seeks for at least one healthy and available backend behind Istio’s Kubernetes load balancer. 需要另外有一台跟k8s部署分开的server，去做一个min availability check,可以 simple HTTP ping against Istio’s load balancer in Kubernetes to ensure that at least one backend is available at all times. 这台去做min availability check的server链接了 Notification server. If consecutive execution of these heartbeats fails, an alert is sent to the notification service for the operations team to investigate further.

现在我们可以确保通过K8s Health scheduler 确保data plane with Envoy proxy 是up的, 下一步是需要他们可以 successfully route requests to proxies on other nodes in the same cluster or across clusters?
The goal of checking proxies’ running health to detect issues that can be solved by restarting the problematic container.

So we need to configure a health check that ensures a full loop through the proxy with a response code that is digestible by Kubernetes’ objects, i.e. 200 response is healthy, and non 200 responses mark containers as unhealthy。 - whether the proxies are healthy and capable of routing to different domains,

对于proxy是否可以route我们的信息到期待的目的地，可以在每个cluster加一个pod，里面有一个service叫做 prob service. 我们在独立于k8s外的服务器上，去链接这些prob service, 看是否能通过这个prob pod去与本cluster内部的pod去沟通，或者说cross datacenter的沟通。


### 随着我们service Mesh的完善，我们开始迁移更多的项目到k8s上。

我们的dev team把一些mission crital 的项目从 REST 迁移到了 gRPC 上。

| 对比 | payload | server 之间交流 | 传输速度 | HTTP Version | 
| ------ | ------ | ------ | ------ | ------ |
| Rest | Json | - For every REST call between the services, a new connection is established and there is the overhead of SSL handshake. This would cause an increase in overall latency. | JSON payloads are simple messages that have relatively slower serialization and deserialization performance. | HTTP 1.1 | 
| gRPC | Protobufs |Reuse connection, reducing the latency of creating new connections for each request  | Protobufs as payload has better serialization/deserialization into binary format. | HTTP/2| 


### CI/CD gRPC project
1. Write service and request and response definitions in the form of protobufs.
2. Validate the protobuf files before using them to generate code for implementing clients.
3. Implement the server and client(s).

Cycle如下：
1. Push protocol buffer to code repo. (pre push rule check)
2. 用 prototool 去validate这个新加入的文件 (CI stage)
3. Create release tags in the git repository from the master for the changes.
4. Fetch proto files from code repo and implement to client & server
5. 用proto gen doc去generate文档
