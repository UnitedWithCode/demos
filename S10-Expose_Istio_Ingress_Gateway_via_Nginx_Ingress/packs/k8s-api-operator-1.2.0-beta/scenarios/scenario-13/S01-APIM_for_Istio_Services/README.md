# API Management for Istio Services

In this scenario, we have several microservices which are deployed in Istio. For applying API management for
those microservices, we can expose an API for those microservices. 

This works only in Istio permissive mode.

![Alt text](permissive-mode.png?raw=true "Istio in Permissive mode")

### Installation Prerequisites

- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

- [Kubernetes v1.14 or above](https://Kubernetes.io/docs/setup/) <br>

    - Minimum CPU : 6vCPU
    - Minimum Memory : 6GB

- [Istio v1.3.x or above](https://istio.io/docs/setup/platform-setup/)

- An account in DockerHub or private docker registry

- Download [k8s-api-operator-1.2.0-beta.zip](https://github.com/wso2/k8s-api-operator/releases/download/v1.2.0-beta/k8s-api-operator-1.2.0-beta.zip) and extract the zip

    1. This zip contains the artifacts that required to deploy in Kubernetes.
    2. Extract k8s-api-operator-1.2.0-beta.zip
    
    ```sh
    cd k8s-api-operator-1.2.0-beta/scenarios/scenario-13/S01-APIM_for_Istio_Services/
    ```
 
    **_Note:_** You need to run all commands from within the ```S01-APIM_for_Istio_Services``` directory.

<br />

#### Step 1: Configure API Controller

- Download [API controller v3.2.0-beta](https://github.com/wso2/product-apim-tooling/releases/tag/v3.2.0-beta).

- Extract the API controller distribution and navigate inside the extracted folder using the command-line tool

- Add the location of the extracted folder to your system's $PATH variable to be able to access the executable from anywhere.

- You can find available operations using the below command.
    
  ```sh
  >> apictl --help
  ```
<br />

#### Step 2: Install API Operator

- Execute the following command to install API Operator interactively and configure repository to push the microgateway image.
- Select "Docker Hub" as the repository type.
- Enter repository name of your Docker Hub account (usually it is the username as well).
- Enter username and the password
- Confirm configuration are correct with entering "Y"

    ```sh
    >> apictl install api-operator
    Choose registry type:
    1: Docker Hub
    2: Amazon ECR
    3: GCR
    4: HTTP Private Registry
    5: HTTPS Private Registry
    6: Quay.io
    Choose a number: 1: 1
    Enter repository name: jennifer
    Enter username: jennifer
    Enter password: *******
    
    Repository: jennifer
    Username  : jennifer
    Confirm configurations: Y: Y
    ```

    Output:
    ```sh
    customresourcedefinition.apiextensions.k8s.io/apis.wso2.com created
    customresourcedefinition.apiextensions.k8s.io/ratelimitings.wso2.com created
    ...
    
    namespace/wso2-system created
    deployment.apps/api-operator created
    ...
    
    [Setting to K8s Mode]
    ```
<br />

#### Step 3: Deploy Microservices

- When you execute this command, it creates a namespace called micro and enable Istio sidecar injection
  for that namespace. Also this deploys 3 microservices.

    ```sh
    >> apictl create -f microservices.yaml
    ```
   
    ```sh
    >> apictl get pods -n micro
  
    Output:
    NAME                         READY   STATUS    RESTARTS   AGE
    inventory-7dc5dfdc58-gnxqx   2/2     Running   0          9m
    products-8d478dd48-2kgdk     2/2     Running   0          9m
    review-677dd8fbd8-9ntth      2/2     Running   0          9m
    ```
<br />

#### Step 4: Deploy an API for the microservices
 
- We are creating a namespace called wso2 and deploy our API there. In this namespace, we have not enabled
  Istio sidecar injection.

    **Note:** For this sample, using the flag `--override` to update configs, if there are images in the docker registry
    which where created during older versions of API Operator.
   
   ```sh
    >> apictl create ns wso2
    >> apictl add api \
                -n online-store-api \
                --from-file=./swagger.yaml \
                --namespace=wso2 \
                --override
   ```

    ```sh
     >> apictl get pods -n wso2
  
    Output:
    NAME                                                        READY   STATUS      RESTARTS   AGE
    online-store-api-5748695f7b-jxnpf                           1/1     Running     0          14m
    online-store-api-kaniko-b5hqb                               0/1     Completed   0          14m
    ```
<br />

#### Step 5: Invoke the API
 
 - Retrieve the API service endpoint details
 
     The API service is exposed as the Load Balancer service type. You can get the API service endpoint details
     by using the following command.
 
     ```sh
     >> apictl get services -n wso2
     
     Output:
     NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                         AGE
     online-store-api       LoadBalancer   10.83.9.142    35.232.188.134   9095:31055/TCP,9090:32718/TCP   57s
     ```
 
 <details><summary>If you are using Minikube click here</summary>
 <p>
 
 **_Note:_**  By default API operator requires the LoadBalancer service type which is not supported in
 Minikube by default. Here is how you can enable it on Minikube.
 
 - On Minikube, the LoadBalancer type makes the Service accessible through the minikube service command.
 
     ```sh
     >> minikube service <SERVICE_NAME> --url
     >> minikube service online-store --url
     ```
     
     The IP you receive from above output can be used as the "external-IP" in the following command.
 
 </p>
 </details>
 
---
 
 - Invoke the API as a regular microservice
 
     Let’s observe what happens if you try to invoke the API as a regular microservice.
     ```sh
     >> curl -X GET "https://<EXTERNAL-IP>:9095/storemep/v1.0.0/products" -k
     ```
     
     You will get an error as below.
     
     ```json
     {
         "fault": {
             "code": 900902,
             "message": "Missing Credentials",
             "description": "Missing Credentials. Make sure your API invocation call has a header: \"Authorization\""
         }
     }
     ```
     
     Since the API is secured now, you are experiencing the above error. Hence you need a valid access token to invoke the API.
     
 - Invoke the API with an access token
 
     You can find a sample token below.
     
     ```sh
    TOKEN=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5UZG1aak00WkRrM05qWTBZemM1TW1abU9EZ3dNVEUzTVdZd05ERTVNV1JsWkRnNE56YzRaQT09In0.eyJhdWQiOiJodHRwOlwvXC9vcmcud3NvMi5hcGltZ3RcL2dhdGV3YXkiLCJzdWIiOiJhZG1pbkBjYXJib24uc3VwZXIiLCJhcHBsaWNhdGlvbiI6eyJvd25lciI6ImFkbWluIiwidGllciI6IjEwUGVyTWluIiwibmFtZSI6InNhbXBsZS1jcmQtYXBwbGljYXRpb24iLCJpZCI6NCwidXVpZCI6bnVsbH0sInNjb3BlIjoiYW1fYXBwbGljYXRpb25fc2NvcGUgZGVmYXVsdCIsImlzcyI6Imh0dHBzOlwvXC93c28yYXBpbTozMjAwMVwvb2F1dGgyXC90b2tlbiIsInRpZXJJbmZvIjp7fSwia2V5dHlwZSI6IlBST0RVQ1RJT04iLCJzdWJzY3JpYmVkQVBJcyI6W10sImNvbnN1bWVyS2V5IjoieF8xal83MW11dXZCb01SRjFLZnVLdThNOVVRYSIsImV4cCI6MzczMTQ5Mjg2MSwiaWF0IjoxNTg0MDA5MjE0LCJqdGkiOiJkYTA5Mjg2Yy03OGEzLTQ4YjgtYmFiNy1hYWZiYzhiMTUxNTQifQ.MKmGDwh855NrZ2wOvXO7TwFbCtsgsOFuoZW4DBVIbJ1KQ2F6TgTgBbtzBUvrYGPslEExMemhepfvvlYv8Gd6MMo3GVH4aO8AKyc8gHmeIQ8MQtXGn7u9N00ZW3_9JWaQkU-OYEDsLHvKKHzO0t2umaskSyCS2UkAS4wIT_szZ5sm-O-ez4nKGeJmESiV-1EchFjOhLpEH4p9wIj3MlKnZrIcJByRKK9ZGaHBqxwwYuJtMCDNa2wFAPMOh-45eabIUdo1KUO3gZLVcME93aza1t1jzL9mFsx0LGaXIxB7klrDuBCAdG9Yi3O7-3WUF74QaS2tmCxW36JhhOJ5DdacfQ
     ```
     Copy and paste the above token in the command line. Now you can invoke the API using the cURL command as below.
     
     ```sh
     Format: 
     
     >> curl -X GET "https://<EXTERNAL-IP>:9095/<API-context>/<API-resource>" -H "Authorization:Bearer $TOKEN" -k
     ```
 
     Example commands:
     
     ```sh
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/products" -H "Authorization:Bearer $TOKEN" -k
     
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/products/101" -H "Authorization:Bearer $TOKEN" -k
          
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/review/101" -H "Authorization:Bearer $TOKEN" -k
     
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/inventory/101" -H "Authorization:Bearer $TOKEN" -k
     ```

 <br />
