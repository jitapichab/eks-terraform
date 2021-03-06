                                                  ### eks-terraform ###
                                                     
                                                     
![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/amazon-eks-logo.png)


Terraform module to install Kubernetes using EKS multi-az in AWS with autoscaling groups(tags) for Workers.

# Properties:
###EKS Cluster: Control Plane in multi AZ.

###Autoscaling Groups for Workers with NODEGROUP/PLACEMENT_CONSTRAINT.

###Iam Roles and Policies.

###Aws Integration to create a ELB,NLB,ALB in public subnets.

###VPC, Multiple subnets, public subnets, private subnets, internet gateway and nat gateways.

###NOTE: Check Amazon CNI ips support for instance type. 
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI

Example AWS-CNI
```
c4.xlarge 4interfaces ips 15     15 containers x instance
m4.large  2interfaces ips 10     10 containers x instance
r4.large  3interfaces ips 10     10 containers x instance
```
Testing AWS-CNI plugin with t2.medium support 17 ips, When need create a ip number 18 fails the cni for instance cni limits.
![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/eks-cni.png)

# Architecture

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/eks-art.png)

# VPC-EXAMPLE:
![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/vpc-example.png)

Installation:

1- You need configure :
s3 bucket for tfstate in main.tf
aws_region,aws_profile,key,domain_name,reverse_zone,vpc-cidr,subnets in terraform.tfvars

2- Launch Terraform:
terraform init 
terraform plan
terraform apply -auto-approve

3- Wait 10 minutes and save the first part of output in file :

eks-config-auth.yml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::881653854182:role/eks-nodes
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

4- Load cluster configuration in .kube/config to access the cluster.

```
aws eks update-kubeconfig --name eks-cluster --profile virginia-kubernetes
```

5- Apply eks-config-auth to permit workers registration in EKS-CLUSTER.
```
kubectl create -f eks-config-auth.yml
```
6- Check nodes registration
```
➜  eks-up-and-running k get nodes --label-columns group
NAME                            STATUS    ROLES     AGE       VERSION   GROUP
ip-10-100-18-161.ec2.internal   Ready     <none>    4s        v1.10.3   node
ip-10-100-18-97.ec2.internal    Ready     <none>    0s        v1.10.3   node
ip-10-100-50-11.ec2.internal    Ready     <none>    3s        v1.10.3   traefik
ip-10-100-50-235.ec2.internal   Ready     <none>    1s        v1.10.3   traefik
```

7- Test autoscaling workers groups changing desired and max.

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/asg-desired.png)

wait 1 minute and check nodes to look the correctly registration.

```
➜  eks-up-and-running k get nodes --label-columns group
NAME                            STATUS    ROLES     AGE       VERSION   GROUP
ip-10-100-18-161.ec2.internal   Ready     <none>    14m       v1.10.3   node
ip-10-100-18-97.ec2.internal    Ready     <none>    14m       v1.10.3   node
ip-10-100-19-49.ec2.internal    Ready     <none>    9m        v1.10.3   node
ip-10-100-50-11.ec2.internal    Ready     <none>    14m       v1.10.3   traefik
ip-10-100-50-235.ec2.internal   Ready     <none>    14m       v1.10.3   traefik
ip-10-100-51-22.ec2.internal    Ready     <none>    9m        v1.10.3   traefik
```

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/asg-nodes.png)

8- Check dns-service is success. 
```
➜  eks-up-and-running k get po --all-namespaces
NAMESPACE     NAME                        READY     STATUS              RESTARTS   AGE
kube-system   aws-node-2lg6j              1/1       Running             0          10s
kube-system   aws-node-d98q2              1/1       Running             0          6s
kube-system   aws-node-n9lhz              1/1       Running             0          7s
kube-system   aws-node-ngwfb              1/1       Running             0          9s
kube-system   kube-dns-64b69465b4-js5km   0/3       ContainerCreating   0          6m
kube-system   kube-proxy-kqs7f            1/1       Running             0          10s
kube-system   kube-proxy-mmn2l            1/1       Running             0          9s
kube-system   kube-proxy-rjjnq            1/1       Running             0          6s
kube-system   kube-proxy-vssqc            1/1       Running             0          7s
```
9- Install helm and tiller.
```
cd eks-up-and-running
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller --upgrade
```

