Author Phil Moses pmoses@redhat.com

To allow for more secure GitOps practices, sensitive data should should never be stored in Git. This presents a bit of a conundrum for GitOps, we need the secrets in Git... but we do not want to store the sensitive date. We can, however, store the secrets in a tool such as Hashicorp Vault, then retrieve and inject this data into OpenShift! A detailed how to follows, utilizing the IBM/ArgoCD-Vault plugin with ArgoCD. Keep your hands and arms inside the vehicle, buckle up and hold on, this is going to be a fun ride... or demo!

## Required
**A working OpenShift Cluster or equivalent**

If a cluster is not available, you can use Code Ready Containers from Red Hat to hone your skills.
https://developers.redhat.com/products/codeready-containers/overview

**ArgoCD**

For this demonstration, I am utilizing the OpenShift Operator Hub provided GitOps operator, which includes an implementation of ArgoCD. Also for this demonstration, I have built a custom repo-server image, a critical part of the ArgoCD stack, which will include the ArgoCD Vault Plugin. 

https://github.com/redhat-developer/gitops-operator

**An implementation of Hashicorp Vault**

A full enterprise version of Vault is out of the scope of this demo, I will utilize an dev/ephemeral implementation of Vault and configured this through the pods themselves. This is not intended for production, rather it is a quick and dirty way to have a configured Vault for a proof of concept.  https://learn.hashicorp.com/tutorials/vault/kubernetes-openshift?in=vault/kubernetes

**The IBM ArgoCD-Vault plugin** 

We will use the Kubernetes authentication method for this demo. 

https://github.com/IBM/argocd-vault-plugin

**Podman**

To build the custom container that we will use for our ArgoCD instance. 

https://podman.io/getting-started/installation 

## Hashicorp Vault Installation and configuration

We will install Vault via a Helm chart. We will be utilizing Kubernetes authentication over the other methods which are available.

```
[pmo@pmo-rhel ~]$ oc new-project vault
[pmo@pmo-rhel ~]$ helm repo add hashicorp https://helm.releases.hashicorp.com
[pmo@pmo-rhel ~]$ helm install vault hashicorp/vault --set \ "global.openshift=true" --set "server.dev.enabled=true"
```
After allowing the install to complete we should have a working dev setup of Vault:

```
[pmo@pmo-rhel ~]$ oc get pods
NAME                                        READY   STATUS    RESTARTS   AGE
vault-0                                     1/1     Running   0          22h
vault-agent-injector-56d7d5d4fd-4wp5k       1/1     Running   0          22h
```

We will quickly walk through configuring Vault. This demo is done at the pod level which is not recommended for anything outside of testing, this manner was chosen in order to quickly show a proof of concept. We will need to successfully complete the following 7 steps to have a functioning Vault environment:
* Enable Kubernetes Auth
* Write the Kubernetes config to auth/kubernetes/config; your cluster info can be retrieved from `oc cluster-info`
* Create a secret named supersecret consisting of a username and password
* Verify the secret exists
* Create a policy for the secret(s)
* Create an authentication role
* In OpenShift, create a service account 

**Enable Kubernetes Auth**
```
[pmo@pmo-rhel ~]$ oc rsh vault-0
# vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```
**Write the Kubernetes Config**
```
# vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat/var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
Success! Data written to: auth/kubernetes/config
```
**Create a secret**
```
oc rsh vault-0
# vault kv put secret/vplugin/supersecret username="pmo" \
    password="gopadres"
```
**Verify the secret exists**
```
# vault kv get secret/vplugin/supersecret
...

====== Data ======
Key         Value
---         -----
password    gopadres
username    pmo

```
**Create a policy for the secret**
```
#vault policy write webapp - <<EOF
path "secret/data/vplugin/supersecret" {
  capabilities = ["read"]
}
EOF
```
**Create an authentication role**

Note that you will need to create the SA in the namespace that you will install your ArgoCD instance. 
```
#vault write auth/kubernetes/role/vplugin \
    bound_service_account_names=vplugin \
    bound_service_account_namespaces=vplugindemo \
    policies=vplugin \
    ttl=24h
```

**Create a service account**
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vplugin
---
oc apply -f vplugin.yml
```
At this point we should have Vault installed and configure and be ready to move to ArgoCD and the plugin!

## ArgoCD Installation and Configuration
The Red Hat GitOps operator provides ArgoCD, We can install this from the Operator Hub! 

![Operatorhub listing](https://github.com/pbmoses/IBM-argocd-vault-plugin-demo/blob/main/images/operatorhub-listing.png)

With the GitOps operator installed, it’s time to build an ArgoCD instance. Out of the box Operator, the Vault plugin is not available to the operator. We can make this available in a couple of ways, an init container plus a volume mount or a custom image. I chose the latter for this demo as it will also give an idea of how to use alternate images for ArgoCD instances. 

**Create a Dockerfile**
```
FROM argoproj/argocd:latest
# Switch to root for the ability to perform install
USER root

