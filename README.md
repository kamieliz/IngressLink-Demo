# Phase II - IngressLink

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

6. Click **Finished**.

### Section 2: Install required files

1. Clone the github repo with our example files

```other
git clone https://github.com/kamieliz/IngressLink-Demo.git
```

2. Login to the OpenShift container platform from console

```other
oc login -u f5admin -p f5admin
```

3. Change directory to project folder

```other
cd IngressLink-Demo/
```

### Section 3: Deploy the CIS Controller
Container Ingress Services (CIS) can be deployed on Kubernetes and OpenShift platform. CIS installation may differ based on the resources (for example: ConfigMap, Ingress, Routes, and CRD) used to expose the Kubernetes services. CIS Installation also depends on BIG-IP deployment and Kubernetes cluster networking. To find out more about installing CIS, check out the documentation [here](https://clouddocs.f5.com/containers/latest/userguide/cis-helm.html).

1. Create IngressLink Custom Resource definition:

```other
oc apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/customResourceDefinitions/customresourcedefinitions.yml
```

2. Create a cluster role and cluster role binding on the OpenShift cluster

```other
oc apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/rbac/clusterrole.yaml
```

3. Review the bigip address, partition, and other details in CIS deployment file

```other
nano cis/deployment-k8s-bigip-ctlr-deployment.yaml
```

```other
"—bigip-url=https://10.1.1.12:8443"

"—bigip-partition=ocp"

"—pool-member-type=nodeport"

"—custom-resource-mode=true"
```

4. Verify CIS Deployment

```other
oc get pods -n kube-system
NAME                                                       READY   STATUS    RESTARTS   AGE
k8s-bigip-ctlr-deployment-fd86c54bb-w6phz                  1/1     Running   0          41s
```


### Section 4: Customize NGINX Configuration

In phase I, we configured NGINX Ingress Controller and the following components are already installed in this lab:
   1. a namespace and a service account for Ingress Controller
   2. cluster role and cluster role binding for the IC service account
   3. a secret with a TLS certificate and a key for the default server

You can review these yaml files in the NGINX config folder and find additional documentation [here](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

1. Edit the config map created for NGINX Ingress Controller

```other
nano ~/kubernetes-ingress/deployments/common/nginx-config.yaml
```
Under the data section, add:

```other
data:
  proxy-protocol: "True"
  real-ip-header: "proxy_protocol"
  set-real-ip-from:"0.0.0.0/0"
```
Apply the changes

```other
Oc apply -f ~/kubernetes-ingress/deployments/common/nginx-config.yaml
```

2. Edit the ingress controller deployment to add ingresslink arguments

```other
Nano ~/kubernetes-ingress/deployments/deployment/nginx-ingress.yaml
```
Under the args section, add:
```other
- -ingresslink=nginx-ingress
- -report-ingress-status
```
Apply changes
```other
oc apply -f ~/kubernetes-ingress/deployments/deployment/nginx-ingress.yaml
```

3. Review previously created service for the Ingress Controller pods

```other
nano ~/1_kubernetes-ingress/deployments/service/nodeport_custom.yaml
```

4. Verify NGINX ingress deployment

```other
oc get pods -n nginx-ingress
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-744d95cb86-xk2vx   1/1     Running   0          16s
```

### Section 5: Create an IngressLink Resource

1. Add the virtual server address to the BIG-IP to ingresslink yaml file

```other
nano ingresslink.yaml
```

```other
virtualServerAddress: "10.1.1.12"
```

2. Create IngressLink resource

```other
oc create -f ingresslink.yaml
```

3. To test the integration, deploy a sample application:

```other
oc create -f ingress-example/cafe.yaml
oc create -f ingress-example/cafe-secret.yaml
oc create -f ingress-example/cafe-ingress.yaml
```

. Access the application to test traffic by running the following command

```other
$ curl --resolve cafe.example.com:443:10.1.1.12 https://cafe.example.com:443/coffee --insecure
Server address: 10.244.0.18:80
Server name: coffee-7586895968-r26zn
...
```

. Check the status of the cafe-ingress, you will see the IP of the VirtualServerAddress

```other
$ oc get ing cafe-ingress
NAME           HOSTS              ADDRESS         PORTS     AGE
cafe-ingress   cafe.example.com   10.1.1.12    80, 443   115s
```

# Resources

[Using with F5 BIG-IP Container Ingress Services | NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/f5-ingresslink/)

[OpenShift - Installing CIS manually](https://clouddocs.f5.com/containers/latest/userguide/openshift/#installing-cis-manually)

[IngressLink User Guide](https://clouddocs.f5.com/containers/latest/userguide/ingresslink/)