10- Install HPA-metrics server.

```
➜  eks-metrics mkdir eks-metrics
➜  eks-metrics cd eks-metrics
```

```
➜  eks-metrics git clone https://github.com/kubernetes-incubator/metrics-server.git
```
You need change this file metrics-server-deployment.yaml and add this line to fix eks2 to works metrics-server:
```
command:
    - /metrics-server
    - --kubelet-preferred-address-types=InternalIP
```
```
➜  eks-metrics vi metrics-server/deploy/1.8+/metrics-server-deployment.yaml
```
```
 ---
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: metrics-server
   namespace: kube-system
 ---
 apiVersion: extensions/v1beta1
 kind: Deployment
 metadata:
   name: metrics-server
   namespace: kube-system
   labels:
     k8s-app: metrics-server
 spec:
   selector:
     matchLabels:
       k8s-app: metrics-server
   template:
     metadata:
       name: metrics-server
       labels:
         k8s-app: metrics-server
     spec:
       serviceAccountName: metrics-server
       volumes:
       # mount in tmp so we can safely use from-scratch images and/or read-only containers
       - name: tmp-dir
         emptyDir: {}
       containers:
       - name: metrics-server
         image: k8s.gcr.io/metrics-server-amd64:v0.3.1
         imagePullPolicy: Always
         command:
             - /metrics-server
             - --kubelet-preferred-address-types=InternalIP
         volumeMounts:
         - name: tmp-dir
           mountPath: /tmp
```
git diff to look changes in metrics-server-deployment.yaml
```
➜  metrics-server git:(master) ✗ git diff
--- a/deploy/1.8+/metrics-server-deployment.yaml
+++ b/deploy/1.8+/metrics-server-deployment.yaml
@@ -31,6 +31,9 @@ spec:
       - name: metrics-server
         image: k8s.gcr.io/metrics-server-amd64:v0.3.1
         imagePullPolicy: Always
+        command:
+            - /metrics-server
+            - --kubelet-preferred-address-types=InternalIP
         volumeMounts:
         - name: tmp-dir
           mountPath: /tmp
(END)
```
```
➜  eks-metrics k create -f metrics-server/deploy/1.8+
clusterrole.rbac.authorization.k8s.io "system:aggregated-metrics-reader" created
clusterrolebinding.rbac.authorization.k8s.io "metrics-server:system:auth-delegator" created
rolebinding.rbac.authorization.k8s.io "metrics-server-auth-reader" created
apiservice.apiregistration.k8s.io "v1beta1.metrics.k8s.io" created
serviceaccount "metrics-server" created
deployment.extensions "metrics-server" created
service "metrics-server" created
clusterrole.rbac.authorization.k8s.io "system:metrics-server" created
clusterrolebinding.rbac.authorization.k8s.io "system:metrics-server" created

➜  eks-metrics k top nodes
NAME                            CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%
ip-10-100-18-108.ec2.internal   19m          1%        320Mi           16%
ip-10-100-19-76.ec2.internal    16m          1%        359Mi           18%
ip-10-100-50-18.ec2.internal    19m          1%        368Mi           19%
ip-10-100-50-231.ec2.internal   15m          1%        338Mi           17%

```

11- Launch kubernetes-dashboard and 2 applications , 1 public application with ELB and 1 private application.

```
cd eks-up-and-running
```
```

➜  eks-up-and-running k create -f eks-app-autoscaling
horizontalpodautoscaler.autoscaling "nginx" created
deployment.extensions "nginx" created
service "nginx" created
deployment.extensions "nginx2" created
service "nginx2" created

➜  eks-app-autoscaling k get svc -owide
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE       SELECTOR
kubernetes   ClusterIP      172.20.0.1      <none>        443/TCP        42m       <none>
nginx        LoadBalancer   172.20.71.17    <pending>     80:30417/TCP   1m        app=nginx,env=stg
nginx2       ClusterIP      172.20.30.143   <none>        80/TCP         6m        app=nginx,env=stg
```
```
➜  eks-app-autoscaling k get svc -owide
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)        AGE       SELECTOR
kubernetes   ClusterIP      172.20.0.1      <none>                                                                    443/TCP        42m       <none>
nginx        LoadBalancer   172.20.71.17    afe917206f44011e8abc402ddbc5496a-1082087107.us-east-1.elb.amazonaws.com   80:30417/TCP   2m        app=nginx,env=stg
nginx2       ClusterIP      172.20.30.143   <none>                                                                    80/TCP         6m        app=nginx,env=stg
```