# Install tools needed for your repo-server to retrieve & decrypt secrets, render manifests
# (e.g. curl, awscli, gpg, sops)
RUN apt-get update && \
    apt-get install -y \
        curl \
        awscli \
        gpg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install the AVP plugin (as root so we can copy to /usr/local/bin)
RUN curl -L -o argocd-vault-plugin https://github.com/IBM/argocd-vault-plugin/releases/download/v0.7.0/argocd-v
ault-plugin_0.7.0_linux_amd64
RUN chmod +x argocd-vault-plugin
RUN mv argocd-vault-plugin /usr/local/bin

# Switch back to non-root user
USER argocd
```
**Build your image and push the image to your preferred image registry**
```
podman build -t pmo-argovault:v1.0 .
podman push localhost/pmo-argovault:v1.0 quay.io/pbmoses/pmo-argovault:v1.0
```
After the image is built and pushed to the registry,we will need to build a new ArgoCD instance including our custom repo image. You can utilize the ArgoCD create GUI or apply a manifest you have available. There area few things to note which need to be included in your manifest:

*Our repo will need “mountsatoken” present and the SA we created earlier
```
...
repo:
    mountsatoken: true
    serviceaccount: webapp
```
*The image will be that which was pushed in the previous steps
```
...
image: quay.io/pbmoses/pmo-argovault
Version: v1.0
```
*Our config management plugin will need to be defined
```
...
  configManagementPlugins: |-
    - name: argocd-vault-plugin
      generate:
        command: ["argocd-vault-plugin"]
        args: ["generate", "./"]
```
A reference of a working ArgoCD manifest with the previous customizations can be found here:

[Argocd manifest](https://github.com/pbmoses/public/blob/main/argocd_manifest_custom_repo.yml)

With the ArgoCD instance installed, we can check that the plugin is indeed present:
![Plugin Present!](https://github.com/pbmoses/IBM-argocd-vault-plugin-demo/blob/main/images/installed_plugin.png)

* Out of scope of the demo but worthy to note is that you will need to have your ArgoCD RBAC properly setup as out of the box the default policy  is read only. Please see the ArgoCD RBAC info here:
https://argoproj.github.io/argo-cd/operator-manual/rbac/

**Git Secret**

Create a secret in your Git repo. The magic that takes place here is the placeholder that allows the value from Vault to be utilized. Our <> represents the key in Vault. Taking a look at our secret in Git, we see the placeholders for the secrets that live in Vault:
```
kind: Secret
apiVersion: v1
metadata:
  namespace: vplugin-demo
  name: example-secret
  annotations:
    avp_path: "secret/data/vplugin/supersecret"
type: Opaque
stringData:
  username: <username>
  password: <password>
```
**Build an Argo App**

With Vault installed and ArgoCD installed and a secret manifest in Git, we next build an application in ArgoCD and provide our plugin values via environment variables: In the end, this will look like the example below,  which points to our Git Repo which houses a sample secret. It is important to note that the environment variables are prefixed with AVP, this is a requirement of the plugin when using environment variables.

```
project: default
source:
  repoURL: 'https://github.com/pbmoses/IBM-argocd-vault-plugin-demo.git'
  path: .
  targetRevision: HEAD
  plugin:
    name: argocd-vault-plugin
    env:
      - name: AVP_K8S_ROLE
        value: vplugin
      - name: AVP_TYPE
        value: vault
      - name: AVP_VAULT_ADDR
        value: 'http://172.30.189.194:8200'
      - name: AVP_AUTH_TYPE
        value: k8s
destination:
  server: 'https://kubernetes.default.svc'
syncPolicy: {}
```

Upon successful sync of your ArgoCD, you will see the secret in ArgoCD!

![Successful sync](https://github.com/pbmoses/IBM-argocd-vault-plugin-demo/blob/main/images/successful_sync.png)

## Compare the Secrets!!

To be sure we have what is expected, lets compare the secrets:

**Secret on the cluster**
```
[pmo@pmo-rhel ~]$ oc get secret example-secret -o yaml
apiVersion: v1
data:
  password: Z29wYWRyZXM=
  username: cG1v
kind: Secret
metadata:
  annotations:
    avp_path: secret/data/vplugin/supersecret
```
**Decode the data**
```
[pmo@pmo-rhel ~]$ echo 'Z29wYWRyZXM=' | base64 --decode
gopadres
```
**review the secret in Git. Note that your secret data is not present, only placeholders!!!!**
```
kind: Secret
apiVersion: v1
metadata:
  namespace: vplugin-demo
  name: example-secret
  annotations:
    avp_path: "secret/data/vplugin/supersecret"
type: Opaque
stringData:
  username: <username>
  password: <password>
```
## Summary
We’ve used various tools in this demo. Utilizing these tools together we have successfully implemented a placeholder inside of a secret in Git allowing us to protect our sensitive data and store the actual secret in Vault, then injected the secret into OpenShift. How cool is that?!?!
