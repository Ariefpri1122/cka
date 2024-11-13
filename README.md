# CKA

CKA training notes.

# Architecture

- Control Plane:
  - kube-apiserver: Exposes the Kubernetes API, pass data to responsible component.
  - etcd: Stores all API server data.
  - kube-scheduler: Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.
  - kube-controller-manager: Implement Kubernetes API behavior.
  - cloud-controller-manager: Integrates with underlying cloud provider(s).
- Node:
  - Container Runtime (containerd, cri-o, etc.): Run containers.
  - kubelet: Ensures that Pods are running with Container Runtime.
  - kube-proxy: Maintains network rules on nodes to implement Service.

# Installation

## Common configuration

### Modify system settings

```bash
# enable ip forwarding
cat <<EOF | sudo tee /etc/sysctl.d/10-ipv4-forward.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# disable memory swap
sudo swapoff -a
```

### Install containerd

```bash
# add tools
sudo apt-get clean
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y

# add containerd repository
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install containerd
sudo apt-get update && sudo apt-get install containerd.io -y
```

### Configure containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

### Install kubernetes tools

```bash
# add kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install kubernetes tools
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Controller configuration

### Init cluster

```bash
sudo kubeadm init
```

### Configure easy access to cluster

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
source <(kubectl completion bash)
```

### Check cluster

```bash
kubectl get node
kubectl get pods -n kube-system
```

### Generate join token

```bash
kubeadm token create --print-join-command
```

## Worker configuration

### Join cluster

```bash
sudo kubeadm join $CONTROLLER_IP:6443 --token $JOIN_TOKEN --discovery-token-ca-cert-hash $CA_CERT_HASH
```

## CNI installation (controller)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/calico.yaml -O
kubectl apply -f calico.yaml
kubectl get pods -n kube-system | grep calico
```

# Workload & Scheduling

## Deployment

- Manage a set of Pods to run an application workload, usually stateless application.
- Stateless = can be moved between nodes easily.

### Create Deployment

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 5
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - name: kubeapp
        image: kubenesia/kubeapp:1.0.0
EOF

kubectl create deployment kubeapp --image=kubenesia/kubeapp:1.0.0 --dry-run=client --output=yaml > kubeapp.yaml

kubectl apply -f kubeapp.yaml
kubectl get deployment
kubectl get replicaset
kubectl get pods
kubectl describe pods
kubectl get pods --show-labels
kubectl port-forward deployment/kubeapp --address 0.0.0.0 8080:8000
```

### Scale Deployment

```bash
kubectl scale deployment kubeapp --replicas=4
```

### Describe Deployment

```bash
kubectl describe deploy kubeapp
```

### Deployment rolling update

```bash
kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
watch kubectl get pod -l app=kubeapp
```

### Deployment rolling update maxUnavailable parameter

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: kubeapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - image: kubenesia/kubeapp:1.0.0
          name: kubeapp
EOF

kubectl apply -f kubeapp.yaml
kubectl get pods

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
kubectl get pods
```

### Deployment rollout history

```bash
kubectl rollout status deployment kubeapp
kubectl rollout history deployment kubeapp

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.2.0

kubectl rollout status deployment kubeapp
kubectl rollout history deployment kubeapp
```

### Deployment recreate strategy

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: kubeapp
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - image: kubenesia/kubeapp:1.0.0
          name: kubeapp
EOF

kubectl apply -f kubeapp.yaml
kubectl get pods

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
kubectl get pods
```

### Get container image of a Deployment

```bash
kubectl describe deployment kubeapp | grep Image
kubectl get deployment kubeapp -o yaml | yq '.spec.template.spec.containers[].image'
```

### Rollback Deployment

```bash
kubectl rollout undo deployment kubeapp
kubectl rollout undo deployment kubeapp --to-revision=$REVISION_NUMBER
```

### Startup Probe

Used to protect slow starting containers from serving traffic when they are not ready.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 6000
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard

sed -i 's/6000/8080/g' kuard.yaml
kubectl apply -f kuard.yaml
kubectl describe pods -l app=kuard
```

### Liveness Probe

Used to check the health of the Pod periodically.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 8080
        livenessProbe:
          httpGet:
            path: /healthy
            port: 8080
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard
```

### Review

