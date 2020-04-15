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

