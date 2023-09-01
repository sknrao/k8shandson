# Kubernetes Tutorial

## Installation

### Kubeadm
#### On All Nodes
```sh
#!/bin/bash
#
# Common setup for all servers (Control Plane and Nodes)

set -euxo pipefail

# Variable Declaration

KUBERNETES_VERSION="1.26.1-00"

# disable swap
sudo swapoff -a

# keeps the swaf off during reboot
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
sudo apt-get update -y


# Install CRI-O Runtime

OS="xUbuntu_22.04"

VERSION="$(echo ${KUBERNETES_VERSION} | grep -oE '[0-9]+\.[0-9]+')"

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:"$VERSION".list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:"$VERSION"/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:1.26/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_22.04/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
sudo apt-get update
sudo apt-get install cri-o cri-o-runc -y

sudo systemctl daemon-reload
sudo systemctl enable crio --now

echo "CRI runtime installed susccessfully"

# Install kubelet, kubectl and Kubeadm

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet="$KUBERNETES_VERSION" kubectl="$KUBERNETES_VERSION" kubeadm="$KUBERNETES_VERSION"
sudo apt-get update -y
sudo apt-get install -y jq

local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```
#### On Master Nodes

```sh
#!/bin/bash
#
# Setup for Control Plane (Master) servers

set -euxo pipefail

# If you need public access to API server using the servers Public IP adress, change PUBLIC_IP_ACCESS to true.

PUBLIC_IP_ACCESS="false"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"

# Pull required images

sudo kubeadm config images pull

# Initialize kubeadm based on PUBLIC_IP_ACCESS

if [[ "$PUBLIC_IP_ACCESS" == "false" ]]; then
    
    MASTER_PRIVATE_IP=$(ip addr show eth0 | awk '/inet / {print $2}' | cut -d/ -f1)
    sudo kubeadm init --apiserver-advertise-address="$MASTER_PRIVATE_IP" --apiserver-cert-extra-sans="$MASTER_PRIVATE_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

elif [[ "$PUBLIC_IP_ACCESS" == "true" ]]; then

    MASTER_PUBLIC_IP=$(curl ifconfig.me && echo "")
    sudo kubeadm init --control-plane-endpoint="$MASTER_PUBLIC_IP" --apiserver-cert-extra-sans="$MASTER_PUBLIC_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

else
    echo "Error: MASTER_PUBLIC_IP has an invalid value: $PUBLIC_IP_ACCESS"
    exit 1
fi

# Configure kubeconfig

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Install Claico Network Plugin Network 

curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

#### On Worker Nodes
Just run the kubeadm join command.

### Minikube

Download and install minikube binary: [Directions](https://minikube.sigs.k8s.io/docs/start/)

## Health-Check

```sh
kubectl get nodes -o wide
NAME       STATUS   ROLES           AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
minikube   Ready    control-plane   3d7h   v1.27.4   192.168.49.2   <none>        Ubuntu 22.04.2 LTS   6.2.0-26-generic   docker://24.0.4

kubectl get pods -ALL
kubectl describe node minikube
```

## Managing Pods and Deployments

### Imperative Commands
When using imperative commands, a user operates directly on live objects in a cluster. The user provides operations to the kubectl command as arguments or flags. This is the recommended way to get started or to run a one-off task in a cluster. Because this technique operates directly on live objects, it provides no history of previous configurations.

```sh
# Create a new pod and run a container (ghost) 
kubectl run ghost --image=ghost
# kubectl run was updated to only create pods and it lost its deployment-specific options 
# you can use create command for deployments
kubectl create deployment nginx --image nginx
# Use for other objects
kubectl create service nodeport <myservicename>
```

Other imperative commands are

1. expose: Create a new Service object to load balance traffic across Pods.
2. autoscale: Create a new Autoscaler object to automatically horizontally scale a controller, such as a Deployment.
3. scale: Horizontally scale a controller to add or remove Pods by updating the replica count of the controller.
4. annotate: Add or remove an annotation from an object.
5. label: Add or remove a label from an object.
6. edit: Directly edit the raw configuration of a live object by opening its configuration in an editor.
7. patch: Directly modify specific fields of a live object by using a patch string. For more details on patch strings
8. get, describe, logs all fall here..

### Imperative object configuration
n imperative object configuration, the kubectl command specifies the operation (create, replace, etc.), optional flags and at least one file name. The file specified must contain a full definition of the object in YAML or JSON format.

```yaml
# redis.yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: redis
spec:
  containers:
    - name: redis
      image: redis
