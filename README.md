
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

# Configuring Istio on Azure Kubernetes Service (AKS)

This document provides step-by-step guidelines for installing and configuring Istio & Envoy on an Azure Kubernetes Service (AKS) cluster. 
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


## Authors

- [@skumars1234](https://www.github.com/skumars1234)

