CKAD-notes
---
### vi shortcuts
- `dD` - Delete from cursor location to end of line
- `dd` - cut
- `yy` - copy
- `p` - past
- `u` - undo
- `ZZ` - save and exit (equivalent to :wq)
- `Esc+V` - Mark lines(then arrow keys)
- Copy marked lines: `y`
- Cut marked lines: `d`
- Past lines: `p`
- `:kn` - run kubectl explain inside vi
- `$` - end of line
- `0` - begining of the line

```
vim ~/.bashrc
# then add those two:
alias k=kubectl
alias kn='kubectl config set-context --current --namespace '
alias ka='kubectl apply -f '
alias kd='kubectl delete --grace-period 0 --force  '
alias kdd='kubectl describe '

# get current namespace
alias kns='kubectl config view --minify --output 'jsonpath={..namespace}''
```

```
vi ~/.vimrc
# then add these two lines to the file
set tabstop=2
set expandtab
set shiftwidth=2

inoremap jk <Esc>
cnoremap ke kubectl explain
```


### Kubectl run --restart flag
NEVER try to write YAML manifests by yourself. `—-dry-run -oyaml` Combine it with the `— restart` flag tip, and you have a way of generating typical manifest file without copy/pasting anything.

**Update**: `--restart` flag is removed and you should not rely on it for creating anything but the `Pod`.


```sh
kubectl run nginx --image=nginx   (deployment)
kubectl run nginx --image=nginx --restart=Never   (pod)
kubectl run nginx --image=nginx --restart=OnFailure   (job)  
kubectl run nginx --image=nginx  --restart=OnFailure --schedule="* * * * *" (cronJob)

kubectl run nginx -image=nginx --restart=Never --port=80 --namespace=myname --command --serviceaccount=mysa1 --env=HOSTNAME=local --labels=bu=finance,env=dev  --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --dry-run -o yaml - /bin/sh -c 'echo hello world'

kubectl run frontend --replicas=2 --labels=run=load-balancer-example --image=busybox  --port=8080
kubectl expose deployment frontend --type=NodePort --name=frontend-service --port=6262 --target-port=8080
kubectl set serviceaccount deployment frontend myuser
kubectl create service clusterip my-cs --tcp=5678:8080 --dry-run -o yaml


kubectl cp /tmp/foo <some-namespace>/<some-pod>:/tmp/bar
kubectl cp <some-namespace>/<some-pod>:/tmp/foo /tmp/bar

```

Unix bash on-liners: 

```sh

args: ["-c", "while true; do date >> /var/log/app.txt; sleep 5;done"]
args: [/bin/sh, -c,'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done’]
args: ["-c", "mkdir -p collect; while true; do cat /var/data/* > /collect/data.txt; sleep 10; done"]

a=10;b=5; if [ $a -le $b ]; then echo "a is small" ; else echo "b is small"; fi
x=1; while [ $x -le 10 ]; do echo "welcome $x times"; x=$((x+1)); done

Use of GREP: 
Kubectl describe pods | grep --context=10 annotations:
Kubectl describe pods | grep --context=10 Events:
```


```
Get the memory and CPU usage of all the pods and find out top 3 pods which have the highest usage and put them into the cpu-usage.txt file

kubectl top pod --all-namespaces | sort --reverse --key 3 --numeric | head -3

Get first column of string delimited by ":"

cat /etc/passwd | cut -f 1 -d ':' > /etc/foo/passwd  (split line by delimeter ':' and cur the first token and write it to /etc/foo/passwd)


Get current namespace:
kubectl config view --minify | grep namespace

```

## K8S Objects

### PODs

- Get Pod labels with IP/Node info

  ```kubectl get pods -o wide --show-labels```
- Create Pod

  ```
  kubectl run hello —-image=busybox —-restart=Never --dry-run -o yaml > pod.yaml
  
  OR
  
  kubectl run --generator=run-pod/v1 nginx --image=nginx
  ```