- Try to create a Deployment with image `gcr.io/kuar-demo/kuard-amd64:blue` with 5 replicas and access port 8080 of the container.
- Try to create a Deployment with the following spec:
  - Name: `web`
  - Two images: `nginx:1.27.2` and `gcr.io/kuar-demo/kuard-amd64:blue`
  - Replica: 10
  - Startup probe to nginx
  - Liveness probe to kuard

## DaemonSet

Ensure all nodes run a copy of a Pod. Usually used for operational tools or add-ons.

### Create DaemonSet

```bash
cat <<EOF >fluentd-elasticsearch-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      # it may be desirable to set a high priority class to ensure that a DaemonSet Pod
      # preempts running Pods
      # priorityClassName: important
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF

kubectl apply -f fluentd-elasticsearch-daemonset.yaml
kubectl -n kube-system get ds fluentd-elasticsearch
kubectl -n kube-system get ds fluentd-elasticsearch -o wide
kubectl -n kube-system get pods -o wide | grep fluentd
kubectl delete -f fluentd-elasticsearch.yaml
```

### Review

- Create a DaemonSet with image `nginx:1.27.2`, make sure it doesn't scheduled on controller node.
- Create a DaemonSet with image `httpd:2.4.62`, make sure it scheduled on all nodes including the controller.
- Create a DaemonSet with image `caddy:2.8.4`, schedule only on nodes with label role=web.

## StatefulSet

Like Deployment, but used to deploy stateful application. Maintain a sticky identity for each Pod.

### Create StatefulSet

```bash
cat <<EOF >mysql-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
      name: mysql
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.4.2
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "P@ssw0rd"
        - name: MYSQL_DATABASE
          value: "mydb"
        - name: MYSQL_USER
          value: "user"
        - name: MYSQL_PASSWORD
          value: "P@ssw0rd"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        hostPath:
          path: /data/mysql
          type: DirectoryOrCreate
EOF

kubectl apply -f mysql-statefulset.yaml
kubectl get sts -o wide
```

### Create database

```bash
kubectl exec -ti mysql-0 -- bash
mysql -h 127.0.0.1 -u root -p mydb
select version();
create table users (id int primary key, email text);
show tables;
exit
```

### Modify database version

```bash
sed -i 's/8.4.2/8.4.3/g' mysql-statefulset.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl exec -ti mysql-0 -- bash
mysql -h 127.0.0.1 -u root -p mydb
select version();
show tables;
```

### Review

