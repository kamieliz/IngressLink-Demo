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

### Section 1: Create the  Proxy Protocol iRule on Bigip

Proxy Protocol is required by NGINX to provide the applications PODs with the original client IPs. Use the following steps to configure the Proxy_Protocol_iRule

1. Login to BIG-IP GUI

```other
username: admin
password: Freiburg123
```

2. On the Main tab, click Local Traffic > iRules.
3. Click **Create**.
4. In the Name field, type name as "Proxy_Protocol_iRule".
5. In the Definition field, Copy the definition from "Proxy_Protocol_iRule" from below.

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

### Section 3: Install the CIS Controller

1. Add BIG-IP credentials as Kubernetes Secrets

```other
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=Freiburg123
```

2. Create a service account for deploying CIS

```other
oc create serviceaccount bigip-ctlr -n kube-system
```

3. Create a cluster role and cluster role binding

```other
oc create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:bigip-ctlr
```

4. create CIS IngressLink Custom resource definition schema

```other
oc create -f cis/customresourcedefinition.yaml
```

5. Review the bigip address, partition, and other details in CIS deployment file

```other
nano cis/cis-deployment.yaml
```

```other
"—bigip-url=10.1.1.12:8443"

"—bigip-partition=ocp"

"—pool-member-type=nodeport"

"—custom-resource-mode=true"
```

6. Verify CIS Deployment

```other
oc get pods -n kube-system
NAME                                                       READY   STATUS    RESTARTS   AGE
k8s-bigip-ctlr-deployment-fd86c54bb-w6phz                  1/1     Running   0          41s
```

7. View CIS logs (note: CIS log level is currently set to DEBUG)

```other
oc logs -f deploy/k8s-bigip-ctlr-deployment -n kube-system | grep --color=auto -i '\[debug'
```

### Section 4: Customize NGINX Configuration

1. In phase I, we configured NGINX Ingress Controller and the following components are already installed in this lab:
   1. a namespace and a service account for Ingress Controller
   2. cluster role and cluster role binding for the IC service account
   3. a secret with a TLS certificate and a key for the default server

You can review these yaml files in the NGINX config folder

1. Create a config map for customizing NGINX configuration

```other
oc create -f nginx-config/nginx-config.yaml
```

2. Create Ingress Controller deployment. When you run the IC by using a Deployment, K8s will create one ingress controller pod

```other
oc create -f nginx-config/nginx-ingress.yaml
```

3. Create a service for the Ingress Controller pods for ports 80 and 443

```other
oc create -f nginx-config/nginx-service.yaml
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

3. Access the application to test traffic by running the following command

```other
$ curl --resolve cafe.example.com:443:10.1.1.12 https://cafe.example.com:443/coffee --insecure
Server address: 10.244.0.18:80
Server name: coffee-7586895968-r26zn
...
```

4. Check the status of the cafe-ingress, you will see the IP of the VirtualServerAddress

```other
$ oc get ing cafe-ingress
NAME           HOSTS              ADDRESS         PORTS     AGE
cafe-ingress   cafe.example.com   10.1.1.12    80, 443   115s
```

# Resources

[Using with F5 BIG-IP Container Ingress Services | NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/f5-ingresslink/)

[OpenShift - Installing CIS manually](https://clouddocs.f5.com/containers/latest/userguide/openshift/#installing-cis-manually)

[IngressLink User Guide](https://clouddocs.f5.com/containers/latest/userguide/ingresslink/)