- Export existing pod to manifest file

  ```kubectl get pods hello -o yaml --export > pod.yaml```

### Deployments
- Get Deployment with labels

  ```kubectl get deployments -o wide --show-labels```
- Create deployments

  ```
  kubectl run hello —-image=busybox --dry-run -o yaml > deployment.yaml
  
  OR with --replicas
  
  kubectl run hello —-image=busybox --dry-run --replicas=4 -o yaml > deployment
  
  OR with resource quota
  
  kubectl run hello --image=busybox --replicas=15 --limits=cpu=1  --limits=memory=256Mi --requests=cpu=0.5 --requests=memory=256M  --dry-run -o yaml
  ```

- Export existing deployments to manifest file

  ```kubectl get deployments hello --export -o yaml > deployment.yaml```
  
  
- Rollout Strategy:
  ```yaml
  spec:
    strategy:
      rollingUpdate:
        maxUnavailable: 0
        maxSurge: 1
  ```

  ```yaml
  spec:
    strategy:
      type: Recreate

  ```


### Service
- Create a Service named nginx of type NodePort and expose it on port 30080 

  `kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run -o yaml > service.yaml`
  
- Create ClusterIP Service
  `kubectl create service  clusterip  redis-service --tcp=6379 --dry-run -o yaml`
  

### ConfigMaps
- ConfigMaps from literal

  `kubectl create cm hello-cm --from-literal=a=b --from-literal=b=c --dry-run -o yaml > cm.yaml`

- ConfigMaps from file

  `kubectl create cm hello-cm --from-file=config.properties --dry-run -o yaml > cm.yaml`
  
- Generate manifest from existing `cm`

  `kubectl get cm hello-cm -o yaml --export > cm.yaml`
```yaml

ConfigMap possible uses:
---

spec:
  containers:
      env:
        - name: HTTP_PORT
          valueFrom:
            configMapKeyRef:
              name: my-cm
              key: port
---
spec:
  containers:
      envFrom:
        - configMapRef:
              name: my-cm
---
spec:
  volumes:
    - name: vol
      configMap:
        name: my-cm

```

### Secrets

- Create Secret from literal

  `kubectl create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --dry-run -o yaml > secret.yaml`

- Secrets from file

  `kubectl create secret generic db-secret --from-file=config.properties --dry-run -o yaml > cm.yaml`

- Generate manifest from existing `secret`

  `kubectl get secrets db-secret -o yaml --export > secret.yaml`


```yaml
Secret possible uses:
---

spec:
  containers:
      env:
        - name: HTTP_PORT
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: port
---
spec:
  containers:
      envFrom:
        - secretRef:
              name: my-secret
---
spec:
  volumes:
    - name: vol
      secret:
        secretName: my-secret

```

### Namespaces
- Set current context to a namespaces

  ```
  kubectl config set-context $(kubectl config current-context) --namespace=dev
  OR
  kubectl config set-context --current --namespace=dev
  ```
  
- Get resources in all namespaces
  ```
  kubectl get pods --all-namespaces
  ```

### ResourceQuota (for namespaces)
  `kubectl create quota my-quota --hard=cpu=1,memory=1G,pods=2,services=3`
  
  ```yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: dev-ns-resource-quota
    namespace: dev
  spec:
    hard:
      requests.cpu: 250m
      requests.memory: 250mMi
      limits.cpu: 2
      limits.memory: 2G
  ```

### LimitRange (for pods)

`LimitRange` is for managing constraints at a pod and container level within the namespace. An individual Pod or Container that requests resources outside of these LimitRange constraints will be rejected, whereas a `ResourceQuota` only applies to all of the namespace/project's objects in aggregate.

If a pod is created without any limits/requests specified, values defined by `LimitRange` will be used.

Default Range (Memory): `256Mi - 512Mi`

Default Range (CPU): `250m - 1`

Request-Limit range (CPU): `250m - 2`