- Try to create a StatefulSet for postgres.
- Check [here](https://hub.docker.com/_/postgres) for information related to the container image.

## ConfigMap

### Create ConfigMap

```bash
cat <<EOF >kubeapp-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeapp
data:
  MODE: development
  COLOR: blue
  SERVICE: kubeapp
EOF

kubectl apply -f kubeapp-configmap.yaml
kubectl get cm kubeapp
kubectl get cm kubeapp -o yaml
```

### ConfigMap as env variable

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        envFrom:
          - configMapRef:
              name: kubeapp
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
printenv | egrep 'COLOR=|MODE=|SERVICE='
exit
```

### ConfigMap as file

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: kubeapp
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config
exit
```

## Secret

### Create Secret

```bash
kubectl create secret generic kubeapp --from-literal=MODE=development --from-literal=COLOR=blue
kubectl get secret
kubectl get secret kubeapp -o yaml
```

### Secret as env variable

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        envFrom:
          - secretRef:
              name: kubeapp
EOF

kubectl exec -ti deployment/kubeapp -- sh
printenv | egrep 'COLOR=|MODE=|SERVICE='
exit
```

### Secret as file

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: kubeapp
EOF

kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config
exit
```

### Review

- Create a `postgres` StatefulSet as before, but use a `Secret` to configure `POSTGRES_PASSWORD`.
- Check with `kubectl exec -ti postgres-0 -- psql -h 127.0.0.1 -U postgres`

## Node selector

- Simplest way to add a constraint for node selection by using node labels as criteria.

### Add label to node

```bash
kubectl label node worker1 disk=hdd
kubectl label node worker2 disk=ssd
```

### Show labels

```bash
kubectl label node worker1 --list
kubectl label node worker2 --list
kubectl get nodes --show-labels
```

### Remove label

```bash
kubectl label node worker1 disk-
kubectl label node worker2 disk-
```

### Pod with nodeSelector

- Pod will be scheduled to worker2.

```bash
cat <<EOF >pod-myapp-ssd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-ssd
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    disk: ssd
EOF

kubectl apply -f pod-myapp-ssd.yaml
kubectl get nodes -l disk=ssd
kubectl get pods -o wide
```

- Try to remove the label from worker2 and recreate the pod, it will stuck in "Pending" state.

## Taints & Tolerations

- Taints: allow node to repel or "kick" a set of pods.
- Tolerations: allow pods to "bypass" taints.

### Node selector + Taints + Tolerations

```bash
kubectl taint nodes worker1 role=gpu:NoSchedule
kubectl describe node worker1 | grep Taint

cat <<EOF >pod-myapp-gpu-taints-tolerations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-gpu-taints-tolerations
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: role
    operator: Equal
    value: gpu
    effect: NoSchedule
EOF

kubectl apply -f pod-myapp-gpu-taints-tolerations.yaml
kubectl get pods -o wide
```

### Node selector + Taints without Tolerations

```bash
cat <<EOF >pod-myapp-gpu-without-tolerations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-gpu-without-tolerations
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    disk: hdd
EOF

kubectl apply -f pod-myapp-gpu-without-tolerations.yaml
kubectl get pods -o wide

# Remove taint
kubectl taint nodes worker1 role-
```

## Pod affinity & anti-affinity

- Used to constraint inter-pod placement.
- Use `kubectl explain deployment.spec.template.spec.affinity` to check for available fields.

### Pod anti-affinity

- Prevent pod to be scheduled alongside of specific pods.

```bash
cat <<EOF >cache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  selector:
    matchLabels:
      app: cache
  replicas: 2
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: redis
        image: redis
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
EOF

kubectl apply -f cache.yaml
kubectl get pods -o wide

kubectl scale deployment cache --replicas=4
kubectl scale deployment cache --replicas=2
```

### Pod anti-affinity + affinity

- Schedule pod alongside of specific pods while also preventing it to be placed with some other pods.

```bash
cat <<EOF >webserver.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
spec:
  selector:
    matchLabels:
      app: webserver
  replicas: 2
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - name: webserver
        image: nginx
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - webserver
            topologyKey: "kubernetes.io/hostname"
EOF

kubectl apply -f webserver.yaml
kubectl get pods -o wide
kubectl scale deployment webserver --replicas=3
kubectl get pods -o wide
```

### Pod anti-affinity soft rule

- Schedule the pods even if it can't find a matching node.

```bash
cat <<EOF >webserver-pref.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver-pref
spec:
  selector:
    matchLabels:
      app: webserver-pref
  replicas: 3
  template:
    metadata:
      labels:
        app: webserver-pref
    spec:
      containers:
      - name: web-app
        image: nginx
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - webserver-pref
              topologyKey: "kubernetes.io/hostname"
EOF

kubectl apply -f webserver-pref.yaml
kubectl get pod
kubectl scale deploy webserver-pref --replicas=3
kubectl get pod
```

## Static Pod

- Manually run a pod on a specific node, bypassing the scheduler.

```bash
ssh $WORKER
cat <<EOF | sudo tee /etc/kubernetes/manifests/pod-static.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-static
spec:
  containers:
  - name: nginx
    image: nginx
EOF

ls /etc/kubernetes/manifests/

ssh $CONTROLLER
kubectl get pods -o wide
```

# Resource requests and limits

- Used to control the resource usage (CPU and memory) of a Pod.
- Best practice: do not use CPU limit to prevent throttling: (https://home.robusta.dev/blog/stop-using-cpu-limits)
- Always define resource requests
  - Easier capacity planning
  - Less surprise

## Create Pod with resource requests and limits

```bash
cat <<EOF >pod-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeapp-requests-limits
spec:
  containers:
  - name: kubeapp
    image: kubenesia/kubeapp
    resources:
      requests:
        memory: 128Mi
        cpu: 250m
      limits:
        memory: 256Mi
EOF

kubectl apply -f pod-requests-limits.yaml

kubectl get pods
kubectl describe pods

kubectl describe node worker1
kubectl describe node worker2
```

## Create Pod with bad resource requests and limits

- If resource requests can't be fulfilled by any node, then Pod will stuck in Pending state.

```bash
cat <<EOF >pod-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeapp-requests-limits
spec:
  containers:
  - name: kubeapp
    image: kubenesia/kubeapp:1.2.0
    resources:
      requests:
        memory: 128Gi
        cpu: 250m
      limits:
        memory: 256Gi
        cpu: 500m
EOF

kubectl apply -f pod-requests-limits.yaml

kubectl get pods
kubectl describe pods

kubectl describe node worker1
kubectl describe node worker2

kubectl delete pod kubeapp-requests-limits
```

# Autoscaling

- Change replica number of a Deployment based on resource usage

## Install metrics-server

```bash
curl -fsSLo metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
vim metrics-server.yaml
# add --kubelet-insecure-tls under args
kubectl apply -f metrics-server.yaml
kubectl -n kube-system get pods
kubectl top nodes
kubectl top pods
kubectl describe nodes k8s-worker1-trainer
```

## Create Deployment + HPA

```bash
kubectl delete pods --all
kubectl delete deployment --all

cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - name: kubeapp
        image: kubenesia/kubeapp:1.2.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

kubectl apply -f kubeapp.yaml
kubectl autoscale deployment kubeapp --min=2 --max=10 --cpu-percent=50 --force
kubectl get hpa
```

## Load test

```bash
sudo apt install apache2-utils -y
kubectl port-forward deployment/kubeapp --address 0.0.0.0 8080:8000

watch kubectl get hpa

ab -n 100000 -c 1000 http://$PUBLIC_IP:8080/
```

# Services & Networking

- Containers inside the same pod can communicate via `localhost`.
- Each pod in a cluster gets its own unique IP address.
- All pods can communicate with all other pods by default.

## Service

- Service provides a stable IP address or hostname instead of manually using IP of pods.

### ClusterIP

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - name: kubeapp
          image: kubenesia/kubeapp:1.2.0
          ports:
            - name: http
              containerPort: 8000
EOF

cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  type: ClusterIP
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-deployment.yaml -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp
kubectl get ep
kubectl describe ep
kubectl get pods -o wide
kubectl get pods -l app=kubeapp

kubectl run test -ti --rm --image=kubenesia/kubebox -- sh
curl kubeapp
nslookup kubeapp
exit
```

### ExternalName

```bash
cat <<EOF >get-ip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: get-ip
spec:
  type: ExternalName
  externalName: google.com
EOF

kubectl apply -f get-ip-service.yaml
kubectl get svc
kubectl describe svc get-ip

kubectl run test -it --rm --image=kubenesia/kubebox -- sh
curl get-ip
nslookup get-ip
exit
```

### NodePort

```bash
cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
spec:
  type: NodePort
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp

export PUBLIC_IP=$(curl -s icanhazip.com)
export NODE_PORT=$(kubectl get svc kubeapp-service -o yaml | yq '.spec.ports[0].nodePort')
curl "$PUBLIC_IP:$NODE_PORT"
```

### Headless Service

- Used to bypass the cluster-wide IP address, name will be resolved directly to pod IPs.

```bash
cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp

kubectl run test -it --rm --image=kubenesia/kubebox -- sh
curl kubeapp
nslookup kubeapp
exit
```

### Review

- Run the following commands:

```bash
cat <<EOF >nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    application: nginx
    role: web
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
EOF
kubectl apply -f nginx-service.yaml
kubectl create deployment nginx --image=nginx:1.27.2
kubectl port-forward svc/nginx --address 0.0.0.0 1234:8080
curl localhost:1234
```

- Does it work? If not, what's wrong?

## Ingress

- Route HTTP/HTTPS traffic into cluster workloads.

### Install ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Create Deployment & Service

```bash
kubectl create deployment blue --image=gcr.io/kuar-demo/kuard-amd64:blue --port=8080
kubectl create deployment green --image=gcr.io/kuar-demo/kuard-amd64:green --port=8080

kubectl expose deployment blue --port=80 --target-port=8080
kubectl expose deployment green --port=80 --target-port=8080
```

### Route by host header

```bash
kubectl create ingress blue --class=nginx --rule="blue.$PUBLIC_IP.sslip.io/*=blue:80"
kubectl create ingress green --class=nginx --rule="green.$PUBLIC_IP.sslip.io/*=green:80"

kubectl get ingress
kubectl get ingress blue -o yaml
kubectl get ingress green -o yaml

export NODE_PORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o yaml | yq '.spec.ports[0].nodePort')
echo $NODE_PORT

curl http://blue.$PUBLIC_IP.sslip.io:$NODE_PORT
curl http://green.$PUBLIC_IP.sslip.io:$NODE_PORT
```

### Route by path

````bash
cat <<EOF >blue-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: blue
            port:
              number: 80
        path: /blue
        pathType: Prefix
EOF

kubectl apply -f blue-ingress.yaml

```bash
cat <<EOF >green-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: green
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: green
            port:
              number: 80
        path: /green
        pathType: Prefix
EOF

kubectl apply -f green-ingress.yaml

curl http://$PUBLIC_IP.sslip.io:$NODE_PORT/blue
curl http://$PUBLIC_IP.sslip.io:$NODE_PORT/green
````

### Review

- Deploy kubeapp application
  - Use image: `kubenesia/kubeapp:1.2.0`.
  - Minimum CPU is 0.5 core.
  - Minimum memory is 128 MiB.
  - Scale based on CPU utilization with maximum replicas of 10.
  - Make it accessible on `kubeapp.$PUBLIC_IP.sslip.io`.

## Gateway

### Install Gateway controller implementation

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcontour/contour/release-1.23/examples/gateway/00-crds.yaml
```

### Create GatewayClass

```bash
kubectl apply -f - <<EOF
kind: GatewayClass
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: contour
spec:
  controllerName: projectcontour.io/gateway-controller
EOF
```

### Create Gateway

```bash
kubectl apply -f - <<EOF
kind: Namespace
apiVersion: v1
metadata:
  name: projectcontour
---
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: contour
  namespace: projectcontour
spec:
  gatewayClassName: contour
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
EOF

kubectl apply -f https://projectcontour.io/quickstart/contour.yaml

kubectl apply -f - <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: contour
  namespace: projectcontour
data:
  contour.yaml: |
    gateway:
      gatewayRef:
        name: contour
        namespace: projectcontour
EOF

kubectl -n projectcontour rollout restart deployment/contour

export NODE_PORT=$(kubectl -n projectcontour get svc envoy -o yaml | yq '.spec.ports[0].nodePort')
```

### Route by host header

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: blue
  labels:
    app: blue
spec:
  hostnames:
  - blue.$PUBLIC_IP.sslip.io
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: blue
      port: 80
EOF

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: green
  labels:
    app: green
spec:
  hostnames:
  - green.$PUBLIC_IP.sslip.io
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: green
      port: 80
EOF

curl http://$PUBLIC_IP.sslip.io:$NODE_PORT/blue
curl http://$PUBLIC_IP.sslip.io:$NODE_PORT/green
```

### Route by path

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: blue
  labels:
    app: blue
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /blue
    backendRefs:
    - kind: Service
      name: blue
      port: 80
EOF

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: green
  labels:
    app: green
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /green
    backendRefs:
    - kind: Service
      name: green
      port: 80
EOF

curl $PUBLIC_IP.sslip.io/blue
curl $PUBLIC_IP.sslip.io/green
```

### Weight

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: weight
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /weight
    backendRefs:
    - kind: Service
      name: blue
      port: 80
      weight: 50
    - kind: Service
      name: green
      port: 80
      weight: 50
EOF

curl -I $PUBLIC_IP:$NODE_PORT/weight
curl -I $PUBLIC_IP:$NODE_PORT/weight
```

## Network Policy

```bash
# Create target workload
kubectl create deployment kubeapp --image=kubenesia/kubeapp:1.2.0 --port=8000
kubectl expose deployment kubeapp --port=80 --target-port=8000

# Create NetworkPolicy to prevent access from other namespace
cat <<EOF >netpol-deny-from-other-ns.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: default
  name: deny-from-other-namespaces
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
EOF
kubectl apply -f netpol-deny-from-other-ns.yaml

# Check NetworkPolicy
kubectl get networkpolicy

# Test access from other namespace
kubectl create ns dev
kubectl run test -it -n dev --rm --image=kubenesia/kubebox -- sh
wget -qO- --timeout=2 kubeapp.default # fail
kubectl run test -it --rm --image=kubenesia/kubebox -- sh
wget -qO- --timeout=2 kubeapp # ok

# Remove NetworkPolicy
kubectl delete netpol deny-from-other-namespaces
```

# Storage

## Share directory between containers inside a pod using emptyDir

```bash
cat <<EOF >pod-with-empty-dir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-empty-dir
spec:
  containers:
  - name: nginx
    image: nginx:1.27.2
    volumeMounts:
    - name: wwwroot
      mountPath: /usr/share/nginx/html
  - name: caddy
    image: caddy:2.8.4
    command:
      - caddy
      - file-server
      - --root
      - /var/www/html
      - --listen
      - :8080
    volumeMounts:
    - name: wwwroot
      mountPath: /var/www/html
  volumes:
  - name: wwwroot
    emptyDir:
      sizeLimit: 500Mi
EOF

kubectl apply -f pod-with-empty-dir.yaml

kubectl exec -ti -c nginx pod-with-empty-dir -- bash
echo "Test emptyDir volume with containers" >/usr/share/nginx/html/index.html
exit

kubectl port-forward pod-with-empty-dir 1234:80
curl -v localhost:1234

kubectl port-forward pod-with-empty-dir 1234:8080
curl -v localhost:1234

kubectl exec -ti -c caddy pod-with-empty-dir -- sh
echo "New page" >/var/www/html/index.html
exit

kubectl port-forward pod-with-empty-dir 1234:80
curl -v localhost:1234

kubectl port-forward pod-with-empty-dir 1234:8080
curl -v localhost:1234
```

## Set up NFS server

```bash
sudo apt-get install --yes nfs-kernel-server
cat <<EOF | sudo tee /etc/exports
/srv/nfs4 10.0.0.0/8(rw,no_subtree_check,all_squash)
EOF
sudo systemctl restart nfs-kernel-server
sudo mkdir /srv/nfs4
sudo chown 65534:65534 /srv/nfs4
```

## Static provisioning

```bash
export NFS_SERVER=$(ip -4 a | grep global | head -n 1 | awk '{print $2}' | cut -d '/' -f 1)

cat <<EOF >nginx-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx
spec:
  storageClassName: "nfs"
  capacity:
    storage: 1Gi
  accessModes:
   - ReadWriteMany
  nfs:
    server: $NFS_SERVER
    path: /srv/nfs4
EOF

kubectl apply -f nginx-pv.yaml

kubectl get pv
kubectl describe pv nginx

cat <<EOF >nginx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f nginx-pvc.yaml
kubectl get pvc
kubectl describe pvc nginx
```

## Mount volume on Pod

```bash
cat <<EOF >nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27.2
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: nginx
EOF

kubectl apply -f nginx-pod.yaml
kubectl describe pods nginx
kubectl get pods nginx -o wide

kubectl port-forward nginx 8080:80

echo "Old landing page" | sudo tee /srv/nfs4/index.html
curl localhost:8080

echo "New landing page" | sudo tee /srv/nfs4/index.html
curl localhost:8080

kubectl delete pods nginx
kubectl delete pvc nginx
kubectl delete pv nginx
```

# Security

## Create additional user

### Create certificate

```bash
# Create Certificate Signing Request
openssl req -nodes -new -keyout andy.key -out andy.csr -subj "/CN=andy/O=dev"

# Sign the CSR using Kubernetes CA
sudo openssl x509 -req -in andy.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out andy.crt -days 365

# View the signed certificate content
openssl x509 -in andy.crt -text -noout
```

### Create new kubeconfig context

```bash
# Create new context for the new user
kubectl config set-credentials andy --client-certificate=andy.crt --client-key=andy.key
kubectl config set-context andy@kubernetes --cluster=kubernetes --user=andy

# Change context
kubectl config get-contexts
kubectl config use-context andy@kubernetes

# Both commands will fail
kubectl get nodes
kubectl get pods

# View content of kubeconfig
cat ~/.kube/config
```

### Create ClusterRole

```bash
kubectl config use-context kubernetes-admin@kubernetes

cat <<EOF >andy-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: andy
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
EOF

kubectl apply -f andy-clusterrole.yaml
```

### Create ClusterRoleBinding

```bash
cat <<EOF >andy-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: andy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: andy
subjects:
  - kind: User
    name: andy
EOF

kubectl apply -f andy-clusterrolebinding.yaml
```

## Test RBAC (ClusterRole)

```bash
kubectl config use-context andy@kubernetes

# should success
kubectl get pods -n kubesystem
kubectl get svc -n kube-system

# should fail
kubectl get secret -n kube-system

kubectl auth can-i get pods
kubectl auth can-i get svc

kubectl auth can-i get secrets

# back to admin
kubectl config use-context admin@kubernetes

# delete ClusterRole and ClusterRoleBinding
kubectl delete clusterrolebinding andy
kubectl delete clusterrole andy
```

## Create Role

```bash
cat <<EOF >andy-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: andy
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
EOF

kubectl apply -f andy-role.yaml
```

## Create RoleBinding

```bash
cat <<EOF >andy-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: andy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: andy
subjects:
  - kind: User
    name: andy
EOF

kubectl apply -f andy-rolebinding.yaml
```

## Test RBAC (Role)

```bash
kubectl config use-context andy@kubernetes

# should success
kubectl get pods
kubectl get svc

# should fail
kubectl get pods -n kube-system
kubectl get svc -n kube-system

# should fail
kubectl get secret

kubectl auth can-i get pods
kubectl auth can-i get svc

kubectl auth can-i get secrets

# back to admin
kubectl config use-context kubernetes-admin@kubernetes

# delete Role and RoleBinding
kubectl delete rolebinding andy
kubectl delete role andy
```

# Troubleshooting

## Debugging Pod

```bash
# list pods
kubectl get pods -o wide

# execute command on running pods
kubectl exec $POD_NAME - $COMMAND

# interactive command
kubectl exec -it $POD_NAME -- sh

# get details of pod
kubectl describe pod $POD_NAME
kubectl get pod $POD_NAME -o yaml

# show logs of pod
kubectl logs $POD_NAME
kubectl logs $POD_NAME -f
```

## Debugging Service

```bash
# list services
kubectl get svc

# get details of service
kubectl describe svc $SVC_NAME

# get list of pods based on labels
kubectl get pods -l $KEY=$VALUE
```

## Debugging cluster

```bash
# list nodes
kubectl get node

# show cluster status
kubectl cluster-info

# check log of control plane components
kubectl logs -n kube-system -l=component=kube-apiserver
kubectl logs -n kube-system -l=component=kube-scheduler
kubectl logs -n kube-system -l=component=kube-controller-manager
kubectl logs -n kube-system -l=component=etcd
sudo systemctl status kubelet
```

# Cluster maintenance

## etcd backup

```bash
# backup existing etcd manifests
sudo cp /etc/kubernetes/manifests/etcd.yaml etcd.yaml

# install client tools
sudo apt-get install etcd-client

# create ns to verify restore later
kubectl create ns before-backup

# do backup
sudo ETCDCTL_API=3 etcdctl \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save snapshot.db

kubectl delete ns before-backup

# view backup result
sudo ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshot.db
```

## etcd restore

```bash
sudo ETCDCTL_API=3 etcdctl \
    --data-dir /var/lib/etcd-from-backup \
    snapshot restore snapshot.db

sudo vim /etc/kubernetes/manifests/etcd.yaml
# replace /var/lib/etcd to /var/lib/etcd-from-backup on hostPath

# ns should be restored
kubectl get ns before-backup
```

## Version upgrade

### Controller

```bash
kubectl get nodes -o wide
sudo vim /etc/apt/sources.list.d/kubernetes.list
# set version to 1.31
sudo apt-get update && apt-cache madison kubeadm
sudo apt-get install kubeadm=1.31.2-1.1 kubelet=1.31.2-1.1 kubectl=1.31.2-1.1
sudo kubeadm upgrade apply v1.31.2 --ignore-preflight-errors all
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl get nodes -o wide
```

### Worker

```bash
ssh $MASTER
kubectl get nodes -o wide
kubectl cordon $WORKER_NODE
kubectl drain $WORKER_NODE --ignore-daemonsets --delete-emptydir-data --force

ssh $WORKER_IP
sudo vim /etc/apt/sources.list.d/kubernetes.list
# set version to 1.31
sudo apt-get update && apt-cache madison kubeadm
sudo apt-get install kubeadm=1.31.0-1.1 kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

ssh $MASTER
kubectl uncordon $WORKER_NODE
```

# Exam tips

## Example tasks

1. Create PV
2. Backup & Restore etcd
3. Troubleshoot node NotReady
4. Multi container pod with emptyDir volume
5. Create Deployment
6. Create Ingress
7. Create Pod
8. ReadinessProbe
9. Fix Service selector
10. Upgrade master node
11. Create ClusterRole + ClusterRoleBinding
12. Scale replica
13. NodeSelector
14. PVC
15. NetworkPolicy
16. Static Pod
17. Tshoot NodePort
