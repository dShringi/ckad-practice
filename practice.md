## Core Concepts

1. List all the pods showing name and namespace with a json path expression
```markdown
k get po -o=jsonpath="{range .items[*]}{['metadata.name','metadata.namespace']}{'\n'}{end}"
```
2. Create the nginx pod with version 1.17.4 and expose it on port 80
```markdown
kubectl run nginx --image=nginx:1.17.4 --restart=Never --port=80
```
3. Change the Image version to 1.15-alpine for the pod
```markdown
kubectl set image po/nginx nginx=nginx:1.15-alpine
```
4. Check the Image version without the describe command
```markdown
k get po app -o jsonpath="{range .spec.containers[*]}{.image}{'\n'}{end}"
```
5. Create the nginx pod and execute the simple shell on the pod
```markdown
kubectl exec -it frontend -- /bin/sh [For a running container named frontend]
```
6. Create a busybox pod with command sleep 3600 
```markdown
kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c "sleep 3600"
```
7. Check the connection of the nginx pod from the busybox pod
```markdown
kubectl get po nginx -o wide
kubectl exec -it busybox -- wget -o- <IP Address>
```
8.Create a busybox pod and echo message ‘How are you’ and have it deleted immediately
```markdown
kubectl run busybox --image=nginx --restart=Never -it --rm -- echo "How are you"
```
9. List the nginx pod with custom columns POD_NAME and POD_STATUS
```markdown
kubectl get po -o=custom-columns="POD_NAME:.metadata.name, POD_STATUS:.status.containerStatuses[].state"
```
10. List all the pods sorted by name
```markdown
kubectl get pods --sort-by=.metadata.name
```
---
## Multi-Container Pods

1. Create a Pod with three busy box containers with commands “ls; sleep 3600;”, “echo Hello World; sleep 3600;” and “echo this is the third container; sleep 3600” respectively and check the status
```markdown
kubectl run busybox --image=busybox --restart=Never --dry-run -o yaml -- bin/sh -c "sleep 3600; ls" > multi-container.yaml
// edit the pod to following yaml and create it
kubectl create -f multi-container.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - args:
    - bin/sh
    - -c
    - ls; sleep 3600
    image: busybox
    name: busybox1
    resources: {}
  - args:
    - bin/sh
    - -c
    - echo Hello world; sleep 3600
    image: busybox
    name: busybox2
    resources: {}
  - args:
    - bin/sh
    - -c
    - echo this is third container; sleep 3600
    image: busybox
    name: busybox3
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
2. Check the logs of each container that you just created
```markdown
kubectl logs busybox -c busybox1
kubectl logs busybox -c busybox2
kubectl logs busybox -c busybox3
```
3. Show metrics of the above pod containers and puts them into the file.log
```markdown
kubectl top pod busybox --containers > file.log
```
4. Create a Pod with main container busybox and which executes this “while true; do echo ‘Hi I am from Main container’ >> /var/log/index.html; sleep 5; done” and with sidecar container with nginx image which exposes on port 80. Use emptyDir Volume and mount this volume on path /var/log for busybox and on path /usr/share/nginx/html for nginx container. Verify both containers are running.
```markdown
kubectl run multi-cont-pod --image=busbox --restart=Never --dry-run -o yaml > multi-container.yaml
// edit the yml as below and create it
kubectl create -f multi-container.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-cont-pod
  name: multi-cont-pod
spec:
  volumes:
  - name: var-logs
    emptyDir: {}
  containers:
  - image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'Hi I am from Main container' >> /var/log/index.html; sleep 5;done"]
    name: main-container
    resources: {}
    volumeMounts:
    - name: var-logs
      mountPath: /var/log
  - image: nginx
    name: sidecar-container
    resources: {}
    ports:
      - containerPort: 80
    volumeMounts:
    - name: var-logs
      mountPath: /usr/share/nginx/html
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
---

## Pod Design