Request-Limit range (Memory): `256Mi - 3G`


  ```yaml
  apiVersion: v1
  kind: LimitRange
  metadata: 
    name: cpu-mem-limit-range
  spec: 
    limits: 
      - 
        default: 
          cpu: "1"
          memory: "512Mi"
        defaultRequest: 
          cpu: "250m"
          memory: "256Mi"
        max: 
          cpu: "2"
          memory: "3Gi"
        min: 
          cpu: "250m"
          memory: "256Mi"
        type: Container
  ```


### SecurityContext
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: security-context-demo
  spec:
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      fsGroup: 2000
    volumes:
    - name: sec-ctx-vol
      emptyDir: {}
    containers:
    - name: sec-ctx-demo
      image: busybox
      command: [ "sh", "-c", "sleep 1h" ]
      volumeMounts:
      - name: sec-ctx-vol
        mountPath: /data/demo
      securityContext:
        capabilities:
          add: ["NET_ADMIN", "SYS_TIME"]
        allowPrivilegeEscalation: false
  ```


### ServiceAccounts
- Create SA
  ```
  kubectl create sa dashboard-sa
  ```


### Taints & Tolerations
`kubectl taint node node01 key=value:effect`
`effect` can be either `NoSchedule` or `NoExecute`

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  tolerations:
  - key: spray
    operator: Equal (possible values are Equal/Exists)
    value: blue
    effect: NoSchedule
```

### NodeSelector
Lable the nodes first: `kubectl label nodes Node1 size=Large`
```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  nodeSelector:
    size: Large
```

### NodeAffinity (for complex needs e.g. where size is either `Large` or `Medium`)

Valid operators are `In`, `NotIn`, `Exists`, `DoesNotExist`. `Gt`, and `Lt`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In (possible values are In, NotIn, Exists, DoesNotExist, Gt, Lt)
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
```
### Liveness & Readiness


**Readiness** probe allows you to define warm up time for the pod before it can start taking traffic.
**Liveness** Probe is like a health check. Container will be restarted if it fails the liveness check.


```yaml
livenessProbe:
  httpGet:
    path: /live
    port: 80

readinessProbe:
  exec:
    command:
    - ls
    - /app/is_ready
---    
OR
---
readinessProbe:
  tcpScoket:
    port: 3306

```

### Labels & Selectors

**Labels and Selectors** are used to group and select objects, while **Annotations** are used to record other details for informatory purpose (e.g. build number, tools used, email, phone number etc…)

`kubectl get pods --selector app=frontend`
Add labels:
`kubectl run nginx --image=nginx --restart=Never --labels=app=myapp,version=v1`

Update Label:

`kubectl label po nginx version=v2 --overwrite`

Get Pods with version=v2 label:

`kubectl get po -l version=v2`

OR

`kubectl get po --selector=version=v2`

Remove label `app` from multiple pods:

`kubectl label po nginx{1..3} app-`

Annotate pods:

`kubectl annotate po nginx{1..3} description='my description'`

Remove annotation:

`kubectl annotate po nginx{1..3} description-`


### Ingress
You have to have an ingress controller setup in your cluster for ingress to work. Once the ingress controller is installed, you can start creating ingress resources.

https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/#create-an-ingress-resource

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
 name: example-ingress
spec:
 rules:
 - host: hello-world.info
   http:
     paths:
     - path: /
       backend:
         serviceName: web
         servicePort: 8080
```

### NetworkPolicy
Limit access to the resources.

To limit the access to the nginx service so that only Pods with the label access: true can query it, create a NetworkPolicy object as follows:
https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
```

### PV & PVCs

https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage

Kubernetes supports hostPath for development and testing on a single-node cluster. A hostPath PersistentVolume uses a file or directory on the Node to emulate network-attached storage.

In a production cluster, you would not use hostPath. Instead a cluster administrator would provision a network resource like a Google Compute Engine persistent disk, an NFS share, or an Amazon Elastic Block Store volume.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

The next step is to create a PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  resources:
    requests:
      storage: 3Gi
```

