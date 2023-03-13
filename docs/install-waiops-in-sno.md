<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Install WAIOps (Watson AIOps) AI Manager on SNO (Single Node OCP)](#install-waiops-watson-aiops-ai-manager-on-sno-single-node-ocp)
  - [Option 1: install per IBM doc](#option-1-install-per-ibm-doc)
  - [Option 2: install via gitops](#option-2-install-via-gitops)
    - [Preparation](#preparation)
      - [Prerequisite](#prerequisite)
      - [Set pods capacity in single node](#set-pods-capacity-in-single-node)
        - [Config](#config)
        - [Verify after cluster restarted](#verify-after-cluster-restarted)
    - [Install CP4WAIOps from UI](#install-cp4waiops-from-ui)
      - [Login to Argo CD](#login-to-argo-cd)
      - [Grant Argo CD Cluster Admin Permission](#grant-argo-cd-cluster-admin-permission)
      - [Obtain an entitlement key](#obtain-an-entitlement-key)
      - [Update the OCP global pull secret](#update-the-ocp-global-pull-secret)
      - [Storage Considerations](#storage-considerations)
      - [Install CP4WAIOps with Custom Size Using All-in-One Configuration](#install-cp4waiops-with-custom-size-using-all-in-one-configuration)
        - [Define application](#define-application)
        - [Set SNO flag](#set-sno-flag)
        - [Set if other storageclass](#set-if-other-storageclass)
        - [Override PARAMETERS](#override-parameters)
      - [Install Custom Build](#install-custom-build)
      - [Wourkaround pending redis pod](#wourkaround-pending-redis-pod)
    - [Verify installation](#verify-installation)
    - [Access CP4WAIOps](#access-cp4waiops)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Install WAIOps (Watson AIOps) AI Manager on SNO (Single Node OCP)

For how to install OpenShift on a single node, please refer to https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-installing-sno.html.

In SNO (Single Node OCP) cluster, there is **only one node** acting as both server and worker node role, so it is a little diffrent from normal multi-node OCP cluster, such as redis cluster. To install WAIOps AI Manager on SNO, some special config and workaround steps are needed. 




## Option 1: install per IBM doc
To prepare and install a deployment of IBM Cloud Pak® for Watson AIOps AI Manager, please refer to https://www.ibm.com/docs/en/cloud-paks/cloud-pak-watson-aiops/3.4.0?topic=installing.


## Option 2: install via gitops

### Preparation

#### Prerequisite
For Prerequisite, please refer to [Prerequisite](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops-36.md#prerequisites)

#### Set pods capacity in single node

By default, the pod capacity is about 250 in each worker node, indicating that at most 250 pods can be deployed in each worker node. In SNO, since there is only one node acting as both server and worker on which all pods are deployed, the default pod capacity must be increased. And this can be done via applying  `kubeletconfig` config.


##### Config 

Save below yaml into file `set-max-pods.yaml` :

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  annotations:
  name: set-max-pods
spec:
  kubeletConfig:
    maxPods: 360
    podsPerCore: 15
  machineConfigPoolSelector:
    matchLabels:
      node-role.kubernetes.io/master: ""
```

Apply it and then **restart the SNO cluster** to make it effect.
```sh
oc apply -f set-max-pods.yaml
```

 #####  Verify after cluster restarted
  node for `pods` in `Capacity` updated:

```console
$ oc get node <sno-node-name> -o yaml | yq .status.capacity.pods
360

```

### Install CP4WAIOps from UI

#### Login to Argo CD

To Login to Argo CD,  please refer to [Login to Argo CD](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops.md#login-to-argo-cd)

#### Grant Argo CD Cluster Admin Permission

To Grant Argo CD Cluster Admin Permission,  please refer to [Grant Argo CD Cluster Admin Permission](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops-36.md#grant-argo-cd-cluster-admin-permission)



#### Obtain an entitlement key
To Obtain an entitlement key, please refer to [Obtain an entitlement key](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops-36.md#obtain-an-entitlement-key)

#### Update the OCP global pull secret
To Update the OCP global pull secret, please refer to [Update the OCP global pull secret](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops-36.md#update-the-openshift-container-platform-global-pull-secret)


#### Storage Considerations

During installation, Storageclasses must be installed and the default Storageclasses must be set.   
In All-in-One Configuration installation, Ceph will be installed and `rook-cephfs` will be set as default Storageclasses by default.   
If your OpenShift cluster already have default storageclass configured, you can disable Ceph install and config your own storageclass in All-in-One Configuration. [Set if other storageclass](https://github.ibm.com/lihongbj/study-notes/blob/main/install-waiops-in-sno.md#set-if-other-storageclass)


#### Install CP4WAIOps with Custom Size Using All-in-One Configuration

The all-in-one configuration allows you to install following components in one go:

- Ceph storage (optional)
- AI Manager
- Event Manager

For AI Manager, you can specify the installation profile with the values that are officially supported for production use:
- large
- small

And in All-in-One, you can even specify the installation ***extra small*** profile, which is sub-profile requesting much less cpu and memory, only for demo, PoC, or dev environment in a resource (cpu and memory) limited cluster, below sub-profile are supported:
  - x-small
  - x-small-idle
  - x-small-custom
  
##### Define application 
Just fill in the form using the suggested field values listed in following table when you create the Argo CD App:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | anyname (e.g. waiops-sno)                          |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Repository URL        | https://github.com/IBM/cp4waiops-gitops               |
| Revision              | release-3.4                                           |
| Path                  | config/all-in-one                                     |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | openshift-gitops                                      |

##### Set SNO flag
And adding following YAML snippet to `HELM` > `VALUES` field :
```yaml
cp4waiops:
  isSNO: true
```

##### Set if other storageclass
If other storageclass has been installed in advance, for example `nfs-client` has been installed and set as default, it must be added into config, and the final `HELM` > `VALUES` field with `isSNO` will look like :

```yaml
cp4waiops:
  isSNO: true
  storageClass: nfs-client
  storageClassLargeBlock: nfs-client
```

##### Override PARAMETERS
In Helm PARAMETERS, you should override `cp4waiops.profile` and `cp4waiops.eventManager.enabled`, and you can also update other install parameters that are commonly used to customize the install behavior.

| Parameter                             | Type   | Default Value      |    override|  Description 
| ------------------------------------- |--------|--------------------| ----------------------------- | -------------
| **cp4waiops.profile**                 | string | small              |  <li>small</li><li>***x-small***</li><li>***x-small-idle***</li><li>***x-small-custom***</li>       |  The CP4WAIOps deployment profile.<p> e.g.: large, small and ***x-small, x-small-idle, x-small-custom for custom sizing***.
| **cp4waiops.eventManager.enabled**    | bool   | true               |     **false** if *x-small-* profile  |  Specify whether or not to install Event Manager.
| rookceph.enabled                      | bool   | true               |  **false** if other storageClass is configured           |  Specify whether or not to install Ceph as storage used by CP4WAIOps.
| argocd.cluster                        | string | openshift          |            |  The type of the cluster that Argo CD runs on, valid values include: openshift, kubernetes.
| argocd.allowLocalDeploy               | bool   | true               |            |  Allow apps to be deployed on the same cluster where Argo CD runs.
| cp4waiops.version                     | string | v3.4               |            |  Specify the version of CP4WAIOps, e.g.: v3.2, v3.3, v3.4.
| cp4waiops.aiManager.enabled           | bool   | true               |            |  Specify whether or not to install AI Manager.
| cp4waiops.aiManager.namespace         | string | cp4waiops          |            |  The namespace where AI Manager is installed.
| cp4waiops.aiManager.instanceName      | string | aiops-installation |            |  The instance name of AI Manager.
| cp4waiops.eventManager.namespace      | string | noi                |            |  The namespace where Event Manager is installed.
| cp4waiops.eventManager.clusterDomain  | string | REPLACE_IT         |            |  The domain name of the cluster where Event Manager is installed.

NOTE:

- For `cp4waiops.profile`, defaut is `small`, and you can specify the ***extra small*** sub-profile `x-small, x-small-idle, x-small-custom` that are only for demo, PoC, or dev environment in a resource (cpu and memory) limited cluster, in which `x-small-idle` does not require a sustained workload. If you are looking for official installation, use profile such as `small` or `large` instead.
- For `cp4waiops.eventManager.enabled`, it needs to be **false** if you use `x-small, x-small-idle, x-small-custom` profile as they only cover AI Manager, not including Event Manager.
- For `rookceph.enabled`,  default is `true` to install ceph as storage during installation. If the default storageclass has been configured in advance, it must be override to `false` and configure it in `HELM` > `VALUES` field.


#### Install Custom Build 

The above install will install CP4WAIOps **3.4 GA** build, to install **Custom Build**, please refer [Install CP4WAIOps using Custom Build](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops.md#install-cp4waiops-using-custom-build)




#### Wourkaround pending redis pod

In normal multi-node OCP cluster, redis HA will deployed into diffrent nodes. Since in SNO there is only one node to deploy, redis HA will failed to install multi redis into one node resulting redis member pods pending. To workaround this in SNO,  `podAntiAffinity` must be set as null and restart the pods.


If redis member pods pending, 
```console
$ oc get pod | grep redis-m-
NAME                              READY   STATUS             RESTARTS           AGE
c-example-redis-m-0               0/4     Pending                1              66m
c-example-redis-m-1               0/4     Pending                1              66m
```

then
- Check redissentinel

```console
$ oc get redissentinels

NAME            TYPE        STATUS   REASON   MESSAGE
example-redis   Available   True     Synced   Synced successfully

#make sure its '.spec.members.affinity' is empty.
$ oc get redissentinels example-redis -o yaml | yq .spec.members.affinity
{}

#make sure its '.spec.size' is '2'.
$ oc get redissentinels example-redis -o yaml | yq .spec.size
2
```


- Check Statefulsets

To Verify Statefulsets  `c-example-redis-m`, 

```console
$ oc get sts c-example-redis-m

NAME                READY   AGE
c-example-redis-m   2/2     66m

#make sure its '.spec.template.spec.affinity.podAntiAffinity' is empty.
$ oc get sts c-example-redis-m -o yaml | yq .spec.template.spec.affinity.podAntiAffinity
null

#make sure its '.spec.replicas' is '2'.
$ oc get sts c-example-redis-m -o yaml | yq .spec.replicas
2
```

- Delete pod `c-example-redis-m-0` and `c-example-redis-m-1` , and new pods will be started and running.

```console
$ oc get pod | grep redis-m-
NAME                              READY   STATUS             RESTARTS           AGE
c-example-redis-m-0               4/4     Running                1              66m
c-example-redis-m-1               4/4     Running                1              66m
```





### Verify installation

Wait about 1 hour that all pods are running, such as , and the number of pods is more than 140.


```console
$ oc get pod | wc -l 
146

$ oc get pod | grep topology | wc -l
11

$ oc get AIManager,aiopsedge,asm -A -o custom-columns="KIND:kind,NAMESPACE:metadata.namespace,NAME:metadata.name,STATUS:status.phase"
KIND                         NAMESPACE   NAME             STATUS
AIManager                    aiops       aimanager        Completed
AIOpsEdge                    aiops       aiopsedge        Configured
ASM                          aiops       aiops-topology   OK

```

For more details, please refer to [Verify CP4WAIOps Installation](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops.md#verify-cp4waiops-installation)

### Access CP4WAIOps

If all pods for CP4WAIOps are up and running, you can login to CP4WAIOps UI as follows: [Access CP4WAIOps](https://github.com/IBM/cp4waiops-gitops/blob/docs/docs/how-to-deploy-cp4waiops.md#access-cp4waiops)
