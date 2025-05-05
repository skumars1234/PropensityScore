
# Cytora - PropensityScore : Solution Architecture

This page details about the Solution Architecture of Cytora - PropensityScore.




## Architecture


<img width="1430" alt="image" src="https://github.com/user-attachments/assets/3eb82326-b317-4bdd-ad9a-884b3e2dd25b" />

**Main components in this architecture:**

•	**Cytora** : Third party service from which Allianz API receives the request.

•	**APIGEE** : Manages external API access and provides API management capabilities.

•	**AKS** Provides the Kubernetes platform for deploying and managing the microservices.

•	**Istio** : Provides the service mesh layer for managing traffic, security, and observability within the AKS cluster.

•	**Envoy**  : Acts as the data plane, handling the actual network communication between services

•	**ML Service** : The Azure ML service which actually provides the propensity score value.

**Architecture flow:**

&nbsp;1.&nbsp;	User: The user initiates a request to the microservice application.

&nbsp;2.&nbsp;	APIGEE Proxy Layer: The request first hits the APIGEE proxy layer. APIGEE acts as an API gateway, handling tasks like authentication, authorization, rate limiting, and request transformation.

&nbsp;3.&nbsp;	AKS Cluster: APIGEE then routes the request to the Azure Kubernetes Service (AKS) cluster where the microservice application is deployed.

&nbsp;4.&nbsp;	Istio Ingress Gateway: Within the AKS cluster, the request is received by the Istio Ingress Gateway. This gateway acts as the entry point for external traffic into the service mesh. It handles routing, TLS termination, and other ingress-related concerns.

&nbsp;5.&nbsp;	Microservices with Envoy Sidecar Proxies: The Ingress Gateway then routes the request to the appropriate microservice. Each microservice instance has an Envoy proxy running as a sidecar container.

&nbsp;6.&nbsp; Envoy for Internal Traffic Management: The Envoy proxies handle all communication between the microservices. They provide features like service discovery, load balancing, retries, circuit breaking, and mutual TLS (mTLS) for secure communication.

&nbsp;7.&nbsp; Response Flow: The response from the microservice follows the reverse path, going through the Envoy proxy, Istio Ingress Gateway, APIGEE, and finally back to the user.

# Configuring Istio-Envoy on Azure Kubernetes Service (AKS)

This document provides step-by-step guidelines for installing and configuring Istio & Envoy on an Azure Kubernetes Service (AKS) cluster which receives requests from an APIGEE proxy layer. 
## Prerequisites

Before you begin, ensure you have the following:

* **Azure Subscription:** An active Azure subscription.
* **AKS Cluster:** An existing and running AKS cluster. If you don't have one, you'll need to create it first.
* **`kubectl`:** Kubernetes command-line tool installed and configured to connect to your AKS cluster. You can use the Azure CLI to get the cluster credentials:
    ```bash
    az aks get-credentials --resource-group <your-resource-group> --name <your-aks-cluster-name>
    ```
