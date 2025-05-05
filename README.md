
# Cytora - PropensityScore : Solution Architecture

This page details about the architecture of Cytora - PropensityScore.




## Architecture

<img width="808" alt="image" src="https://github.com/user-attachments/assets/14257cf6-8efd-4c9e-a780-ad09c54269e0" />

Main components in this architecture:

•	**Cytora** Third party service from which Allianz API receives the call.

•	**APIGEE** manages external API access and provides API management capabilities.

•	**AKS** provides the Kubernetes platform for deploying and managing the microservices.

•	**Istio** provides the service mesh layer for managing traffic, security, and observability within the AKS cluster.

•	**Envoy** acts as the data plane, handling the actual network communication between services

•	**ML Service** The Azure ML service which actually provides the propensity score value.

**Architecture flow:**

&nbsp;1.&nbsp;	User: The user initiates a request to the microservice application.

&nbsp;2.&nbsp;	APIGEE Proxy Layer: The request first hits the APIGEE proxy layer. APIGEE acts as an API gateway, handling tasks like authentication, authorization, rate limiting, and request transformation.

&nbsp;3.&nbsp;	AKS Cluster: APIGEE then routes the request to the Azure Kubernetes Service (AKS) cluster where the microservice application is deployed.

&nbsp;4.&nbsp;	Istio Ingress Gateway: Within the AKS cluster, the request is received by the Istio Ingress Gateway. This gateway acts as the entry point for external traffic into the service mesh. It handles routing, TLS termination, and other ingress-related concerns.

&nbsp;5.&nbsp;	Microservices with Envoy Sidecar Proxies: The Ingress Gateway then routes the request to the appropriate microservice. Each microservice instance has an Envoy proxy running as a sidecar container.

&nbsp;6.&nbsp; Envoy for Internal Traffic Management: The Envoy proxies handle all communication between the microservices. They provide features like service discovery, load balancing, retries, circuit breaking, and mutual TLS (mTLS) for secure communication.

&nbsp;7.&nbsp; Response Flow: The response from the microservice follows the reverse path, going through the Envoy proxy, Istio Ingress Gateway, APIGEE, and finally back to the user.




## Authors

- [@skumars1234](https://www.github.com/skumars1234)