```
```sh
kubectl create -f redis.yaml
```
You can *delete*, *get* or *replace* too.

### Declarative object configuration
When using declarative object configuration, a user operates on object configuration files stored locally, however the user does not define the operations to be taken on the files. Create, update, and delete operations are automatically detected per-object by kubectl. This enables working on directories, where different operations might be needed for different objects.
This sets the kubectl.kubernetes.io/last-applied-configuration: '{...}' annotation on each object. The annotation contains the contents of the object configuration file that was used to create the object.
```sh
kubectl apply -f <directory>
```

```yaml
deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
```sh
kubectl diff -f deployment.yaml
kubectl apply -f deployment.yaml
# In this get output you will that annotation was written to the
# live configuration, and it matches the configuration file
kubectl get -f deployment.yaml -o yaml
```


## Service

### ClusterIP and NodePort

First deploy the Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  labels:
    tier: frontend
spec:
  containers:
  - name: webserver
    image: nginx:latest
    ports:
    - containerPort: 80
```
Then Deploy ClusterIP service.
```yaml
kind: Service
apiVersion: v1
metadata:
  name: sampleweb
spec:
  selector:
    tier: frontend
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 80
```
Check for the clusterIP address that was assigned to sampleweb service

```sh
kubectl get svc

NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
sampleweb   ClusterIP   10.110.166.211   <none>        8080/TCP         11m     tier=frontend

```
Then try to run curl (`curl 10.110.166.211:8080`) from (a) Outside the minikue (b) Inside the minikube, *after minikube ssh*. (a) should fail, whereas (b) should work - because with (b) you are accessing the service from inside the cluster.

Deploy NodePort service
```yaml
kind: Service
apiVersion: v1
metadata:
  name: sampleweb
spec:
  selector:
    tier: frontend
  type: NodePort
  ports:
    - port: 8080
      targetPort: 80
```
Now with NodePort service deployed, check the services.

```sh
kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
sampleweb   ClusterIP   10.110.166.211   <none>        8080/TCP         11m     tier=frontend
sampleweb1    NodePort    10.99.48.194     <none>        8080:30909/TCP   9m28s   tier=frontend
```

Then try to curl

1. 10.99.48.194:30909 from inside minikube
2. 10.99.48.194:8080 from inside minikube
3. 192.168.49.2:30909 from outside minikube

1 should fail and 3 and 2 should work. The port of Nodeport with clusterIP will not work.


## Ingress
[Online Exercise](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)

### First deploy the ingress controller.
The ingress controller is based on nginx, which also needs a default-backed. The default backend is a service which handles all URL paths and hosts the Ingress-NGINX controller doesn't understand (i.e., all the requests that are not mapped with an Ingress).
Basically a default backend exposes two URLs:
    
    1. /healthz that returns 200
    2. / that returns 404

That can be deployed with the below specification - backend.yaml

```yaml
#https://github.com/kubernetes/contrib/blob/master/ingress/controllers/nginx/examples/default-backend.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: default-http-backend
spec:
  replicas: 1
  selector:
    app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60      
      containers:
      - name: default-http-backend
        # Any image is permissable as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---
# create a service for the default backend
apiVersion: v1
kind: Service
metadata:
  labels:
    app: default-http-backend
  name: default-http-backend
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: default-http-backend
  sessionAffinity: None
  type: ClusterIP
---
# Replication controller for the load balancer
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-lb
spec:
  replicas: 1
  selector:
    k8s-app: nginx-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-lb
        name: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60      
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.8.2
        name: nginx-ingress-lb
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10249
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        # use downward API
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        args:
        - /nginx-ingress-controller
        - --default-backend-service=default/default-http-backend
```

Next Deploy a nginx pod, a clusterIP service and an ingress

```yaml
# nginx app
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
---
# nginx service
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
---
# Create the ingress resource
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
  - host: nginx.192.168.49.2.nip.io
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```

This Ingress rule is used by the Ingress controller (started by the ​backend.yaml​ manifest) to re-configure the nginx proxy running on the head node (in our case, minikube​). The rule will proxy requests for host ​nginx.192.168.49.2.nip.io​ to the internal service called nginx​.We use the ​nip.io​ service. It is a wildcard DNS service that is very handy for testing. It will resolve
nginx.192.168.49.2.nip.io​ to ​192.168.49.2​ the IP of minikube. Note that you may
need to edit the Ingress rule manifest if the IP of your minikube is different.
Once the rule is implemented by the controller (could take O(10) s), open your browser at nginx.192.168.49.2.nip.io​ and enjoy nginx​.

## Daemonset, Scheduling


## Custom Resources



## Helm