1. Get the pods with label information
```markdown
kubectl get pods --show-labels
```
2.  Create a nginx pods with labels env=prod, app=hazelcast and environment variable host=google.com, port=80
```markdown
kubectl run nginx-prod --image=nginx --restart=Never --labels="env=prod,app=hazelcast" --env="host=google.com" --env="port=80"
```
3. Get the pods with labels env=dev and env=prod
```markdown
kubectl get pods -l 'env in (dev,prod)'
```
4. Change the label for one of the pod to env=uat
```markdown
kubectl label pod/nginx-prod env=uat --overwrite
```
5. Remove the labels for the pods that we created now
```markdown
kubectl label pod nginx-prod{1..2} env-
```
6. Let’s add the label app=nginx for all the pods
```markdown
kubectl label pod nginx-prod{1..2} app=nginx
```
7. Create a Pod that will be deployed on this node with the label nodeName=nginxnode
```markdown
kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > pod.yaml
// add the nodeSelector like below and create the pod
kubectl create -f pod.yaml
```
```yaml
spec:
  nodeSelector:
    nodeName: nginxnode
```
8. Annotate the pods with name=webapp
```markdown
kubectl annotate pod nginx-prod{1..2} name=webapp
```
9. Remove the annotations on the pods and verify
```markdown
kubectl annotate pod nginx-prod{1..2} name-
```
10. Scale the deployment from 5 replicas to 20 replicas
```markdown
kubectl scale deploy webapp --replicas=20
```
11. Get the deployment rollout status
```markdown
kubectl rollout status deploy webapp
```
12. Update the deployment with the image version 1.17.4
```markdown
kubectl set image deploy/webapp nginx=nginx:1.17.4
kubectl rollout status deploy webapp
```
13. Check the rollout history and undo the deployment to previous version
```markdown
kubectl rollout history deploy webapp
kubectl rollout undo deploy webapp
```
14. Update the deployment to the Image 1.17.1
```markdown
kubectl rollout undo deploy webapp --to-revision=3
```
15. Check the history of the specific revision of that deployment
```markdown
kubectl rollout history deploy webapp --revision=7
```
16. Apply the autoscaling to this deployment with minimum 10 and maximum 20 replicas and target CPU of 85% and verify hpa is created and replicas are increased to 10 from 1
```markdown
kubectl autoscale deploy webapp --min=10 --max=20 --cpu-percent=85
kubectl get hpa
kubectl get pod -l app=webapp
```
17. Create the Job with the image busybox which echos “Hello I am from job”, make it run 10 times with 5 jobs in parallel
```markdown
kubectl create job hello-job --image=busybox --dry-run -o yaml -- echo "Hello I am from job" > hello-job.yaml
// edit the yaml file to add completions: 10 and parallelism: 5
kubectl create -f hello-job.yaml
```
```yaml
spec:
  completions: 10
  parallelism: 5
```
19. Create a Cronjob with busybox image that prints date and hello from kubernetes cluster message for every minute
```markdown
kubectl create cj date-job --image=busybox --schedule="*/1 * * * *" -- bin/sh -c "date; echo Hello from kubernetes cluster"
```
---
## State Persistence
1.Create a hostPath PersistentVolume named task-pv-volume with storage 10Gi, access modes ReadWriteOnce, storageClassName manual, and volume at /mnt/data 
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
2.  Create a PersistentVolumeClaim of at least 3Gi storage and access mode ReadWriteOnce
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
3. Create a Pod with an image Redis and configure a volume that lasts for the lifetime of the Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```
4. Create an nginx pod with containerPort 80 and with a PersistentVolumeClaim task-pv-claim and has a mouth path "/usr/share/nginx/html"
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
---

## Confiuration

1. Create an nginx pod and load environment values from the configmap envcfgmap and exec into the pod and verify the environment variables and delete the pod
```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
    env:
    - name: ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: envcfgmap
          key: var1
