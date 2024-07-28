# Deployment process

<br>

## Kubernetes manifests for deploying the application.

##### <span style="color: red;">NOTE: All manifests have been succesfully tested on AWS EKS cluster.</span>

#### 1) [Fronend App](Frontend-manifests/)
#### 2) [Backend App](Backend-manifests/)
#### 3) [MongoDB](MongoDB-manifests/)
#### 4) [RabbitMQ](RabbitMq-manifests/)

<br>

## Deployment process.

<br>

### Prerequisites and Assumptions.

#### 1) AWS EKS Cluster and EKS Node Group.
#### 2) Install a EBS CSI Driver Addon on EKS cluster.
#### 3) Install a Loadbalancer controller on EKS cluster ([Ref](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html)).
#### 4) Tag Subnets with the following tags which were used to create EKS cluster and EKS Nodegroup.
```bash
    kubernetes.io/role/elb: 1
    kubernetes.io/cluster/${cluster_name}: shared
```
#### 5) Install a metric server in EKS cluster.
#### 6) Install cluster autoscaler in EKS cluster ([Ref](https://community.aws/content/2a9qUKMTGUM6DkFdi0dNwtQnAke/cluster-autoscaler-configure-using-aws-eks--1-24?lang=en))

<br>

### MongoDB deployment.

<br>

#### 1) Apply all the [manifest](MongoDB-manifests) using below command. 
```bash
    kubectl apply -f MongoDB-manifests/
```
#### 2) Initiate the replication in Mongo pods using below command.
```bash
    kubectl exec -it -n mongodb mongodb-0 -- mongosh

    rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "mongodb-0.mongodb:27017", priority: 2 },
        { _id: 1, host: "mongodb-1.mongodb:27017", priority: 1 },
        { _id: 2, host: "mongodb-2.mongodb:27017", priority: 1 }
      ]
    })
```
#### 3) We have configure a network policy which allows only backend deployment to connect to mongodb pods.

<br>

### RabbitMQ deployment.

<br>

#### 1) Apply all the [manifest](RabbitMq-manifests) using below command. 
```bash
    kubectl apply -f RabbitMq-manifests/
```
#### 2) We have also configure a LoadBalancer type service for RabbitMQ Deployment so that we can access RabbitmQ GUI.
#### 3) We are passing rabbitmq configuration using a Configmap resource.

<br>

### Frontend Deployment.

<br>

#### 1) Apply all the [manifest](Frontend-manifests) using below command. 
```bash
    kubectl apply -f Frontend-manifests/
```
#### 2) We have already install Load Balancer controller in our EKS cluster and we created a LoadBalancer type service to expose our Frontend to internet.
#### 3) We have configure HPA and Pod Distrubtion Budget for our Frontend Deployment.
#### 4) We are using a ConfigMap for nginx configuration.

<br>

### Backend Deployment.

<br>

#### 1) Apply all the [manifest](Backend-manifests) using below command. 
```bash
    kubectl apply -f Backend-manifests/
```
#### 2) We have a apply a network policy which only allows Frontend pod to connect to Backend pod.
#### 3) We have configure HPA and Pod Distrubtion Budget for our Frontend Deployment.
#### 4) We are using a Configmap and secret for defining runtime variable for our backend deployment.

<br><br>

## Troubleshooting steps.

<br>

### Frontend Accessibility

<br>

#### The frontend service is not accessible externally post-deployment.

<br>

#### 1) After deploying all the frontend manifests, make sure all pods are healthy.
```bash
    kubectl get pods -n frontend
```
#### 2) If any pod is in crash or in other state apart from running then check the logs and describe the events.
```bash
    kubectl logs -f -n frontend <pod_name>
    kubectl describe pod -n frontend <pod_name>
```
#### 3) Check the service status of frontend which is a Loadbalancer type service.
```bash
    kubectl describe svc -n frontend frontend
```
#### 4) If service was not able to create the AWS ELB then check the logs of ELB controller pod. Common known issues are appropriate tagging in VPC subnets or Permission issues to ELB controller role.
```bash
    kubectl logs -f -n kube-system <pod_name>
```

<br>

#### Intermittent Backend-Database Connectivity

<br>

#### 1) Most probably this is not the issue with the network policy as if it was then the whole connectivity would have been impacted and not the intermittent connectivity issues.
#### 2) We need to make sure which urls we are using for interacting with mongoDB deployment. For write related querries we need use mongodb-0.mongodb:27017 and for all read related querries we can use oother endpoints mongodb-1.mongodb:27017/mongodb-2.mongodb:27017
#### 3) We need to check the memory and cpu utilisation of mongodb pods from grafana GUI and if they are using high resources then we need to vertically scale those deployment by increasing the limits in resources section of manifest.

<br>

#### Order Processing Delays

<br>

#### 1) Check the available messages in the queue by visiting the GUI of rabbitmq.
#### 2) If there are high no. of messages than expected then make sure consumer app is not facing any slowness and check the utilisation and logs of the consumer app find any issues.
#### 3) Scale the Consumer application to fasten the processing of the messages in the RabbitMQ queue.
#### 4) Check the resource utilisation of the rabbitmq pods and if possible then increase the limit resources of the rabbitmq deployment.

