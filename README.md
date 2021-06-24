# CKAD Notes and Imperative Commands

> **Prepared for K8 Version 1.20 (Q2).**

> Curriculum of CKAD is changing from Q3. [Details here](https://github.com/cncf/curriculum/blob/master/CKAD-2021_Curriculum_Coming_Q3_2021.pdf)

- [CKAD Notes and Imperative Commands](#ckad-notes-and-imperative-commands)
  - [Generic Linux commands and settings](#generic-linux-commands-and-settings)
  - [Core concepts](#core-concepts)
    - [Pods](#pods)
    - [Resources](#resources)
  - [Multi container pods:](#multi-container-pods)
  - [Pod Design](#pod-design)
    - [Labels and Annotations](#labels-and-annotations)
    - [Deployment](#deployment)
    - [Jobs & Cron Jobs](#jobs--cron-jobs)
  - [Configuration](#configuration)
    - [Commands & Arguments](#commands--arguments)
    - [Resource requirements](#resource-requirements)
    - [Config map and secrets](#config-map-and-secrets)
    - [Config maps and secret using volume mounts](#config-maps-and-secret-using-volume-mounts)
    - [Security contexts & Service Accounts](#security-contexts--service-accounts)
    - [Taints and Toleration:](#taints-and-toleration)
    - [Node selectors](#node-selectors)
    - [Node Affinity](#node-affinity)
  - [Service and networking](#service-and-networking)
    - [Ingress](#ingress)
    - [Network policies](#network-policies)
  - [Observability](#observability)
    - [Logging](#logging)
    - [Metrics](#metrics)
    - [Readiness and Liveness probe](#readiness-and-liveness-probe)
  - [Volumes](#volumes)
    - [Persistent volume](#persistent-volume)
    - [Persistent Volume claim](#persistent-volume-claim)
    - [Configure pod](#configure-pod)
  - [Reference:](#reference)
    - [From official Kubernetes site](#from-official-kubernetes-site)
    - [Course and exercises to follow](#course-and-exercises-to-follow)
    - [Tips for CKAD exams](#tips-for-ckad-exams)

## Generic Linux commands and settings

- `wc -l <filename>` prints the line count
- Alias and autocomplete

  ```sh
  alias k=kubectl
  alias kx=kubectl explain
  alias kd=kubectl describe
  source <(kubectl completion bash)
  complete -F __start_kubectl k
  alias kcs='k config set-context --namespace'
  # Even better alias:
  alias kns='kubectl config set-context --current --namespace' # example: kns <mynamspace>
  ```

- `k config set-context --namespace=<namespace> --current` : use namespace for current context
- Another way of configuring alias
  ```sh
  export NS=default
  alias k='kubectl -n $NS'
  alias ka='kubectl -n $NS apply -f'
  alias kgp='kubectl get pod -n $NS'
  ```

- Vim editor settings
  
  ```sh 
  vi ~/.vimrc

  set nu   #for numbering
  set expandtab #Expand TABs to spaces
  set shiftwidth=2
  set tabstop=2 #tab width
  syntax on
  colorscheme desert
  ```

- Create manifest using `cat` command
  
  ```sh 
  cat <<eof > file.yaml
  <your content>
  eof
  ```

- If using tmux

  ```sh
  tmux

  CTRL+B % (Split tmux screen vertically)
  CTRL+B " (Split tmux screen horizontally)
  CTRL+B <- -> (Switch between panes)
  CTRL+B x (Exit)
  ```

- netcat to check connectivity from one pod to another: `nc -z -v -w 1 <name of service> <port>`


## Core concepts

### Pods

- `kubectl config set-context --current --namespace=cloud-app-gateway`
- `k config set-context --namespace=ingress-space --current`
- `kcs ingress-namespace --current` : using alias (Saves lot of time)
- `kubectl run nginx --image=nginx`
- `kubectl run nginx --image=nginx  --dry-run=client -o yaml`
- `k explain pod --recursive | grep -A8 -B4 envFrom`
- `kubectl get pods -o wide --show-labels | grep -i finance | wc -l`  -- word count
- `kubectl get pods --selector bu=finance` - filter based on label
- `kubectl get all --selector env=prod` - get all based on labels
- `kubectl get all --selector env=prod,bu=finance,tier=frontend` multiple selectors
- `kubectl get po -l env=dev --no-headers | wc -l` - No headers - for not displaying the first line of the result. 
- `k run nginx --image=nginx --port=80  --expose` : Creates a service

### Resources

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: # At container level
      limits:
        cpu: 200m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 256Mi
```

- `kubectl run nginx --image=nginx --restart=Never --port=80 --namespace=myname --command --serviceaccount=mysa1 --env=HOSTNAME=local --labels=bu=finance,env=dev  --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --dry-run -o yaml -- /bin/sh -c 'echo hello world'`

```yaml
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - echo hello world
```

- `kubectl run nginx --image=nginx --restart=Never --port=80 --namespace=myname --serviceaccount=mysa1 --env=HOSTNAME=local --labels=bu=finance,env=dev  --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --dry-run -o yaml -- /bin/sh -c 'echo hello world'`

```yaml
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - echo hello world
```


## Multi container pods: 

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: box
  name: box
spec:
  initContainers: #
  - args: #
    - /bin/sh #
    - -c #
    - wget -O /work-dir/index.html http://neverssl.com/online #
    image: busybox #
    name: box #
    volumeMounts: #
    - name: vol #
      mountPath: /work-dir #
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    volumeMounts: #
    - name: vol #
      mountPath: /usr/share/nginx/html #
  volumes: #
  - name: vol #
    emptyDir: {} #
```

- [Read about Logging architecture in K8](https://kubernetes.io/docs/concepts/cluster-administration/logging/)

## Pod Design

### Labels and Annotations

- `k annotate pod nginx1 nginx2 nginx3 description='my description'`
- `k annotate pod nginx1 nginx2 nginx3 description-`
- `k label pod nginx1 app=v2 --overwrite`
- `k label pod nginx3 app-`
- `kubectl label po nginx1 nginx2 nginx3 app-` : Remove labels at once

### Deployment

- `kubectl create deployment --image=nginx nginx`
- `kubectl create deployment --image=nginx nginx --dry-run -o yaml`
- `kubectl create deployment nginx --image=nginx --replicas=4`
- `kubectl scale deployment nginx --replicas=4`
- `kubectl create deployment nginx --image=nginx--dry-run=client -o yaml > nginx-deployment.yaml`
- `kubectl set image deployment/nginx busybox=busybox nginx=nginx:1.9.1`
- When asked for specific image or when looking of something specific try to use grep command
- `kubhectl expose deployement my-dep --name=my-service --target-port=8080 --type=NodePort --port=80`
- `kubectl rollout undo deployment/myapp-deployment`
- `kubectl autoscale deployment nginx --min=5 --max=10 --cpu-percent=80` : Autoscale deployment
- `kubectl rollout status deployment nginx`
- `kubectl rollout history deployment nginx`
- `kubectl rollout history deployment nginx --revision=1`
- `kubectl set image deployment nginx nginx=nginx:1.17 --record`
- `kubectl rollout undo deployment nginx  --to-revision=2` : To a specific revision
- `k get rs nginx-67dfd6c8f9 -o yaml --show-managed-fields=false` : To avoid extra fields in yaml

- `.spec.strategy.type==RollingUpdate`
- `.spec.strategy.rollingUpdate.maxUnavailable`
- `.spec.strategy.rollingUpdate.maxSurge`

### Jobs & Cron Jobs


- `kubectl create cronjob throw-dice-cron-job --image=kodekloud/throw-dice --schedule="30 21 * * *"`
- `k create job busybox --image=busybox -- /bin/sh -c 'echo hello;sleep 30;echo world'`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  ## Cron Spec
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      ## Job Spec
      template:
        ## Pod spec
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure

## Few important properties:

- completions: Number of jobs to run
- parallelism: Parallel executions of jobs
- backOffLimit: to specify the number of retries before considering a Job as failed.
- activeDeadlineSeconds: Number of seconds after the job is scheduled 
- ttlSecondsAfterFinished: Clean up the jobs after it finishes after n number of seconds
```


## Configuration

### Commands & Arguments


- `kubectl run nginx --image=nginx --restart=Never --port=80 --namespace=myname --command --serviceaccount=mysa1 --env=HOSTNAME=local --labels=bu=finance,env=dev  --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --dry-run -o yaml -- /bin/sh -c 'echo hello world'`

```yaml
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - echo hello world
```

- `kubectl run nginx --image=nginx --restart=Never --port=80 --namespace=myname --serviceaccount=mysa1 --env=HOSTNAME=local --labels=bu=finance,env=dev  --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --dry-run -o yaml -- /bin/sh -c 'echo hello world'`

```yaml
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - echo hello world
```


### Resource requirements

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: # At container level
      limits:
        cpu: 200m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 256Mi
```

- Default limits by LimitRange
  
  ```yaml
  apiVersion: v1
  kind: LimitRange
  metadata:
    name: mem-limit-range
  spec:
    limits:
    - default:
        memory: 512Mi
      defaultRequest:
        memory: 256Mi
      type: Container
  ```

### Config map and secrets

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - key: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: 
          key: 

        
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - key: APP_COLOR
      valueFrom:
        secretKeyRef:
          name: 
          key: 

spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name:

spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - secretRef:
        name:
```

### Config maps and secret using volume mounts


```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config

spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:     # container level
      - name: secret-volume
        mountPath: /etc/secrets
  volumes:        # Pod level
    - name: secret-volume
      secret:
        secretName: secret-values
```

### Security contexts & Service Accounts

```yaml
spec:
  securityContext:    # Pod level 
    runAsUser: 1000
  containers:
  - image: nginx
    name: nginx
    securityContext:  # Container level
      runAsUser:
      capabilities:
        add: ["MAC_ADMIN"]
```

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
  serviceAccount: dashboard-sa  # At pod level

```

- `k create sa myuser`
- `k run nginx1 --image=nginx --port=80  --serviceaccount=myuser`
- `kubectl set serviceaccount deployment frontend myuser`

### Taints and Toleration:

- Taints on node
- Toleration on pod
- Does not prevent tolerated pods to be scheduled on other nodes
- Taint effects:
  - NoSchedule : No new pods will be scheduled
  - PreferNoSchedule: Try to avoid new pods schedule
  - NoExecute : No New pods will be scheduled and existing pods will be deleted from the node. 

- `k taint nodes node-name key=value:taint-effect`

```yaml

spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:    # Pod level
  - key: "app"
    operator: "Equals" 
    value: "blue"
    effect: "NoSchedule"
```

### Node selectors

- Schedule pod on specific node.

```yaml
spec:
  nodeSelector:
    size: large     ## These are labels assigned to nodes
  containers:
  - image: nginx
    name: nginx
```

- Limitations: 
  - Does not support OR conditions like Large or Medium but not small
  - For this node affinity and anti-affinity features were introduced.

### Node Affinity

- Support advanced expressions

```yaml
spec: 
  containers:
  - image: nginx
    name: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```



## Service and networking

- `k run nginx --image=nginx --port=80  --expose` : creates pod and exposes a service
- `kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml` : Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379 (This will automatically use the pod's labels as selectors)

  OR

- `kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml`  : (This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service)


- `kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml` : Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes. 
(This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)

  OR

- `kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml` (This will not use the pods labels as selectors)


- Both the above commands have their own challenges. While one of it cannot accept a selector the other cannot accept a node port. I would recommend going with the `kubectl expose` command. If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.


- `kubectl expose deployment simple-web-app --name=my-service --targetPort=8080 --type=NodePort --port=8080`
- `kubectl run frontend --replicas=2 --labels=run=load-balancer-example --image=busybox  --port=8080`
- `kubectl expose deployment frontend --type=NodePort --name=frontend-service --port=6262 --target-port=8080`
- `kubectl create service clusterip my-cs --tcp=5678:8080 --dry-run -o yaml`

- To access a service from temporary pod using service name (if service is running in different namespace than temporary pod).
- `k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 <svc-name>.<namespace>:<port>`
- `k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 <svc-name>.<namespace>.svc.cluster.local:<port>`

### Ingress

- `kubectl create ingress ingress1 --class=default --rule="foo.com/path*=svc:8080" --rule="bar.com/admin*=svc2:http" -o yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80

# Single host multiple paths

- host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```


### Network policies

```yaml
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-prod
      namespaceSelector:    # If you want to restrict for pods in a specific namespace.
        matchLabels:
          name: prod
    ports:
    - protocol: TCP
      port: 3306  
```

## Observability

### Logging

- `kubectl logs question-13-pod -c question-thirteen --v 4`  -- logs with verbose levels
- `kubectl get events | grep -i error`
- `kubectl logs nginx --previous`

### Metrics

- `k top pods -A | sort -k 4n`
- `k top pod --no-headers -A --sort-by=cpu | awk '{print $2}' | head -1` : get name of the pod with max cpu utilization.


### Readiness and Liveness probe

```yaml

## Running a command
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 8

## HTTP Get 
readinessProbe:
  httpGet:
    path: /
    port: 80

### Sample
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```


## Volumes

- To find the storage class that does not support dynamic provisioning Look for the storage class name that uses `no-provisioner`
- `kubectl cp busybox:etc/passwd ./passwd` : copy from container to local

```sh
$ cat etc/passwd

root:x:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/bin/false
bin:x:2:2:bin:/bin:/bin/false
sys:x:3:3:sys:/dev:/bin/false
sync:x:4:100:sync:/bin:/bin/sync
mail:x:8:8:mail:/var/spool/mail:/bin/false
www-data:x:33:33:www-data:/var/www:/bin/false
operator:x:37:37:Operator:/var:/bin/false
nobody:x:65534:65534:nobody:/home:/bin/false


# Cut first column
cat /etc/passwd | cut -f 1 -d ':' > /etc/foo/passwd 
```

### Persistent volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
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

### Persistent Volume claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

### Configure pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

## Reference:

### From official Kubernetes site
- [Cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [K8 Commands](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- [K8 Tasks](https://kubernetes.io/docs/tasks/)

### Course and exercises to follow
- If you are new to `vim` then use `vimtutor` to get hands dirty on `vim` editor.
- [Udemy Course - Mumshad Mannambeth](https://www.udemy.com/course/certified-kubernetes-application-developer/)
- [Udemy Course - Zeal Vora](https://www.udemy.com/course/mastering-certified-kubernetes-application-developer/)
- [CKAD-Exercises](https://github.com/dgkanatsios/CKAD-exercises). 
- [CKAD Simulator](https://killer.sh). You get access to one free simulator for 36 hours using Linux foundation single sign on. 
- [5 Sample Questions - CKAD](https://matthewpalmer.net/kubernetes-app-developer/articles/ckad-practice-exam.html)
- [Answers to 5 sample questions](https://thospfuller.com/2020/11/09/answers_to_five_kubernetes_ckad_practice_questions_2021/)

### Tips for CKAD exams
- [Tips on preparing for Certified Kubernetes Application Developer (CKAD)](https://www.youtube.com/watch?v=rnemKrveZks)
- [Hands-on Tips to Pass the CKAD Exam (Cloud Academy)](https://www.youtube.com/watch?v=L6K_8dOFR5w)