Open the url using the FQDN detailed in EXTERNAL-IP. 

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/app1.png)

12- Launch Kubernetes-dashboard and get token.
```
➜  eks-up-and-running k create -f eks-dashboard
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created

kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &

➜  eks-dashboard  aws-iam-authenticator token -i eks-cluster
{"kind":"ExecCredential","apiVersion":"client.authentication.k8s.io/v1alpha1","spec":{},"status":{"token":"k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFKR05UNlRRMk9WSE1CMldRJTJGMjAxODExMzAlMkZ1cy1lYXN0LTElMkZzdHMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDE4MTEzMFQwMTU0NTZaJlgtQW16LUV4cGlyZXM9NjAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JTNCeC1rOHMtYXdzLWlkJlgtQW16LVNpZ25hdHVyZT1jNjBlNTljYzI2ZmY5MzIyMDJlZGYzMGRiODAzMzc5MzE4NjgwMjY3ZGQyMDNiYmE2NTBkM2I2NzBhOTM3OGQy"}}
```

Open dashboard url in your browser using kube-proxy in port 8080 and use the token auth.

http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/dash-token.png)

Look the dashboard.

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/dask-kube.png)

13- You can configure hpa and testing the applications with load average.

cd eks-app-autoscaling
```
➜  eks-app-autoscaling git:(master) ✗ cat autoscaling.yml
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: nginx
  minReplicas: 30
  maxReplicas: 35
  targetCPUUtilizationPercentage: 30
  
➜  eks-app-autoscaling git:(master) ✗ k apply -f autoscaling.yml

➜  eks-app-autoscaling git:(master) ✗ k get hpa
NAME      REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx     Deployment/nginx   2%/30%    30        35        30         3h
```

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/load.png)

![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/eks-stack.png)

14- I created the configuration to install traefik in EKS with simple auth to dashboard access.

cd eks-traefik

#CREATE USER WITH SIMPLE AUTH
```
htpasswd -c ./auth zz1
cat auth
kubectl create secret generic mysecret --from-file auth --namespace=kube-system
```

#CREATE ROLE TRAEFIK ACCESS-READ THE k8s METADATA
```
➜  eks-traefik kubectl create serviceaccount --namespace kube-system traefik-ingress-controller
➜  eks-traefik kubectl create clusterrolebinding traefik-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:traefik-ingress-controller
```
```
k apply -f .
➜  eks-traefik git:(master) ✗ k create -f .
configmap "traefik-conf" created
serviceaccount "traefik-ingress-controller" created
deployment.extensions "traefik-ingress-controller" created
ingress.extensions "traefik-ingress-service-admin" created
service "traefik-ingress-service-admin" created
service "traefik-ingress-service" created
```
#Login to traefik-dashboard and k8s-dashboard with simple-auth
![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/eks-traefik-simple-auth.png)

#Look traefik-dashboard reading k8s-metadata
![alt text](https://raw.githubusercontent.com/nightmareze1/eks-terraform/master/img/eks-traefik.png)

THANKS for use this repo!!!

# REGARDS AND INSPIRATION:

https://github.com/anmolnagpal/terraform-eks

https://github.com/terraform-aws-modules/terraform-aws-eks

https://www.iheavy.com/2018/07/31/how-to-setup-an-amazon-eks-demo-with-terraform/

https://github.com/christopherhein/terraform-eks

https://github.com/howdio/terraform-aws-eks

https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html

# AWS - EKS Workshop

https://eksworkshop.com/

https://aws.amazon.com/es/blogs/opensource/horizontal-pod-autoscaling-eks/