The next step is to create a Pod that uses your PersistentVolumeClaim as a volume


```yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  volumes:
    - name: vol
      persistentVolumeClaim:
        claimName: pvc
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: vol
```

Possible options for volumes are as below:

```yaml
  volumes:
    - name: vol
      emptyDir: {}
      
  volumes:
    - name: vol
      configMap:
        name: my-cm

  volumes:
    - name: vol
      configMap:
        name: my-cm
        items:
          - key: application.properties
            path: application.properties

  volumes:
    - name: vol
      secret:
        secretName: my-secret
  
  volumes:
    - name: vol
      persistentVolumeClaim:
        claimName: my-pvc

# This is how all of above can be mounted on a contaner

  volumeMounts:
  - name: vol
    mountPath: /etc/config
```



### Dynamic Provisioning ( & StorageClass)
Dynamic volume provisioning allows storage volumes to be created on-demand. Without dynamic provisioning, cluster administrators have to manually make calls to their cloud or storage provider to create new storage volumes, and then create `PersistentVolume` objects to represent them in Kubernetes. The dynamic provisioning feature eliminates the need for cluster administrators to pre-provision storage. Instead, it automatically provisions storage when it is requested by users.



Storag Class:
- A StorageClass provides a way for administrators to describe the "classes" of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called "profiles" in other storage systems.





### Important concepts

  • Vimrc 3 commands, marking and indenting multiple lines in one go
  • Set image for deployment, pod etc
  • Pause & resume & undo the deployment
  • Run a command after creating the pod -- /bin/sh -c “cmd”
  • Execute a cmd after container creation => k exec -it pod_name container_name -- /bin/sh
  • Check the logs of a container => k logs pod -c container_name
  • pods listing sorted by name, timestamp => --sort-by
  • emptyDir volume (only needed volume and not pvc)
  • hostPath & Volume example
  • PersistentVolume, PersistentVolumeClaim and Pod example (claim storage shouldbe lesser than pv, storage classes & access modes should match)
  • ConfigMap (literals, props file, env file) and Secret usage withing Pod using - load complete cm/secret or a particular key)
  • add label while creating pod (--lables=l1=v1,l2=v2)
  • add label in existing objects - pod, node etc (k label pod nginx l1=v1)
  • modify existing label (k label pod nginx l1=v1 --overwrite)
  • Remove a label (k label pod nginx l1-)
  • Add same label to existing multiple pods => k label pod nginx{1..3} l1=v1
  • Assign a Pod to a particular node => nodeSelector concept, node affinity concept, nodeName
  • Annotations (commands same as labels)
  • Scaling a deployment, min, max, autoscale 
  • Limits & requests while creating pod (--limits=cpu=””,memory=””,--requests=cpu=””,memory=””)
  • Resource quota (--hard=cpu=1,memory=1G,pods=2,services=3,replicationcontrollers=2,resourcequotas=1,secrets=5,persistentvolumeclaims=10)
  • Specify namespace, serviceaccount, securitycontexts (runAs user, group - verify also from within the container whoami, capabilities in a pod and overriding between Pod & container)
  • Job & Cronjob (schedule for every minute) , completions, parallelism, sequential, backOffLimit etc
  • Liveness & readiness Probes - http, tcp, commands examples  initial delay, period seconds


During Exam:
---
- You will be given couple of clusters for the exam duration. Each question will have instructions about what cluster and what namespace to use for that question.
- Pay attention to the cluster they are asking you to connect on top of every qeustions. Make sure you're on the right cluster and namespace. If namespace is not mentioned I assumed its the default namespace.

- Get comfortable with jsonpath. e.g. get events sorted by creatin time. I wasted the most time on a question that would be much quicker if I was better at jsonpath.

e.g.
```
kubectl get events  --sort-by='.metadata.creationTimestamp' 

kubectl get events --sort-by='.lastTimestamp'
```