```
2. Create a configmap called cfgvolume with values var1=val1, var2=val2 and create an nginx pod with volume nginx-volume which reads data from this configmap cfgvolume and put it on the path /etc/cfg
```yaml
spec:
  volumes:
  - name: nginx-volume
    configMap:
      name: cfgvolume
  containers:
  - image: nginx
    name: nginx
    resources: {}
    volumeMounts:
    - name: nginx-volume
      mountPath: /etc/cfg
```
3. Create a pod called secbusybox with the image busybox which executes command sleep 3600 and makes sure any Containers in the Pod, all processes run with user ID 1000 and with group id 2000
```yaml
spec:
  securityContext: # add security context
    runAsUser: 1000
    runAsGroup: 2000
  containers:
  - args:
    - /bin/sh
    - -c
    - sleep 3600;
    image: busybox
    name: secbusybox
```
4. Create the same pod as above this time set the securityContext for the container as well and verify that the securityContext of container overrides the Pod level securityContext
```yaml
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - args:
    - /bin/sh
    - -c
    - sleep 3600;
    image: busybox
    securityContext:
      runAsUser: 2000
    name: secbusybox
```
5. Create pod with an nginx image and configure the pod with capabilities NET_ADMIN and SYS_TIME
```yaml
spec:
  containers:
  - image: nginx
    securityContext:
      capabilities:
        add: ["SYS_TIME", "NET_ADMIN"]
```
6. Create a Pod nginx and specify a memory, cpu request and a memory, cpu limit of 100Mi, 1 and 200Mi, 2 respectively
```markdown
k run po nginx --image=nginx --requests=cpu=1,memory=256Mi --limits=cpu=2,memory=512Mi
```
7. Create a secret mysecret with values user=myuser and password=mypassword
```markdown
kubectl create secret generic my-secret --from-literal=username=user --from-literal=password=mypassword
```
8. Create an nginx pod which reads username as the environment variable
```yaml
spec:
  containers:
  - image: nginx
    env:
    - name: USER_NAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username
```
9. Create an nginx pod which loads the secret as environment variables
```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    envFrom:
    - secretRef:
        name: my-secret
```
10. Create a busybox pod which executes this command sleep 3600 with the service account admin
```yaml
spec:
  serviceAccountName: admin
```
---

## Observability

1. Create an nginx pod with containerPort 80 and it should only receive traffic only if checks the endpoint / on port 80
```yaml
spec:
  containers:
  - image: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
```
2. Create an nginx pod with containerPort 80 and it should check the pod running at endpoint / healthz on port 80
```yaml
spec:
  containers:
  - image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
```
3. Check what all are the options that we can configure with readiness and liveness probes
```markdown
kubectl explain Pod.spec.containers.livenessProbe
kubectl explain Pod.spec.containers.readinessProbe
```
4. Create the pod nginx with the above liveness and readiness probes so that it should wait for 20 seconds before it checks liveness and readiness probes and it should check every 25 seconds
```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      initialDelaySeconds: 20
      periodSeconds: 25
      httpGet:
        path: /healthz
        port: 80
    readinessProbe:
      initialDelaySeconds: 20
      periodSeconds: 25
      httpGet:
        path: /
        port: 80
```
---

## Services and Networking

1. Create an nginx pod with a yaml file with label my-nginx and expose the port 80
```markdown
kubectl run nginx --image=nginx --restart=Never --port=80
```
2. Create the service for this nginx pod with the pod selector app: my-nginx
```markdown
spec:
  selector:
    app: my-nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
3. Delete the service and create the service with kubectl expose command
```markdown
kubectl expose po nginx --port=80 --target-port=9376
```
4. Create the service again with type NodePort
```markdown
kubectl expose po nginx --port=80 --type=NodePort
```
5. Create the temporary busybox pod and hit the service. Verify the service that it should return the nginx page index.html
```markdown
kubectl get svc nginx -o wide
// create temporary busybox to check the nodeport
kubectl run busybox --image=busybox --restart=Never -it --rm -- wget -o- <Cluster IP>:80
```
6. Create a NetworkPolicy which denies all ingress traffic
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```
