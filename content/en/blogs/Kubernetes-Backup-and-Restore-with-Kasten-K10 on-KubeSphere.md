---
title: 'Kubernetes Backup and Restore with Kasten K10 on KubeSphere'
keywords: Backup and Restore, Kasten K10, Kubernetes
description: Several steps to realize Kubernetes backup and restore with Kasten K10 on KubeSphere
tag: 'Kasten K10, Kubernetes'
createTime: '2021-09-25'
author: 'Mars Zhang， Yicen Xie'
snapshot: '/images/blogs/en/kastenk10image/KastenK10-architecture.png'
---

## Kasten on KubeSphere

Purpose-built for Kubernetes, [Kasten 10 ](https://docs.kasten.io/)provides enterprise operations teams an easy-to-use, scalable, and secure system for backup/restore, disaster recovery, and mobility of Kubernetes applications. 

![kasten k10 arch](/images/blogs/en/kastenk10image/KastenK10-architecture.png)
[KubeSphere](https://kubesphere.io/docs/introduction/what-is-kubesphere/) is a distributed operating system for cloud-native application management, using Kubernetes as its kernel. It provides a plug-and-play architecture, allowing third-party applications to be seamlessly integrated into its ecosystem.
![kubesphere arch](/images/blogs/en/kastenk10image/kubesphere-architecture.png)

In this article, we will introduce the deployment of Kasten K10 on KubeSphere.

## Provision a KubeSphere Cluster

This article will introduce how to deploy Kasten on on KubeSphere Container Platform. You can install KubeSphere on any Kubernetes cluster or Linux system, refer to [KubeSphere documentation](https://kubesphere.io/docs/quick-start/all-in-one-on-linux/) for more details or vist the [Github]( https://github.com/kubesphere/website) of KubeSphere.

After the creation of KubeSphere cluster, you can log in to KubeSphere web console:

![web console](/images/blogs/en/kastenk10image/kastenk10-step3.png)
Click the button  "Platform" in the upper left corner and then select "Access Control"; Create a new workspace called Kasten-Workspace.

![kasten-workspace](/images/blogs/en/kastenk10image/kastenk10-step4.png)
Enter "Kasten-workspace" and select "App Repositoties"; Add an application repository named Kasten.

![add repository](/images/blogs/en/kastenk10image/kastenk10-step5.png)
Add the official Helm Repository of Kasten to KubeSphere. **Helm repository address**[2]：`https://charts.kasten.io/`

![official helm repository](/images/blogs/en/kastenk10image/kastenk10-step6.png)
Once completed, the repository will find its status be "successful".

![successful status](/images/blogs/en/kastenk10image/kastenk10-step7.png)
## Deploy Kasten K10 on Kubernetes to Backup and Restore Cluster

First, create a Namespace to run Kasten.

![create a name space](/images/blogs/en/kastenk10image/kastenk10-step8.png)
After creation, click the Namespace and select "App"-"Deploy New APP"-"From App Template"successively.

![deploy new app](/images/blogs/en/kastenk10image/kastenk10-step9.png)
Select "Kasten" from the drop-down menu and then select "K10".

![select k10](/images/blogs/en/kastenk10image/kastenk10-step10.png)
Click "K10" and enter the detail page of Chart; Click "Configuration File" to view or download the default values.yaml; Select the version and click to deploy; Set "App Name" as "K10" and select the "App Version"; Confirm the "Deployment Location", and click "Next".

![page of chart](/images/blogs/en/kastenk10image/kastenk10-step11.png)
For this step, we need to input required parameters to deploy Kasten K10 on KubeSphere. We need to adjust the parameters in line with our own needs. During the installation, adjust the parameters as follows and click "Deploy".

![adjust parameters](/images/blogs/en/kastenk10image/kastenk10-step12.png)
  ```
global: 
    airgapped: 
      repository: "ccr.ccs.tencentyun.com/kasten-k10"
    persistence: 
      storageClass: "csi-hostpath-sc"
  auth: 
    basicAuth: 
      enabled: "true"
      htpasswd: "mars:$apr1$Cgu1sGVZ$w/8aLHZkVT73OqYZ06C0v."
  metering: 
    mode: "airgap"
  injectKanisterSidecar: 
    enabled: "true"
  prometheus: 
    server: 
      persistentVolume: 
        enabled: "false"
  ingress: 
    create: "true"
    class: "nginx"
  ```
Click "Deploy" and wait the status to turn into "running".
![apps](/images/blogs/en/kastenk10image/kastenk10-step13.png)
Click "Deployment" to check if Kasten has deployed workload and is in running status.

![workloads](/images/blogs/en/kastenk10image/kastenk10-step14.png)

In “Application Workloads” - “Routes” page, we can find the Gateway of Ingress configured previously.

![routes](/images/blogs/en/kastenk10image/kastenk10-step15.png)



## Log in the Management Page of Kasten K10

Input `https://192.168.99.100/k10/#`  to the browser for the following log-in interface; Input the company and e-mail address to sign up.
![login page](/images/blogs/en/kastenk10image/kastenk10-step16.png)

Set the locations for storing our backup data. In this case S3 compatible storage is selected.

![settings1](/images/blogs/en/kastenk10image/kastenk10-step17.png)

Finally, start "K10 Disaster Recovery" and we can start to set "Disaster Recovery" for our cloud native application.

![settings2](/images/blogs/en/kastenk10image/kastenk10-step18.png)



## Deploy Cloud Native Applications on Kubernetes

Kasten Dashboard holds 16 applications, which are shown as follows. We can create a Wordpress application with a Wordpress Pod and Mysql Pod, a typical application that is partly stateful and partly stateless. Here are the steps.

![kasten dashboard1](/images/blogs/en/kastenk10image/kastenk10-step19.png)



### Create kustomization.yaml

Create “Secret Generator". A Secrete can store sensitive data such as passwords or keys. Since v1.14, kubectl supports kustomization to manage Kubernetes objects. We can use the generator in `kustomization.yaml` to create a Secret.

We can add a Secret Generator to `kustomization.yaml` as follows. It should be noted that we need to replace YOUR_PASSWORD with our actual passwords.
```
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=YOUR_PASSWORD
EOF
```
**Download configuration file of MySQL deployment**
`curl -LO https://k8s.io/examples/application/wordpress/mysql-deployment.yaml`
**Download configuration file of Wordpress**
`curl -LO https://k8s.io/examples/application/wordpress/wordpress-deployment.yaml`
**Add to kustomization.yaml**
```
cat <<EOF >>./kustomization.yaml
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml
EOF
```
Apply and verify all resources applied to deploy WordPress website within `kustomization.yaml` and MySQL  database. We can set and apply the content as follows.
```
kubectl create ns wordpress
kubectl apply -k ./  -n wordpress
```
Once completed, we can find the number of application increase to 17.
![kasten dashboard2](/images/blogs/en/kastenk10image/kastenk10-step20.png)
In addition, applications of the WordPress can also be find in "Applications".

![wordpress](/images/blogs/en/kastenk10image/kastenk10-step21.png)

## Back Up Cloud Native Applications

Click "Create Policy" and create a data backup strategy. In such case, Kasten can protect applications by creating local snapshot, and back up the application data to cloud, thus to realize the long-term retention of data.
![create policy](/images/blogs/en/kastenk10image/kastenk10-step22.png)

Click "Run Once" to start backup.

![start backup](/images/blogs/en/kastenk10image/kastenk10-step23.png)

Observe Dashboard and we can find backup is completed.

## Complete the Recovery of Cloud Native Applications

Find "Applications" in Dashboard and then click "Restore".

![restore](/images/blogs/en/kastenk10image/kastenk10-step24.png)
Set the time for recovery.

![recovery](/images/blogs/en/kastenk10image/kastenk10-step25.png)

Create a Namespace for recovery, which is named as "Wordpress-restore" in this case.

![](/images/blogs/en/kastenk10image/kastenk10-step26.png)

As every application recovered will create a new Namespace, the number of applications increases from 17 to 18. In Dashboard we can find that applications have been fully recovered.

![fully recovered](/images/blogs/en/kastenk10image/kastenk10-step27.png)

In KubeSphere Dashboard, we can find these applications recovered are running.

![successful running](/images/blogs/en/kastenk10image/kastenk10-step28.png)



## Summary

As a container platform, KubeSphere excels in cloud native application deployment. For application developers who are not familiar with Kubernetes and hope to make simple configuration to deploy Kasten, it is easy to follow the above steps and deploy Kasten with KubeSphere. KubeSphere helps to directly deploy the official Helm repository of Kasten K10, which performs well in data management, including backup, migration and disaster recovery.


### Reference

[KubeSphere Official Documentation](https://kubesphere.io/docs/)

[Helm Repository Address](https://charts.kasten.io/)

[Kasten Official Documentation](https://docs.kasten.io/)

[KubeSphere GitHub](https://github.com/kubesphere/kubesphere)

[KubeSphere Official Document](https://kubesphere.io/docs/introduction/what-is-kubesphere/)

[KubeSphere Documentation](https://kubesphere.io/docs/) 