# Using Nginx as LoadBalancer in Kubernetes NodePort
In this demo we will see how to use Nginx as a Loabalancer in Kubernetes Nodeport. We will install kubernetes cluster manually where there will a master and two worker. In this cluster we will deploy Nginx using NodePort. We will access that using a seperate Nginx which will act as a loadbalancer.

This whole demo is tested in AWS. So there will be 04 (four) EC2 instances. One will be K8 master (K8-master), two will be worker (k8-worker1 and k8-worker2) and other one will be Nginx LB (nginx-lb).

## Step1: Install kubernetes manually  

Run below commands one by one in 03 (three) EC2 instances. 

```
# Update and Upgrade each instances
sudo apt update && sudo apt upgrade -y

# Install docker in each instances
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Adding Kubernetese Repository
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo nano /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main

sudo apt update

# Installing Kubeadm 
sudo apt-get install -y kubelet=1.20.2-00 kubeadm=1.20.2-00 kubectl=1.20.2-00

# The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.
sudo apt-mark hold kubelet kubeadm kubectl

# swapoff for better performance 
sudo swapoff -a

# Run below command only in master node (k8-master)
sudo kubeadm init

# Run below command only in worker nodes (k8-worker1 and k8-worker2). You will get the token_id and fingerprint from master.
sudo kubeadm join <master-node-ip>:6443 --token <token_id> \
--discovery-token-ca-cert-hash sha256:<fingerprint>


# Configure Overlay Network inside the K8s Cluster on Master node. We will use WeaveNet here
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# Now Run below commands in master node to see if the cluster is ready or not
kubectl get nodes
kubectl get pods -A

```

## Step2: NodePort Deployment

Below is the content of the deployment file (nginx-deploy.yaml)
```
# Deployment
# controllers/nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: nginx-deployment
 labels:
  app: nginx-app
spec:
  replicas: 1 # Only one pod will be created
  selector:
   matchLabels:
    app: nginx-app
  template:
   metadata:
    labels:
     app: nginx-app
   spec:
    containers:
    - name: nginx-container
      image: nginx:1.7.9
      ports:
      - containerPort: 80

```

Deploy above file using below command in k8-master

```
kubectl create -f nginx-deploy.yaml
```

## Step3: Create NodePort Service
Content of the service file (nginx-service.yaml) is below
```
# Service
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
 name: nginx-service
 labels:
  app: nginx-app
spec:
 selector:
  app: nginx-app
 type: NodePort
 ports:
 - nodePort: 31000
   port: 80
   targetPort: 80
```

Create the service using below command in k8-master
```
kubectl create -f nginx-service.yaml
```

### Checking the Nodeport Deployment

Use below command to check whether the deployment is ok or not:
```
kubectl get service # Will give you the service IP
kubectl get pode -o wide # Will give you the pod IP
kubectl describe svc nginx-service
```


### Get the Node IP
The Node IP will be the IP of each worker.

## Step4: Configure Nginx LB
In the 4th EC2 instance first update and upgrade and then install nginx.

```
sudo apt update && sudo apt upgrade -y
sudo apt install nginx
```
We need to add below lines in nginx.conf file. We will use upstream directive for this. See nginx.conf file in the repo to see the whole content.

```#Load balancing IP
    
    upstream lb0 {
        server <Node_IP>:3100; #IP of node1
        server <Node_IP>:3100; #IP of node2
     
    }
    
    server {
        location / {
            proxy_pass http://lb0;
        }
    }
```

## Cleanup NodePort
Below Command will delete the service. Here service name is nginx-service.
```
kubectl delete svc nginx-service
```
