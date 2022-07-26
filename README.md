# Phase II - IngressLink - Openshift

# Simplifying Kubernetes Ingress using F5 Technologies

The purpose of this lab guide is to demonstrate Kubernetes Ingress using IngressLink. Ingresslink provides you with modern container application workloads that use both BIG-IP Container Ingress Services and NGINX Ingress Controller for Kubernetes. This control plane solution offers a unified method of working with both technologies from a single interface offering the best of BIG-IP and NGINX.

This architecture diagram demonstrates IngressLink using NodePort.
![image](https://user-images.githubusercontent.com/4666871/180045320-126d12fb-5571-4503-b40d-c0afcc0ee6ed.png)
## IngressLink Compatibility Matrix

Minimum version to use IngressLink:

| **CIS** | **BIGIP** | **NGINX+ IC** | **AS3** |
| ------- | --------- | ------------- | ------- |
| 2.4+    | v13.1+    | 1.10+         | 3.18+   |

- Recommend AS3 version 3.25 [repo](https://github.com/F5Networks/f5-appsvcs-extension/releases/tag/v3.25.0)
- CIS 2.3 private build [repo](https://github.com/F5Networks/k8s-bigip-ctlr/releases/tag/v2.4.0)
- NGINX+ IC [repo](https://github.com/kamieliz/IngressLink-Demo/tree/main/nginx-config)
- Product Documentation [documentation](https://clouddocs.f5.com/containers/latest/userguide/ingresslink/)

## Configure F5 IngressLink with OpenShift

### Section 1: Verify the  Proxy Protocol iRule on Bigip

Proxy Protocol is required by NGINX to provide the applications PODs with the original client IPs. 

1. Login to BIG-IP GUI

```other
username: admin
password: Freiburg123
```

2. On the Main tab, click Local Traffic > iRules.
3. View the rule Proxy_Protocol_iRule and verify that the definition matches what is listed below.

```other
when SERVER_CONNECTED {
      TCP::respond "PROXY TCP[IP::version] [IP::client_addr] [clientside {IP::local_addr}] [TCP::client_port] [clientside {TCP::local_port}]\r\n"
}
```
![image](https://user-images.githubusercontent.com/4666871/181071654-54b680da-bc31-4b8f-a610-b841c3aeebb1.png)

6. Click **Update**.

### Section 2: Install required files

1. Clone the github repo with our example files

```other
git clone https://github.com/kamieliz/IngressLink-Demo.git
```

2. Login to the OpenShift container platform from console:

```other
oc login -u f5admin -p f5admin
```

3. Change directory to project folder

```other
cd IngressLink-Demo/
```

### Section 3: Deploy the CIS Controller
Container Ingress Services (CIS) can be deployed on Kubernetes and OpenShift platform. CIS installation may differ based on the resources (for example: ConfigMap, Ingress, Routes, and CRD) used to expose the Kubernetes services. CIS Installation also depends on BIG-IP deployment and Kubernetes cluster networking. To find out more about installing CIS, check out the documentation [here](https://clouddocs.f5.com/containers/latest/userguide/cis-helm.html).

1. Create IngressLink Custom Resource definition schema:

```other
oc apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/customResourceDefinitions/customresourcedefinitions.yml
```

2. Create a cluster role and cluster role binding on the OpenShift cluster. You can narrow the permissions down to specific resources, namespaces, and more to suit your needs.

```other
oc apply -f cis/openshift_rbac.yaml
```

3. For Openshift, you need to create the Cluster admin privileges for the BIG-IP service account user with the following command:

```other
oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```

4. Review the bigip address, partition, and other details in CIS deployment file. Custom resource mode needs to be set to true for IngressLink. We are also deploying in nodeport mode so we want to set our pool members to nodeport type.

```other
nano cis/deployment-k8s-bigip-ctlr-deployment.yaml
```

```other
"—bigip-url=https://10.1.1.12:8443"

"—bigip-partition=ocp"

"—pool-member-type=nodeport"

"—custom-resource-mode=true"
```

5. Verify CIS Deployment

```other
oc get pods -n kube-system
NAME                                                       READY   STATUS    RESTARTS   AGE
k8s-bigip-ctlr-deployment-fd86c54bb-w6phz                  1/1     Running   0          41s
```


### Section 4: Customize NGINX Configuration

In phase I, we configured NGINX Ingress Controller and the following components are already installed in this lab:
   - a namespace and a service account for Ingress Controller
   - cluster role and cluster role binding for the IC service account
   - a secret with a TLS certificate and a key for the default server
   - Custom resource definitions for [Virtual Server and Virtual ServerRoute](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources), [TransportServer](https://docs.nginx.com/nginx-ingress-controller/configuration/transportserver-resource), [Policy](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource) and [GlobalConfiguration](https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/globalconfiguration-resource) resources.

You can review these yaml files in the NGINX config folder and find additional documentation [here](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/) on the installation process.

1. View the config map created for NGINX Ingress Controller

```other
nano ~/3_demo/webapp_OIDC/8_nginx-config.yaml
```

2. Verify that the following is located under the data section:

```other
data:
  proxy-protocol: "True"
  real-ip-header: "proxy_protocol"
  set-real-ip-from:"0.0.0.0/0"
```

3. Apply the config map resource for customizing NGINX configuration

```other
oc apply -f ~/3_demo/webapp_OIDC/8_nginx-config.yaml
```

2. Edit the ingress controller deployment to add ingresslink arguments

```other
nano nginx-config/deployment-nginx-ingress.yaml
```
Under the args section, uncomment the following:

```other
- -ingresslink=nginx-ingress
- -report-ingress-status
```

3. Create an IngressClass resource (for Kubernetes >= 1.18):

```other
oc apply -f nginx-config/ingress-class.yaml
```
**Note**: The Ingress Controller will fail to start without an IngressClass resource

4. Review Nodeport service for the Ingress Controller pods. This service is used to access the Ingress Controller from ports 80 and 443. 

```other
nano nginx-config/service-nginx-ingress.yaml
```

5. Verify NGINX ingress deployment. When you run the Ingress Controller by using a Deployment, by default, Kubernetes will create one Ingress Controller pod.

```other
oc get pods -n nginx-ingress
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-744d95cb86-xk2vx   1/1     Running   0          16s
```

### Section 5: Create an IngressLink Resource

1. Update the `virtualServerAddress` parameter in the ingresslink.yaml resource. This IP address will be used to configure the BIG-IP device. It will be used to accept traffic and load balance it among the NGINX Ingress Controller pods. 

```other
nano ingresslink.yaml
```

```other
virtualServerAddress: "10.1.1.12"
```

**Note**: The name of the app label selector in IngressLink resource should match the labels of the nginx-ingress service from section 4.

2. Apply updates to the IngressLink resource

```other
oc apply -f ingresslink.yaml
```

3. To test the integration, deploy a sample application:

```other
oc apply -f ingress-example/cafe.yaml
```

4. Create a secret with an SSL certificate and a key:

```other
oc apply -f ingress-example/cafe-secret.yaml
```

5. Create an Ingress resource:

```other
oc apply -f ingress-example/cafe-ingress.yaml
```

4. The Ingress Controller pods are behind the IP configured in step 1. Access the coffee service to test traffic by running the following command

```other
$ curl --resolve cafe.example.com:443:10.1.1.12 https://cafe.example.com:443/coffee --insecure
Server address: 10.244.0.18:80
Server name: coffee-7586895968-r26zn
...
```

5. Access the tea service similarly:

```other
$ curl --resolve cafe.example.com:443:10.1.1.12 https://cafe.example.com:443/tea --insecure
Server address: 10.244.5.15:8080
Server name: tea-6fb46d899f-9j4zj
...
```

6. You can also access the application from the browser

![image](https://user-images.githubusercontent.com/4666871/181071419-75a27eaf-9712-4b3a-870f-c0df1d7b0767.png)


7. View the requests in the NGINX dashboard under cafe.example.com

![image](https://user-images.githubusercontent.com/4666871/181070590-656c1b4a-d23d-472c-a0d7-81365d3979da.png)

# Resources

[Using with F5 BIG-IP Container Ingress Services | NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/f5-ingresslink/)

[OpenShift - Installing CIS manually](https://clouddocs.f5.com/containers/latest/userguide/openshift/#installing-cis-manually)

[IngressLink User Guide](https://clouddocs.f5.com/containers/latest/userguide/ingresslink/)

[Installing NGINX Ingress Controller with Manifests](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

[F5 IngressLink using NodePort demo](https://www.youtube.com/watch?v=wi7vVZWHyxE&ab_channel=MarkDittmer)



