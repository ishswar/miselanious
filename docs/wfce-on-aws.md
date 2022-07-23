
# WF-CE on AWS 

## Introduction 
To deploy WFCE on EKS, these are basic overall steps. 
We assume you have an access to AWS account that can create or access - EKS , ECR , EFS & ELB. 
We are also assuming that you are planing to use most of the things as `default` such as namespace used to deploy WF-CE (i.e `webfocus`) 

In this doc WebFOCUS Container Edition is referred as WF-CE or WFCE all can be used interchangeably.

These steps are specifically writen keeping mind that use is deploying to AWS using EKS ( AWS managed kubernetes cluster). Also when you use any remove Kubernetes cluster  
we need Image repository in this case we are assume you are using ECR . Same with shared storage - since WF-CE is designed to scale horizontally we need common storage on  
kubernetes cluster and that is assumed to be done using Amazon EFS.  

General setup looks like this (only showing AWS components): 

![](https://i.ibb.co/dBMvGrX/Paldi-Page-3.png)

## Steps 

1. Get a Linux machine ( preferably on AWS EC2 itself ) - Install - the latest OS updates, Docker, kubectl, Helm, and Helmfile tools  
   Make sure you have access to user who can run as `sudo` as some of command might require `sudo` access
2. Download WFCE tar from TIBCO  
3. Un-tar it and go over Deployment guide PDF (`TIBCO_WF_CE_Deployment_Guide.pdf`)
4. Either using `eksctl` or AWS Console create a EKS cluster ( kubernetes version 1.20 onward is fine )  - the Linux machine (from step 1 above) and EKS can be on the same AWS region or can be on different regions 
5. Create an EKS with - 2 machine node group, Each machine should be 8 CPU and 16GB RAM to start with ( `c5.12xlarge` EC2 machine type will do it ) 
6. Once EKS is created from Linux box run - aws CLI to get `kubeconfig` file to connect to EKS Cluster.  
   Refer to [doc](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) or [doc2](https://aws.amazon.com/premiumsupport/knowledge-center/eks-generate-kubeconfig-file-for-cluster/) or [link](https://kerneltalks.com/commands/how-to-configure-kubectl-for-aws-eks/)
7. Run `kubectl get nodes` command to see if you can see your Nodes ( a.k.a EC2 machines ) - Make sure you have nodes in the `Ready` state 

##### Sample output from our setup 

```bash
:~/webfocus-ce$kubectl get nodes
NAME                                            STATUS   ROLES    AGE    VERSION
ip-192-168-156-240.us-west-2.compute.internal   Ready    <none>   4d9h   v1.22.9-eks-810597c
ip-192-168-197-122.us-west-2.compute.internal   Ready    <none>   4d9h   v1.22.9-eks-810597c'
'
```

You can use below command to see what is machine capacity of your one of above nodes 
Replace `ip-192-168-156-240.us-west-2.compute.internal` with your node name 

```
:~/webfocus-ce$kubectl get nodes ip-192-168-156-240.us-west-2.compute.internal -o json | grep -A8 capacity
        "capacity": {
            "attachable-volumes-aws-ebs": "25",
            "cpu": "8",
            "ephemeral-storage": "20959212Ki",
            "hugepages-1Gi": "0",
            "hugepages-2Mi": "0",
            "memory": "15918192Ki",
            "pods": "58"
        },
```   

8. Now run the command `kubectl get cs` to see your storage class. 

```
:~/webfocus-ce$kubectl get sc
NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2                kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  4d9h
```

9. You will need to install a storage class that supports the `ReadWriteMany` type of PVC (shared disk) 
10. We recommend using the "Amazon EFS CSI Driver" storage class - follow the steps at https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
11. Above steps will ask you to create a few IAM roles and EFS drivers - steps are in the above doc 
12. In same link above it will ask you to run sample application make sure it looks good (it runs successfully)

After you have successfully installed CSI driver if you re-run command `kubectl get sc` you should see output like this 

```
:~/webfocus-ce$kubectl get sc
NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
efs-sc (default)   efs.csi.aws.com         Delete          Immediate              false                  4d9h
gp2                kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  4d9h
```

Make sure `EFS` storage class is `default` else you can make it (default) using these two commands 

Make `EFS` default 

```
kubectl patch storageclass efs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

and make`gp2` non-default 

```
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

13. You need to also think about how are you going to access WebFOCUS GUI .. This is where you need to consult your company policy about how to access Web UI from outside or within  
    company - for now we will assume you can use NGINX ingress controller 
     
    You can install ingress controller in your cluster using command 
    
    ```
     helm upgrade --install ingress-nginx ingress-nginx \
     --repo https://kubernetes.github.io/ingress-nginx \
     --namespace ingress-nginx --create-namespace
    ```
     
    More info on this at : 
    If your EKS cluster does not have access to internet (Air-gap) than you will have to pull ingress controller image `registry.k8s.io/ingress-nginx/controller` manually and push it to your ECR repo
    You will also have to update Ingress Chart to use that local image repo 
    
    NGINX Ingress will try to create ELB on your AWS account so make sure it has access to do that.
    After you deploy ingress controller if you run command like below you should see output like this :
    
    ```
    :~/webfocus-ce$kubectl get -n ingress-nginx svc ingress-nginx-controller NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                                                     PORT(S)                      AGE
    ingress-nginx-controller   LoadBalancer   10.100.151.76   a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com   80:31453/TCP,443:31142/TCP   4d11h
    ```
    
    Make a note of hostname from above like `a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com` (ELB address)
        
13. Now you are ready to start working on WFCE - first, we need to build docker images 
14. Run script (`scripts/build-images.sh`) that is mentioned in Doc - That script will create 5 Docker images ( we are assuming your Linux box (we call it Build machine or server) has access to the internet - as during Docker build, we do pull images from internet - https://hub.docker.com/r/redhat/ubi8 ]  

If you list docker images ( assuming you have nothing else on this machine ) output should look like below 

```
:~/webfocus-ce/scripts$docker images
REPOSITORY         TAG                 IMAGE ID       CREATED          SIZE
ibi2020/webfocus   postgres-alpine     d72008be450c   17 minutes ago   222MB
ibi2020/webfocus   wfc-9.0-1.0.2       9a451c504fc3   17 minutes ago   1.81GB
ibi2020/webfocus   cachemanager        a5f5cf0fc09c   19 minutes ago   941MB
ibi2020/webfocus   wfs-etc-9.0-1.0.2   e1eb9e30d804   20 minutes ago   4.03GB
ibi2020/webfocus   wfs-9.0-1.0.2       8776166afecd   23 minutes ago   1.06GB
```

15. Tag the images (Remember these tags you will need them later) 

In below example commands we are using ECR repo named `airgaptest` you need to update that with your repo name

```
docker tag ibi2020/webfocus:cachemanager 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:cachemanager
docker tag ibi2020/webfocus:postgres-alpine 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:postgres-alpine
docker tag ibi2020/webfocus:wfc-9.0-1.0.2 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:wfc-9.0-1.0.2
docker tag ibi2020/webfocus:wfs-etc-9.0-1.0.2 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:wfs-etc-9.0-1.0.2
docker tag ibi2020/webfocus:wfs-9.0-1.0.2 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:wfs-9.0-1.0.2
```

After tagging if you list images it should look like this 

```
:~/webfocus-ce$docker images
REPOSITORY                                                TAG                 IMAGE ID       CREATED          SIZE
729111267627.dkr.ecr.us-west-2.amazonaws.com/airgaptest   postgres-alpine     d72008be450c   26 minutes ago   222MB
ibi2020/webfocus                                          postgres-alpine     d72008be450c   26 minutes ago   222MB
729111267627.dkr.ecr.us-west-2.amazonaws.com/airgaptest   wfc-9.0-1.0.2       9a451c504fc3   26 minutes ago   1.81GB
ibi2020/webfocus                                          wfc-9.0-1.0.2       9a451c504fc3   26 minutes ago   1.81GB
729111267627.dkr.ecr.us-west-2.amazonaws.com/airgaptest   cachemanager        a5f5cf0fc09c   28 minutes ago   941MB
ibi2020/webfocus                                          cachemanager        a5f5cf0fc09c   28 minutes ago   941MB
729111267627.dkr.ecr.us-west-2.amazonaws.com/airgaptest   wfs-etc-9.0-1.0.2   e1eb9e30d804   30 minutes ago   4.03GB
ibi2020/webfocus                                          wfs-etc-9.0-1.0.2   e1eb9e30d804   30 minutes ago   4.03GB
ibi2020/webfocus                                          wfs-9.0-1.0.2       8776166afecd   32 minutes ago   1.06GB
729111267627.dkr.ecr.us-west-2.amazonaws.com/airgaptest   wfs-9.0-1.0.2       8776166afecd   32 minutes ago   1.06GB
```

16. Now push these images to ECR - you might have to log in to ECR from your Linux box before you can push images - Refer to this [Doc](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login.html)
17. If you are doing air-gap installation (that means your EKS does not have access to the internet ) - you will have to download a few more images from the internet and push it to your ECR - refer to the doc about what are the images you need to download, tag and push to ECR

Sample commands : 

```
docker pull bitnami/etcd:3.4.14-debian-10-r0
docker pull bitnami/postgresql:11.12.0-debian-10-r1
docker pull quay.io/prometheus/prometheus:v2.31.1
docker pull jimmidyson/configmap-reload:v0.5.0
docker pull k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1
docker pull bitnami/solr:8.8.2-debian-10-r0
docker pull bitnami/zookeeper:3.7.0-debian-10-r0
docker pull rainbond/metrics-server:v0.5.2
```

```
docker tag bitnami/etcd:3.4.14-debian-10-r0 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:3.4.14-debian-10-r0
docker tag bitnami/postgresql:11.12.0-debian-10-r1 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:11.12.0-debian-10-r1
docker tag quay.io/prometheus/prometheus:v2.31.1 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:v2.31.1
docker tag jimmidyson/configmap-reload:v0.5.0 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:v0.5.0
docker tag k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:v0.9.1
docker tag bitnami/solr:8.8.2-debian-10-r0 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:8.8.2-debian-10-r0
docker tag bitnami/zookeeper:3.7.0-debian-10-r0 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:3.7.0-debian-10-r0
docker tag rainbond/metrics-server:v0.5.2 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest:v0.5.2
```

If you change any of above tag name - than make sure to update file `scripts/helmfile/environments/local.yaml.gotmpl` - find that tag name and replace it what you used.

Now push them to ECR

18. Now edit file "scripts/helmfile/export-defaults.sh" and update your Docker image TAG that you did in step 15 above 
19. Source above shell script - We are exporting 4 environment variables into the shell by sourcing the above shell script.

```
:~/webfocus-ce$source scripts/helmfile/export-defaults.sh
Variables exported:
PLATFORM_NAME=webfocus
INFRA_NS=webfocus
WF_TAG=wfc-9.0-1.0.2
RS_TAG=wfs-9.0-1.0.2
ETC_TAG=wfs-etc-9.0-1.0.2
```
You can check if they got exported to your shell 

```
:~/webfocus-ce$env | grep TAG
RS_TAG=wfs-9.0-1.0.2
WF_TAG=wfc-9.0-1.0.2
ETC_TAG=wfs-etc-9.0-1.0.2
```

20. Now edit the file "scripts/helmfile/environments/wf.integ.yaml.gotmpl" and edit key "global.imageInfo.image.repository" with your ECR repo URL (i.e 0123xxxxxx.dkr.ecr.us-west-2.amazonaws.com/airgaptest)
21. We are assuming your EKS can access your ECR without any authentication - if it needs to use different credentials, then you will have to provide username and password - please refer to the Deployment Guide for more info  
22. Go over file "scripts/helmfile/environments/cloud-wf.integ.yaml.gotmpl" - we will be using this file as helmfile environment file 
23. If you are doing the air-gap installation, you need to edit the file "scripts/helmfile/environments/local.yaml.gotmpl" - refer to the Deployment Guide for more info on this 
24. Now you are ready to proceed with deployment -  This step will assume you will use Postgres within the cluster ( we will deploy Postgres DB in Pod ) - if you need to use your own Postgres, then refer to the Deployment Guide.
25. If you are using external Postgres running in AWS, then make sure your EKS cluster nodes ( machines ) have access to ( connect to) Postgres as well (Need to set correct security groups in RDS)( you can use command like : `pg_isready -d <db_name> -h <host_name> -p <port_number> -U <db_user>` - create a temporary pod that has tool `pg_isready` and run command with proper values ) 
26. First part of the deployment is to create infra ( infrastructure component ) - you can find helmfile sync command in doc for that ( you need to be in directory: scripts/helmfile/infra/) 
27. If the infrastructure part gets created successfully, proceed to the next step 
28. Now, from dir "scripts/helmfile" run the second helmfile sync command this will create all major WebFOCUS components 
29. At the end, it will run smoke test, and if that passes, all seems to be running good

### Smoke test sample output 

```
 Running command [/opt/ibi/srv/home/bin/interp00.out -edaconf /opt/ibi/srv/storage/wfs -httptst /opt/ibi/srv/temp/smoke-test.hti -nosleep]
 ===========================================================
 /opt/ibi/srv/temp/smoke-test.hti
 /opt/ibi/srv/home/bin/httptst.out /opt/ibi/srv/temp/smoke-test.hti -nosleep

 ---- Started at 10:40:20 ----

 Received: thread=0001 request=0001; timing: resp=0.364 sec, transf=0.000 sec, start=07/18/2022 10:40:20.742; dbmstime=0.000 sec, servertime=0.364 sec
 Received: thread=0001 request=0002; timing: resp=1.615 sec, transf=0.003 sec, start=07/18/2022 10:40:21.106; dbmstime=0.000 sec, servertime=1.618 sec
 Received: thread=0001 request=0003; timing: resp=0.105 sec, transf=0.000 sec, start=07/18/2022 10:40:22.724; dbmstime=0.000 sec, servertime=0.105 sec

 ---- Finished at 10:40:22 ----

 Total Execution Time:                    2.089 sec
 Total Number of Threads:                 1
 Interval Parameters:                     1000,-1,-1,-1
 Minimum Execution Time of Thread:        2.088 sec
 Maximum Execution Time of Thread:        2.088 sec
 Average Execution Time of Thread:        2.089 sec

 Total Number of Requests:                3
 Average Server Response:                 0.695 sec of total 3 requests
 Average Data Transfer Time:              0.003 sec of total 1 requests
 Average Processing Time:                 0.696 sec of total 3 requests
 Standard Processing Deviation:           0.661 sec
 Minimum Processing Time:                 0.105 sec
 Maximum Processing Time:                 1.618 sec
 Average DBMS Time:                       0.000 sec of total 3 requests
 Minimum DBMS Time:                       0.000 sec
 Maximum DBMS Time:                       0.000 sec
 Average Server Time:                     0.696 sec of total 3 requests
 Minimum Server Time:                     0.105 sec
 Maximum Server Time:                     1.618 sec

 terminating main thread
 +++
 =====================
 Test suceeded - Average server time is [0.696]
 =====================
 ===========================================================
 Running health check test
 ===========================================================
 appserver host : appserver.webfocus
 appserver port : 8080

 [2022-07-18_10-40-22] reporting server test start
 About to run [curl -s --connect-timeout 30 -m 100 http://appserver.webfocus:8080/webfocus/oslo/1.0/health/reportingserver]
 {"security":"ok","jscom3":"ok","headless-browser":"ok","nodejs":"ok","dsml":"fail","dbms":[{"name":"PostgreSQL","connection":"CON1","status":"ok"},{"name":"TIBCO (R) Data Virtualization (

 security ok
 headless-browser ok
 nodejs ok
 [2022-07-18_10-40-23] reporting server test end

 [2022-07-18_10-40-23] reportcaster test start
 About to run [curl -s --connect-timeout 30 -m 100 http://appserver.webfocus:8080/webfocus/oslo/1.0/health/rcaster]
 reportcaster response:
 {"status":"UP"}

 reportcaster ok
 [2022-07-18_10-40-23] reportcaster test end

 [2022-07-18_10-40-23] Repository test start
 About to run [curl -s --connect-timeout 30 -m 100 http://appserver.webfocus:8080/webfocus/oslo/1.0/health/repository]
 repository response
 {"status":"UP"}

 Repository ok
 [2022-07-18_10-40-23] Repository test end
 ```
 
You can also check if all WF-CE and infra components are running - via running below command   
All components run as [StateFullSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
```
:~/webfocus-ce$kubectl get sts -n webfocus
NAME                    READY   AGE
appserver               1/1     4d9h
cachemanager            1/1     4d10h
clm                     1/1     4d9h
edaserver               1/1     4d9h
etcd                    1/1     4d10h
postgresql-postgresql   1/1     4d10h
reportcaster            1/1     4d9h
solr                    3/3     4d10h
solr-zookeeper          1/1     4d10h
```

You can also list all the pods and they should be in Running stats 
This output has Postgres running in cluster - if you were to use external PostgresSQL you will not see pod `postgresql-postgresql-0` in below output
 
```
:~/webfocus-ce$kubectl get pods -n webfocus
NAME                                              READY   STATUS      RESTARTS   AGE
appserver-0                                       1/1     Running     0          4d9h
cachemanager-0                                    1/1     Running     0          4d10h
clm-0                                             1/1     Running     0          4d10h
edaserver-0                                       1/1     Running     0          4d9h
etcd-0                                            1/1     Running     0          4d10h
postgresql-postgresql-0                           1/1     Running     0          4d10h
prom-adapter-prometheus-adapter-c9c958b8d-h76w5   1/1     Running     0          4d10h
prometheus-server-5c46cb6c5c-hgld6                2/2     Running     0          4d10h
reportcaster-0                                    1/1     Running     0          4d9h
smoke-test-dlgvq                                  0/1     Completed   0          4d9h
solr-0                                            1/1     Running     0          4d10h
solr-1                                            1/1     Running     0          4d10h
solr-2                                            1/1     Running     0          4d10h
solr-zookeeper-0                                  1/1     Running     0          4d10h
```

30. One of the last step is to fix `ingress` object inside webfocus 

If above everything got installed correctly in above step thant in `webfocus` namespace we do install one dummy `ingress` object - we need to fix that to adapt to your system.
It should look like this : 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    meta.helm.sh/release-name: appserver
    meta.helm.sh/release-namespace: webfocus
    nginx.ingress.kubernetes.io/affinity: cookie
    nginx.ingress.kubernetes.io/affinity-mode: persistent
    nginx.ingress.kubernetes.io/app-root: /webfocus
    nginx.ingress.kubernetes.io/client-body-buffer-size: 64k
    nginx.ingress.kubernetes.io/proxy-body-size: 200m
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/session-cookie-change-on-failure: "true"
    nginx.ingress.kubernetes.io/session-cookie-expires: "28800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "28800"
    nginx.ingress.kubernetes.io/session-cookie-name: sticknesscookie
    nginx.ingress.kubernetes.io/session-cookie-path: /
    nginx.ingress.kubernetes.io/whitelist-source-range: 0.0.0.0/0
  creationTimestamp: "2022-07-18T10:38:01Z"
  generation: 8
  labels:
    app.kubernetes.io/component: webfocus
    app.kubernetes.io/instance: appserver
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: appserver
    app.kubernetes.io/part-of: TIBCO-WebFOCUS
    app.kubernetes.io/version: 1.0.2
    helm.sh/chart: appserver-1.0.2
  name: appserver
  namespace: webfocus
spec:
  rules:
  - host: a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com
    http:
      paths:
      - backend:
          service:
            name: appserver
            port:
              name: port8080
        path: /(.*)
        pathType: ImplementationSpecific
```

Replace hostname `a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com` above with hostname you had in step 13 

Now save above output as file called `ingress-new.yaml` 
Apply to system 

```
# Backup old one 
kubectl get ingress -n webfocus appserver -o yaml > ingress-old.yaml
# Apply new one 
kubectl replace --force -f ingress-new.yaml
ingress.networking.k8s.io "appserver" deleted
ingress.networking.k8s.io/appserver replaced
```

You can check status of ingress object

```
:~/webfocus-ce$kubectl get -n webfocus ingress
NAME        CLASS    HOSTS                                                                           ADDRESS                                                                         PORTS   AGE
appserver   <none>   a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com   a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com   80      7m10s
```

After few seconds you should be able to access WebFOCUS GUI using URL `http://a353514927db445ffb7dc73a8b000a9f-49af96b79da480fd.elb.us-west-2.amazonaws.com/webfocus/signin` obviously your URL will be different (hostname from step 13)  