* **Azure CLI:** Azure command-line interface installed.
* **`istioctl`:** Istio command-line tool. Follow the instructions on the official Istio website to download and install it: [https://istio.io/latest/docs/setup/install/istioctl/](https://istio.io/latest/docs/setup/install/istioctl/). Make sure to add `istioctl` to your system's PATH.
* **"propensity-score"** Microservice Deployed: Your "propensity-score" microservice should already be deployed in a Kubernetes namespace (let's assume it's in the propensity-ns namespace for this example).

## Step-by-Step Implementation

### 1. Download and Install Istio

* Download the latest stable Istio release:
    ```bash
    curl -L [https://istio.io/downloadIstio](https://istio.io/downloadIstio) | sh -
    ```
    This will download the Istio release into a directory named `istio-<version>`.

* Navigate to the Istio directory:
    ```bash
    cd istio-*
    ```

* Add the `istioctl` client to your PATH (it's recommended to add this to your shell configuration file):
    ```bash
    export PATH=$PWD/bin:$PATH
    ```

### 2. Install Istio Core Components on AKS

* Choose an Istio installation profile. The `default` profile is recommended for most use cases.
* Install Istio using `istioctl`:
    ```bash
    istioctl install --set profile=default -y
    ```
    This command deploys the core Istio control plane components (e.g., `istiod`) to the `istio-system` namespace.

* Verify the installation:
    ```bash
    kubectl get pods -n istio-system
    ```
    Ensure the Istio control plane pods are in a `Running` or `Completed` state.

### 3. Label the Kubernetes Namespace for Automatic Sidecar Injection

* Label the namespace where your microservices are deployed to enable automatic Envoy sidecar injection. Replace `<your-namespace>` with the actual namespace name (e.g., `default` or `my-app`).
    ```bash
    kubectl label namespace <your-namespace> istio-injection=enabled
    ```
    Example for a namespace named `my-app`:
    ```bash
    kubectl label namespace my-app istio-injection=enabled
    ```

### 4. Deploy or Redeploy Your Microservices

* If your microservices are already running, you need to **redeploy** them to inject the Envoy sidecar proxies. You can achieve this by:
    * Deleting and reapplying your deployment manifests.
    * Performing a rolling update:
        ```bash
        kubectl rollout restart deployment -n <your-namespace> <your-deployment-name>
        ```
* New microservices deployed after labeling the namespace will automatically have the Envoy sidecar injected.

### 5. Verify Sidecar Injection

* Confirm that the Envoy sidecar containers are running in your microservice pods:
    ```bash
    kubectl get pods -n <your-namespace> -o yaml | grep -o 'sidecar.istio.io/inject: "true"'
    ```
    Alternatively, inspect a specific pod's details:
    ```bash
    kubectl describe pod -n <your-namespace> <your-pod-name> | grep -A 2 Containers:
    ```
    You should see both your application container and the `istio-proxy` container listed.

### 7. Configure Istio Resources

* Start configuring Istio resources to manage traffic, security, and routing. Examples include:

    * Now that the Envoy sidecar is running with your "propensity-score" microservice, you can start configuring Istio resources to manage its traffic, security, and observability. Here are some common configurations you might need:
      * Service: Ensure you have a Kubernetes Service defined for your "propensity-score" microservice in the propensity-ns namespace. Istio uses this service registry.
       
        ``` bash
        kubectl get svc -n propensity-ns propensity-score-service
        ```
        Replace ```propensity-score-service ``` with the actual name of your service.

      * VirtualService (for Routing and Traffic Management): If you need to control how traffic is routed to different versions of your "propensity-score" service, implement traffic splitting, or define specific routing rules based on headers, etc., you'll create a virtual service.
        ``` YAML
            apiVersion: networking.istio.io/v1alpha3
            kind: VirtualService
            metadata:
                name: propensity-score-vs
                namespace: propensity-ns
            spec:
                hosts:
                - "propensity-score" # The Kubernetes service name
                http:
                - route:
                    - destination:
                        host: propensity-score
                        port:
                        number: <your-service-port> # e.g., 8080
          ```
        Apply this using ```kubectl apply -f propensity-score-vs.yaml.```

      * DestinationRule (for Load Balancing and Traffic Policies): Configure load balancing algorithms, connection pool settings, and TLS settings for traffic to your "propensity-score" service.

        ```
            apiVersion: networking.istio.io/v1alpha3
            kind: DestinationRule
            metadata:
                name: propensity-score-dr
                namespace: propensity-ns
            spec:
                host: propensity-score
                trafficPolicy:
                    loadBalancer:
                        simple: ROUND_ROBIN # Example load balancing policy
        ```

        Apply this using ```kubectl apply -f propensity-score-dr.yaml.```

### 8. Configure Istio Ingress Gateway for External Access (Traffic from Apigee):    

* You need to configure an Istio Gateway in the istio-system namespace (or your chosen gateway namespace) to define how traffic from Apigee will enter the Istio service mesh.
    * Gateway ```(apigee-ingress-gateway.yaml)```:

        ```
            apiVersion: networking.istio.io/v1alpha3
            kind: Gateway
            metadata:
                name: apigee-ingress-gateway
                namespace: istio-system
            spec:
                selector:
                    istio: ingressgateway # Use the default Istio Ingress Gateway
                servers:
                - port:
                    number: 80 # or 443 for HTTPS
                    name: http # or https
                    protocol: HTTP # or HTTPS
                  hosts:
                    - "propensity.your-apigee-domain.com" # A domain Apigee will use
                    # If using HTTPS, configure TLS here
                    # tls:
                    #   mode: SIMPLE
                    #   credentialName: apigee-cert

        ```

    Replace ```"propensity.your-apigee-domain.com" ```with a domain or hostname that you will configure in your Apigee target endpoint.

    * VirtualService for Ingress ```(propensity-vs-ingress.yaml)```:

        You'll also need a VirtualService in the namespace of your "propensity-score" service (propensity-ns in this case) to route traffic coming from the apigee-ingress-gateway to your "propensity-score" service.
        ```
        apiVersion: networking.istio.io/v1alpha3
        kind: VirtualService
        metadata:
            name: propensity-vs-ingress
            namespace: propensity-ns
        spec:
            hosts:
            - "propensity.your-apigee-domain.com" # Must match the Gateway's hosts
            gateways:
            - istio-system/apigee-ingress-gateway
            http:
            - route:
                - destination:
                    host: propensity-score # Kubernetes service name
                    port:
                    number: <your-service-port>

        ```

        Apply these Istio Ingress resources:
        ```
            kubectl apply -f apigee-ingress-gateway.yaml
            kubectl apply -f propensity-vs-ingress.yaml
        ```

### 9. Configure Apigee API Proxy Target Endpoint:

* This is the crucial change. Instead of pointing your Apigee API proxy's target endpoint directly to your "propensity-score" microservice's Kubernetes service IP or an external Load Balancer for the service, you will now point it to the external IP address or DNS name of your Istio Ingress Gateway.

    * Get the External IP of the Istio Ingress Gateway:

      ``` 
      kubectl get service -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 
      ```

      If your Ingress Gateway is exposed via a DNS name, use that instead.
    * Configure Apigee Target Endpoint:

        In your Apigee API proxy configuration, update the target endpoint URL to something like:
        ```
        http://<Istio-Ingress-Gateway-IP>:<Gateway-Server-Port>/
        ```

        or if you configured a specific hostname in the Istio Gateway:
        ```
        http://propensity.your-apigee-domain.com/
        ```

    #### Important Considerations for Apigee Target Endpoint:

    * Path: Ensure the path configured in your Apigee proxy matches the paths handled by your "propensity-score" microservice. You might need to configure path rewriting in either Apigee or the Istio VirtualService if they don't align.
    * Headers: Apigee might forward certain headers that your Istio setup or microservice expects (e.g., host header if using virtual hosts). Ensure these are correctly passed through.
    * TLS (HTTPS): If you want secure communication between Apigee and the Istio Ingress Gateway, you'll need to configure TLS on the Istio Gateway (as shown in the apigee-ingress-gateway.yaml example) and configure your Apigee target endpoint to use HTTPS and trust the certificate presented by the Istio Ingress Gateway.

### 10. Authentication and Authorization in Apigee:

* You will likely handle external-facing authentication and authorization policies within your Apigee API proxy, as you originally intended. Apigee will verify credentials before forwarding the request to the Istio Ingress Gateway. Istio can then handle further internal security policies (e.g., mTLS, fine-grained authorization within the mesh).

### 11. Monitor and Test:

* Test the entire flow: User -> Apigee Proxy -> Istio Ingress Gateway -> Envoy Sidecar -> "propensity-score" Microservice.
* Monitor both Apigee and Istio components to ensure traffic is flowing correctly and policies are being enforced.

### Summary of Changes:

* **Istio Ingress Gateway as the Entry Point:** Apigee now directs traffic to the Istio Ingress Gateway instead of directly to your microservice.
* **Istio Gateway and VirtualService for Ingress:** You need to configure these Istio resources to define how traffic from Apigee enters the service mesh and is routed to your "propensity-score" service.
* **Apigee Target Endpoint Update:** The target endpoint in your Apigee API proxy must be updated to point to the Istio Ingress Gateway's external address and the appropriate port.
* **Internal Istio Configuration Remains:** You still configure Istio resources within your microservice's namespace for internal traffic management and policies.

## Important Considerations

* **Resource Requirements:** Istio and its sidecars consume resources. Monitor your cluster's resource usage.
* **Security:** Implement RBAC policies and network policies for enhanced security.
* **Upgrades:** Follow the official Istio upgrade documentation for seamless upgrades.
* **Complexity:** Understand the concepts and complexity introduced by Istio.
* **Performance Overhead:** Be aware of potential (though generally small) latency introduced by the sidecars. Test your application performance.

This README provides a comprehensive guide to configuring Istio on your AKS cluster. 

## Authors

- [@skumars1234](https://www.github.com/skumars1234)

