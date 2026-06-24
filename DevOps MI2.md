# LO4 — Accelerated delivery of multilayer applications using containers

**1. Create a Deployment named web running nginx:1.25 with 3 replicas using a single imperative kubectl create deployment command, then export its generated YAML to a file.**

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web-deployment.yaml
```

To create it live and then export:
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web -o yaml > web-deployment.yaml
```

---

**2. Enable pulling images from DockerHub as an authenticated user to lower the chance of hitting the pull limit. What kind of resource do you need to create?**

You need to create a **Secret of type `kubernetes.io/dockerconfigjson`** (an image-pull secret). This stores your DockerHub credentials so the kubelet authenticates when pulling images. You create it with `kubectl create secret docker-registry <name> --docker-server=docker.io --docker-username=<user> --docker-password=<pass>` and reference it in `spec.imagePullSecrets` of your pod/deployment. Authenticated pulls have a much higher rate limit (200 pulls/6h vs 100 for anonymous).

---

**3. Scale web from 3 to 5 replicas two different ways — once with kubectl scale, once by editing the manifest — and show the ReplicaSet that results.**

```bash
# Method 1: imperative scale
kubectl scale deployment web --replicas=5

# Method 2: edit manifest and apply
# Change spec.replicas to 5 in web-deployment.yaml, then:
kubectl apply -f web-deployment.yaml

# Show resulting ReplicaSet
kubectl get rs -l app=web
```

Both methods update the Deployment's `.spec.replicas` field; the Deployment controller then adjusts the underlying ReplicaSet to run 5 pods.

---

**4. Perform a rolling update of web from nginx:1.25 to nginx:1.27 and watch it with kubectl rollout status.**

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

The rolling update creates a new ReplicaSet with nginx:1.27 pods, gradually scaling it up while scaling down the old ReplicaSet. `rollout status` streams progress until all replicas are updated and available.

---

**5. View a Deployment's rollout history and roll back to the previous revision. Explain what the CHANGE-CAUSE column shows and how to populate it.**

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

The CHANGE-CAUSE column shows the value of the `kubernetes.io/change-cause` annotation at the time of each revision. It is empty by default; you populate it by annotating the deployment after a change: `kubectl annotate deployment/web kubernetes.io/change-cause="Updated to nginx:1.27"`. This gives you a human-readable audit trail of why each rollout occurred.

---

**6. Set the update strategy to RollingUpdate with maxSurge: 1 and maxUnavailable: 0, and explain the effect on availability during an update.**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

With `maxUnavailable: 0`, Kubernetes never terminates an old pod until a new one is Ready, guaranteeing full capacity at all times. `maxSurge: 1` means at most one extra pod above the desired count is created during the rollout. This ensures zero downtime but makes rollouts slower since each new pod must pass readiness before an old one is removed.

---

**7. Switch a Deployment's strategy to Recreate and describe a concrete scenario where this is required instead of RollingUpdate.**

```yaml
spec:
  strategy:
    type: Recreate
```

Recreate terminates all existing pods before creating new ones, causing a brief downtime window. This is required when your application cannot tolerate two different versions running simultaneously — for example, a database migration pod that holds an exclusive lock on a schema, or an app that uses a shared persistent volume with `ReadWriteOnce` access mode where only one pod can mount it at a time.

---

**8. Add CPU/memory requests and limits to a Deployment's container and verify they appear in the running pod spec.**

```yaml
containers:
- name: nginx
  image: nginx:1.27
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "250m"
      memory: "256Mi"
```

```bash
kubectl apply -f web-deployment.yaml
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].resources}'
```

The output confirms the scheduler used requests for placement and the kubelet enforces the limits via cgroups.

---

**9. Set revisionHistoryLimit: 3 and explain how it affects your ability to roll back and how many old ReplicaSets are kept.**

```yaml
spec:
  revisionHistoryLimit: 3
```

Kubernetes retains only the 3 most recent old ReplicaSets (scaled to 0 replicas), meaning you can roll back to at most 3 previous revisions. Older ReplicaSets are garbage-collected and their revision history is lost. This saves etcd storage but limits how far back you can undo.

---

**10. Set a Deployment's image to a non-existent tag, observe the rollout get stuck, and explain why the old pods keep serving traffic.**

```bash
kubectl set image deployment/web nginx=nginx:nonexistent
kubectl rollout status deployment/web  # hangs
kubectl get pods  # new pods in ImagePullBackOff, old pods still Running
```

The default RollingUpdate strategy only terminates old pods once new pods become Ready. Since the new pods can never pull the non-existent image, they never become Ready, so the old pods are preserved and continue serving traffic. The rollout is stuck but availability is maintained by design.

---

**11. Use a label selector to list only the pods belonging to one Deployment, and explain the Deployment → ReplicaSet → Pod selector relationship.**

```bash
kubectl get pods -l app=web
```

A Deployment's `spec.selector.matchLabels` selects which ReplicaSets it manages; the ReplicaSet uses the same labels in its own selector to own pods. The chain is: Deployment matches ReplicaSets via labels → ReplicaSet matches Pods via the same labels → each layer uses `ownerReferences` for garbage collection.

---

**12. Expose a Deployment with kubectl expose, then explain what object was created and how its selector was derived.**

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl get svc web -o yaml
```

This creates a **Service** object of type ClusterIP. Its `spec.selector` is automatically copied from the Deployment's `spec.selector.matchLabels` (e.g., `app: web`), so the Service routes traffic to all pods managed by that Deployment. The Service gets a stable virtual IP within the cluster.

---

**13. Add a sidecar (second) container to a Deployment's pod template and explain how the two containers share the pod network and volumes.**

```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
      - name: sidecar
        image: busybox
        command: ["sh", "-c", "while true; do date > /html/index.html; sleep 5; done"]
        volumeMounts:
        - name: shared
          mountPath: /html
      volumes:
      - name: shared
        emptyDir: {}
```

Both containers share the same network namespace (same IP, localhost communication, same ports) and can mount the same volumes. They are scheduled together on the same node and share the pod's lifecycle.

---

**14. Write a Deployment manifest for httpd:2.4 with 2 replicas, a named container port, and a nodeSelector; explain why it stays Pending if no node matches.**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - name: http
          containerPort: 80
```

If no node carries the label `disktype=ssd`, the scheduler cannot find a suitable node and the pods remain Pending. The events will show `FailedScheduling` with the message "0/X nodes are available: X node(s) didn't match Pod's node affinity/selector."

---

**15. Create a bare Pod (no controller) running busybox that sleeps, then explain what happens when you delete it versus a Deployment-managed pod.**

```bash
kubectl run bare --image=busybox --restart=Never -- sleep 3600
kubectl delete pod bare  # gone forever
```

A bare pod has no controller, so once deleted it is permanently gone — nothing recreates it. A Deployment-managed pod is owned by a ReplicaSet; when deleted, the ReplicaSet controller immediately creates a replacement to maintain the desired replica count. This is why controllers provide self-healing.

---

**16. Write a StatefulSet for redis:7 with 3 replicas plus a headless Service, and show the stable pod names and per-pod DNS records.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
  - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
```

Pods get stable names: `redis-0`, `redis-1`, `redis-2`. Each has a DNS record: `redis-0.redis.<namespace>.svc.cluster.local`, etc. These names persist across restarts, enabling stable network identity.

---

**17. Create a DaemonSet running a busybox agent and explain why exactly one pod runs per node.**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
      - name: agent
        image: busybox
        command: ["sh", "-c", "while true; do echo running; sleep 60; done"]
```

A DaemonSet controller ensures exactly one pod instance per eligible node by watching the node list and scheduling a pod on each. When nodes join they get a pod; when they leave the pod is removed. This is ideal for node-level concerns like log collection, monitoring agents, or network plugins.

---

**18. Create a Job using busybox/perl that computes something once and completes; show COMPLETIONS and read the result from logs.**

```bash
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(100)"
kubectl get job pi  # COMPLETIONS: 1/1
kubectl logs job/pi
```

The Job controller creates a pod that runs to completion (exit code 0), then marks the Job as Complete. COMPLETIONS shows `1/1` meaning 1 successful completion out of 1 required. The result (pi to 100 digits) is read from the pod's stdout via `kubectl logs`.

---

**19. Create a CronJob that prints the date every minute; show how to suspend it and how to list the Jobs it spawns.**

```bash
kubectl create cronjob dateprinter --image=busybox --schedule="*/1 * * * *" -- date

# Suspend it
kubectl patch cronjob dateprinter -p '{"spec":{"suspend":true}}'

# List spawned Jobs
kubectl get jobs -l job-name  # or:
kubectl get jobs --selector=batch.kubernetes.io/controller-uid
```

A CronJob creates a new Job object on each schedule tick. Suspending it prevents new Jobs from being created but doesn't affect already-running ones.

---

**20. Manually run a Job from an existing CronJob (kubectl create job --from=cronjob/...) to test it on demand.**

```bash
kubectl create job manual-date --from=cronjob/dateprinter
kubectl get job manual-date
kubectl logs job/manual-date
```

This creates an immediate one-off Job using the CronJob's pod template without waiting for the next scheduled time. It's useful for testing the CronJob's logic on demand or re-running a failed execution.

---

**21. Show that a StatefulSet creates and terminates pods in order, and contrast this with a Deployment.**

```bash
kubectl scale statefulset redis --replicas=5
# Pods are created sequentially: redis-3 must be Running before redis-4 starts
kubectl scale statefulset redis --replicas=2
# Pods terminate in reverse: redis-4, then redis-3, then redis-2
```

StatefulSets guarantee ordered, sequential creation (0→1→2) and reverse-order termination. Deployments create/terminate pods in parallel via ReplicaSets with no guaranteed order. This ordered approach is essential for stateful workloads like databases that require leader election or data replication sequencing.

---

**22. Create a multi-container Pod where two containers share an emptyDir — one writes a file, the other reads it. Prove it's shared.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo hello > /data/msg.txt && sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "sleep 5 && cat /data/msg.txt && sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /data
  volumes:
  - name: shared
    emptyDir: {}
```

```bash
kubectl logs shared-vol -c reader  # outputs "hello"
```

Both containers mount the same `emptyDir` volume, proving the filesystem is shared within the pod.

---

**23. List the StorageClasses, PVs and PVCs.**

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc --all-namespaces
```

StorageClasses define provisioner and parameters for dynamic volume creation. PVs are cluster-level storage resources (provisioned statically or dynamically). PVCs are namespace-scoped requests that bind to PVs matching their size and access mode requirements.

---

**24. Mount a ConfigMap as a volume so each key becomes a file, and verify the contents inside the pod.**

```bash
kubectl create configmap myconfig --from-literal=key1=value1 --from-literal=key2=value2
```

```yaml
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /config/key1 && cat /config/key2 && sleep 3600"]
    volumeMounts:
    - name: config-vol
      mountPath: /config
  volumes:
  - name: config-vol
    configMap:
      name: myconfig
```

```bash
kubectl exec <pod> -- ls /config      # shows: key1  key2
kubectl exec <pod> -- cat /config/key1 # shows: value1
```

Each key in the ConfigMap becomes a separate file in the mounted directory.

---

**25. Mount a single ConfigMap key to a specific path using subPath; explain when it's needed and its update caveat.**

```yaml
volumeMounts:
- name: config-vol
  mountPath: /etc/nginx/nginx.conf
  subPath: nginx.conf
volumes:
- name: config-vol
  configMap:
    name: myconfig
```

`subPath` is needed when you want to mount a single file into an existing directory without overwriting the directory's other contents. The caveat is that subPath-mounted files **do not receive live updates** when the ConfigMap changes — unlike full volume mounts, they remain static until the pod is restarted.

---

**26. Mount a Secret as a volume and verify the files contain decoded values with restrictive permissions.**

```bash
kubectl create secret generic db-creds --from-literal=user=admin --from-literal=pass=s3cret
```

```yaml
volumeMounts:
- name: secret-vol
  mountPath: /secrets
  readOnly: true
volumes:
- name: secret-vol
  secret:
    secretName: db-creds
```

```bash
kubectl exec <pod> -- cat /secrets/user  # admin (decoded, not base64)
kubectl exec <pod> -- ls -la /secrets/   # files have 0644 or custom mode
```

Secret volume files are automatically base64-decoded and mounted with restrictive permissions (defaultMode 0644, configurable to 0400).

---

**27. Show that scaling a StatefulSet creates one PVC per replica, then explain what happens to those PVCs when the StatefulSet is deleted.**

```bash
kubectl get pvc -l app=redis
# Shows: data-redis-0, data-redis-1, data-redis-2 (one per replica)
kubectl delete statefulset redis
kubectl get pvc -l app=redis  # PVCs still exist!
```

StatefulSets use `volumeClaimTemplates` to create a unique PVC per pod. When the StatefulSet is deleted, PVCs are **intentionally retained** to prevent accidental data loss. You must manually delete them if you want to reclaim storage. This is a safety feature for stateful workloads.

---

**28. Use an initContainer to pre-populate data into a shared volume before the main container reads it.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init
    image: busybox
    command: ["sh", "-c", "echo 'initialized' > /work/data.txt"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  containers:
  - name: main
    image: busybox
    command: ["sh", "-c", "cat /work/data.txt && sleep 3600"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  volumes:
  - name: workdir
    emptyDir: {}
```

The initContainer runs to completion before the main container starts, guaranteeing the data is available. This pattern is used for downloading configs, running migrations, or waiting for dependencies.

---

**29. Set readOnly: true on a volume mount and prove writes are rejected.**

```yaml
volumeMounts:
- name: config-vol
  mountPath: /data
  readOnly: true
```

```bash
kubectl exec <pod> -- touch /data/test
# Error: touch: /data/test: Read-only file system
```

The `readOnly: true` flag makes the kernel mount the filesystem as read-only, so any write syscall returns EROFS. This is a defense-in-depth measure to protect volume contents from accidental modification.

---

**30. Set a sizeLimit on an emptyDir and explain what happens if the container exceeds it.**

```yaml
volumes:
- name: tmp
  emptyDir:
    sizeLimit: 100Mi
```

If the total data written to the emptyDir exceeds 100Mi, the kubelet's eviction manager detects the overuse and **evicts the pod** (terminates it). The pod's status will show `Evicted` with reason `EmptyDirSizeExceeded`. This is enforced periodically, not instantaneously, so brief spikes may not be caught immediately.

---

**31. Create a generic Secret from literals for a DB username/password, then show the stored values are base64-encoded, not encrypted.**

```bash
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password=s3cret
kubectl get secret db-creds -o yaml
```

Output shows:
```yaml
data:
  username: YWRtaW4=      # base64 of "admin"
  password: czNjcmV0      # base64 of "s3cret"
```

Secrets are only base64-encoded in etcd, **not encrypted** by default. Anyone with API access can decode them. For actual encryption, you need encryption-at-rest configuration or an external secrets manager.

---

**32. Create a Secret from files (--from-file) holding a certificate and key, and identify the resulting keys.**

```bash
kubectl create secret generic tls-files --from-file=cert.pem=./server.crt --from-file=key.pem=./server.key
kubectl get secret tls-files -o jsonpath='{.data}' | jq 'keys'
# ["cert.pem", "key.pem"]
```

The resulting Secret keys match the names specified before the `=` sign (or the filename if no alias is given). Each file's content is stored as a separate base64-encoded data entry under that key.

---

**33. Create a kubernetes.io/tls typed Secret from a cert/key pair and explain where it's consumed.**

```bash
kubectl create secret tls my-tls --cert=./server.crt --key=./server.key
```

This creates a Secret with type `kubernetes.io/tls` containing keys `tls.crt` and `tls.key`. It is consumed by Ingress controllers to terminate TLS — you reference it in an Ingress resource's `spec.tls[].secretName`. It can also be mounted into pods that need to serve HTTPS directly.

---

**34. Mount a Secret as a volume and explain the security trade-offs versus env vars.**

```yaml
volumeMounts:
- name: secret-vol
  mountPath: /secrets
  readOnly: true
volumes:
- name: secret-vol
  secret:
    secretName: db-creds
```

Volume-mounted Secrets are stored as tmpfs (in-memory), can be updated without pod restart, and are only accessible via the filesystem. Environment variables are visible in `/proc/<pid>/environ`, appear in crash dumps, and are often logged accidentally. Volume mounts are generally more secure, but both expose secrets to any process in the container.

---

**35. Use envFrom with a secretRef to load every key of a Secret as env vars at once.**

```yaml
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - secretRef:
        name: db-creds
```

This injects all keys from the `db-creds` Secret as environment variables (e.g., `username=admin`, `password=s3cret`). It avoids listing each key individually but means the pod spec doesn't document which variables the app expects. All keys must be valid environment variable names.

---

**36. Create a docker-registry (image-pull) Secret and reference it via imagePullSecrets; explain when it's required.**

```bash
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=myuser \
  --docker-password=mypass
```

```yaml
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: docker.io/private/myapp:latest
```

This is required when pulling from a private registry or when you need authenticated pulls to avoid rate limits. Without it, the kubelet cannot authenticate and the pull fails with `ImagePullBackOff`.

---

**37. Create a Secret with several keys and selectively mount only one of them into a pod.**

```yaml
volumes:
- name: secret-vol
  secret:
    secretName: db-creds
    items:
    - key: password
      path: db-password
```

```yaml
volumeMounts:
- name: secret-vol
  mountPath: /secrets
```

Only the `password` key is mounted as `/secrets/db-password`; other keys (`username`, etc.) are excluded. This is useful when a container should only have access to specific credentials, following the principle of least privilege.

---

**38. Create a ClusterIP Service for a Deployment and resolve it by DNS from another pod.**

```bash
kubectl expose deployment web --port=80 --type=ClusterIP
kubectl run tmp --rm -it --image=busybox -- nslookup web.default.svc.cluster.local
```

Output confirms the Service resolves to its ClusterIP. The FQDN format is `<service>.<namespace>.svc.cluster.local`. CoreDNS in the cluster provides this resolution, and all pods get `/etc/resolv.conf` configured to search the cluster domain.

---

**39. Create a NodePort Service and reach it via minikube service and $(minikube ip):<nodePort>.**

```bash
kubectl expose deployment web --port=80 --type=NodePort --name=web-np
minikube service web-np --url
# Or manually:
curl $(minikube ip):$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')
```

A NodePort Service opens a static port (30000–32767) on every node's IP. Traffic hitting `<nodeIP>:<nodePort>` is forwarded to the Service's target pods. In minikube, `minikube service` handles the tunneling needed on some drivers.

---

**40. Create a LoadBalancer Service, explain why EXTERNAL-IP is <pending> on minikube, and obtain access with minikube tunnel.**

```bash
kubectl expose deployment web --port=80 --type=LoadBalancer --name=web-lb
kubectl get svc web-lb  # EXTERNAL-IP: <pending>
# In another terminal:
minikube tunnel
kubectl get svc web-lb  # Now shows an IP
```

EXTERNAL-IP stays `<pending>` because minikube has no cloud provider load-balancer controller by default. `minikube tunnel` simulates one by assigning a routable IP from the host network. In a real cloud, the cloud controller manager provisions an actual load balancer.

---

**41. Create a headless Service (clusterIP: None) for a StatefulSet and show it returns per-pod A records instead of a single VIP.**

```bash
kubectl run tmp --rm -it --image=busybox -- nslookup redis.default.svc.cluster.local
```

Instead of a single A record (VIP), you get multiple A records — one for each pod (e.g., IPs of redis-0, redis-1, redis-2). Individual pods are resolvable as `redis-0.redis.default.svc.cluster.local`. This enables clients to discover and connect to specific pods directly.

---

**42. Inspect a Service's Endpoints/EndpointSlice and explain how they're populated from the pod selector.**

```bash
kubectl get endpoints web
kubectl get endpointslices -l kubernetes.io/service-name=web
```

The Endpoints controller watches pods matching the Service's `spec.selector` labels. Only pods that are Running and pass their readiness probe are added as endpoints (IP:port pairs). If labels don't match or pods aren't ready, the endpoints list is empty and the Service routes to nothing.

---

**43. Demonstrate cross-namespace access using the FQDN, and show the short name fails from another namespace.**

```bash
# From namespace "other":
kubectl run tmp -n other --rm -it --image=busybox -- sh
# Inside:
wget -qO- web.default.svc.cluster.local  # WORKS (FQDN)
wget -qO- web                              # FAILS (short name resolves in "other" ns)
```

Short Service names only resolve within the same namespace because the search domain in `/etc/resolv.conf` includes `<pod-namespace>.svc.cluster.local`. Cross-namespace requires the full FQDN: `<svc>.<namespace>.svc.cluster.local`.

---

**44. Compare kubectl port-forward to a Service versus to a Pod — explain the difference and when each fits.**

```bash
kubectl port-forward svc/web 8080:80   # forwards to a random backend pod via Service
kubectl port-forward pod/web-abc 8080:80  # forwards directly to that specific pod
```

Port-forward to a Service lets Kubernetes pick a backend (useful for general access/testing). Port-forward to a Pod targets one specific instance (useful for debugging a particular pod). Both create a tunnel from your localhost — neither goes through the cluster network's Service routing or load balancing properly; it's a development/debug tool, not production access.

---

**45. Explain the difference between a Service's port, targetPort, and nodePort.**

- **port**: The port the Service listens on within the cluster (the ClusterIP port clients connect to).
- **targetPort**: The port on the actual pod/container where traffic is forwarded to (can differ from `port`).
- **nodePort**: (NodePort/LoadBalancer only) The static port opened on every node's external IP for outside access.

Traffic flow: `client → nodePort (node IP) → port (ClusterIP) → targetPort (pod)`. This decoupling allows the Service to present a stable port while the app can listen on any port internally.

---

**46. Verify connectivity to a ClusterIP Service from a throwaway debug pod with wget/nc.**

```bash
kubectl run tmp --rm -it --image=busybox -- sh
# Inside the pod:
wget -qO- http://web.default.svc.cluster.local
# Or:
nc -zv web 80
```

If wget returns the nginx welcome page and nc shows "open", connectivity is confirmed. This proves DNS resolution, endpoint routing, and pod health are all functioning correctly from within the cluster network.

---

**47. Create two Deployments and one Service that load-balances across both by sharing a common label; prove requests hit both.**

```bash
kubectl create deployment web1 --image=nginx --replicas=1
kubectl create deployment web2 --image=httpd --replicas=1
kubectl label pods -l app=web1 role=backend
kubectl label pods -l app=web2 role=backend
kubectl expose deployment web1 --name=shared-svc --port=80 --selector=role=backend --type=ClusterIP
# Actually, create the service with the shared selector:
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shared-svc
spec:
  selector:
    role: backend
  ports:
  - port: 80
```

Running repeated `wget` from a debug pod shows responses alternating between nginx and httpd, proving the Service load-balances across both Deployments' pods based on the shared label.

---

**48. Systematically diagnose why a pod can't reach a Service: DNS → endpoints → selector labels → readiness → port.**

```bash
# 1. DNS: Can the pod resolve the service name?
kubectl exec <pod> -- nslookup <svc-name>
# 2. Endpoints: Does the Service have backends?
kubectl get endpoints <svc-name>
# 3. Selector: Do pod labels match the Service selector?
kubectl get pods --show-labels; kubectl get svc <svc-name> -o yaml | grep selector
# 4. Readiness: Are pods passing readiness probes?
kubectl get pods  # check READY column
# 5. Port: Is targetPort correct and is the app actually listening?
kubectl exec <pod> -- netstat -tlnp
```

Follow this order: if DNS fails, check CoreDNS; if endpoints are empty, labels or readiness are wrong; if endpoints exist but connection fails, the port/targetPort is misconfigured.

---

**49. Expose a service with a NodePort and verify it works.**

```bash
kubectl expose deployment web --type=NodePort --port=80 --name=web-nodeport
kubectl get svc web-nodeport
curl $(minikube ip):$(kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
```

The curl command should return the nginx welcome page, confirming the NodePort is open on the node and correctly forwarding to the pod's port 80.

---

**50. Add an HTTP livenessProbe hitting / on port 80 to an nginx Deployment and verify via kubectl describe pod.**

```yaml
containers:
- name: nginx
  image: nginx:1.27
  livenessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 10
```

```bash
kubectl describe pod <pod-name> | grep -A5 Liveness
```

The output shows the probe configuration and its status (Success). If the probe fails consecutively (default 3 times), the kubelet restarts the container.

---

**51. Add a tcpSocket readiness probe to a redis container on port 6379 and verify it.**

```yaml
containers:
- name: redis
  image: redis:7
  readinessProbe:
    tcpSocket:
      port: 6379
    initialDelaySeconds: 5
    periodSeconds: 10
```

```bash
kubectl describe pod <pod-name> | grep -A5 Readiness
```

The tcpSocket probe checks if the port is accepting connections. Until it succeeds, the pod's IP is not added to Service endpoints, so no traffic is routed to it. This prevents sending requests to a container that isn't ready to serve.

---

**52. Configure a startup probe with a failureThreshold × periodSeconds budget for a ~2-minute boot and justify the numbers.**

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 24
  periodSeconds: 5
```

24 × 5 = 120 seconds (2 minutes) boot budget. During this time, liveness and readiness probes are disabled so they don't kill a slow-starting app. Once the startup probe succeeds, the other probes take over. This is ideal for legacy apps or Java services with long initialization times.

---

**53. Explain the default behavior when no probes are defined at all.**

When no probes are defined, Kubernetes assumes the container is ready and alive as soon as its process starts (PID 1 is running). The pod is immediately added to Service endpoints and receives traffic, even if the application inside hasn't finished initializing. If the process crashes, the kubelet restarts it, but it cannot detect hangs, deadlocks, or partial failures. This means unresponsive containers continue receiving traffic until they actually crash.

---


# LO5 — Solve problems with application shipping by using containers

**1. podman run -d --name web -p 80:8080 docker.io/library/nginx**
*(nginx listens on 80; the site isn't reachable on the host port you expected — what's reversed?)*

**Mistake:** The `-p` flag format is `hostPort:containerPort`. Here, host port 80 maps to container port 8080, but nginx listens on port 80 inside the container, not 8080.

**Fix:**
```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
```

The ports were reversed — you wanted host 8080 → container 80, but wrote host 80 → container 8080 (where nothing is listening). Always remember: `-p HOST:CONTAINER`.

---

**2. podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql**
*(the container exits during init — inspect podman logs.)*

**Mistake:** The `-e` flag requires a value assignment. Writing `-e MYSQL_ROOT_PASSWORD` without `=value` sets the variable to an empty string, and MySQL refuses to start without a non-empty root password.

**Fix:**
```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD=mysecret docker.io/library/mysql
```

`podman logs db` would show: "MYSQL_ROOT_PASSWORD must be set and non-empty." Always use `-e KEY=VALUE`.

---

**3. podman run -d --name app --network host -p 8080:80 docker.io/library/nginx**
*(why is the -p flag effectively ignored here?)*

**Mistake:** When using `--network host`, the container shares the host's network namespace directly — there is no network isolation, so port mapping (`-p`) is meaningless and silently ignored.

**Fix:**
```bash
# Either use host networking (nginx is reachable on host port 80 directly):
podman run -d --name app --network host docker.io/library/nginx
# Or use port mapping without host networking:
podman run -d --name app -p 8080:80 docker.io/library/nginx
```

You cannot combine `--network host` with `-p`; pick one approach.

---

**4. podman run -d --name c1 docker.io/library/busybox**
*(the container goes straight to Exited (0) — why, and how do you keep it running?)*

**Mistake:** Busybox's default entrypoint is `sh`, which exits immediately when there's no TTY and no command to run. A container stops when its PID 1 process exits.

**Fix:**
```bash
podman run -d --name c1 docker.io/library/busybox sleep 3600
# Or for interactive use:
podman run -it --name c1 docker.io/library/busybox sh
```

You need to provide a long-running command (like `sleep` or `tail -f /dev/null`) to keep a minimal container alive in detached mode.

---

**5. podman run --rm -d --name job docker.io/library/alpine echo hello**
*(then podman logs job fails — explain the --rm + detached gotcha.)*

**Mistake:** `--rm` removes the container automatically when it exits. Since `echo hello` completes almost instantly, by the time you run `podman logs job`, the container has already been removed.

**Fix:**
```bash
# Option 1: Run in foreground to see output directly:
podman run --rm --name job docker.io/library/alpine echo hello
# Option 2: Remove --rm and manually inspect:
podman run -d --name job docker.io/library/alpine echo hello
podman logs job
podman rm job
```

With `--rm` and `-d` combined, fast-exiting containers vanish before you can inspect them.

---

**6. podman run -d -p 8080:80 --memory 8m docker.io/library/mysql**
*(the database never becomes healthy — what limit is the problem?)*

**Mistake:** 8MB of memory is far too little for MySQL to initialize — it needs at least ~256MB–512MB. The container is being OOM-killed repeatedly during startup.

**Fix:**
```bash
podman run -d -p 3306:3306 --memory 512m -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
```

`podman inspect` or `podman logs` would show OOM kills. Always set memory limits that match the application's minimum requirements.

---

**7. podman run -d --name a docker.io/library/alpine sleep 1d / podman run -d --name b docker.io/library/alpine sleep 1d / podman exec a ping b**
*(name resolution fails — why, on the default network, and how do you fix it?)*

**Mistake:** On Podman's default network, container name DNS resolution is not enabled. Containers can't resolve each other by name unless they are on a user-defined network.

**Fix:**
```bash
podman network create mynet
podman run -d --name a --network mynet docker.io/library/alpine sleep 1d
podman run -d --name b --network mynet docker.io/library/alpine sleep 1d
podman exec a ping b  # works now
```

User-defined networks run an embedded DNS server that resolves container names; the default bridge network does not.

---

**8. podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx**
*(the files aren't visible in the container, or SELinux denies access — what mount flag is missing?)*

**Mistake:** On SELinux-enabled systems (RHEL/Fedora), the container's SELinux context cannot read host-mounted files unless you add the `:Z` or `:z` suffix to relabel the volume.

**Fix:**
```bash
podman run -d --name web -v ./html:/usr/share/nginx/html:Z docker.io/library/nginx
```

`:Z` applies a private SELinux label (only this container can access it); `:z` applies a shared label. Without this flag, SELinux blocks access even though Unix permissions may allow it.

---

**9–10. FROM debian:12 / RUN apt-get install -y nginx**
*(the build fails to find the package — what's missing before install?)*

**Mistake:** The Debian base image ships without a package index cache. Running `apt-get install` without first updating the package lists causes "Unable to locate package nginx."

**Fix:**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y nginx
```

Always run `apt-get update` before `apt-get install` in the same `RUN` layer to ensure the package index is current and available.

---

**11. FROM python:3.11 / COPY app.py / CMD python app.py**
*(shell vs exec form?)*

**Mistake:** `CMD python app.py` uses shell form, which runs as `/bin/sh -c "python app.py"`. The Python process is not PID 1, so it doesn't receive SIGTERM signals — `podman stop` sends SIGTERM to sh, not your app, causing unclean shutdowns.

**Fix:**
```dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD ["python", "app.py"]
```

Exec form (`CMD ["python", "app.py"]`) makes Python PID 1, allowing it to receive and handle signals properly for graceful shutdown.

---

**12. FROM node:20 / COPY . /app / WORKDIR /app / RUN npm install**
*(every source change triggers a full npm install — reorder for layer caching.)*

**Mistake:** Copying all source files before `npm install` means any code change invalidates the `COPY` layer cache, forcing a full reinstall of dependencies every build.

**Fix:**
```dockerfile
FROM node:20
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

By copying only dependency files first, the `npm install` layer is cached as long as dependencies don't change. Source code changes only invalidate the final `COPY` layer.

---

**13. FROM alpine:3.20 / ENV PATH=/app/bin / RUN apk add --no-cache curl**
*(after this, curl/sh aren't found — what did setting PATH break?)*

**Mistake:** `ENV PATH=/app/bin` completely replaces the PATH instead of appending to it. System binaries in `/usr/bin`, `/bin`, etc. are no longer discoverable.

**Fix:**
```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin:$PATH
RUN apk add --no-cache curl
```

Always append to PATH (`/app/bin:$PATH`) rather than overwriting it, or you lose access to all standard system utilities.

---

**14–15. FROM ubuntu:24.04 / EXPOSE 8080 / CMD ["python3", "-m", "http.server", "3000"]**
*(you map -p 8080:8080 but nothing answers — what mismatch is there, and what does EXPOSE actually do?)*

**Mistake:** The app listens on port 3000 but you're mapping and expecting port 8080. `EXPOSE` is only documentation metadata — it does NOT actually open or publish a port. The real issue is the mismatch between the app's listen port (3000) and the mapped port (8080).

**Fix:**
```dockerfile
FROM ubuntu:24.04
EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]
```
```bash
podman run -d -p 8080:3000 myimage  # host 8080 → container 3000
```

`EXPOSE` is informational only; the `-p` mapping and the app's actual listen port must agree.

---

**16. FROM debian:12 / RUN apt-get update && apt-get install -y build-essential**
*(the image is huge — how do you cut the size in the same layer?)*

**Mistake:** The apt cache and package lists remain in the layer, bloating the image by hundreds of MBs unnecessarily.

**Fix:**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
```

Cleaning up in the same `RUN` statement ensures the garbage doesn't persist in the layer. Doing it in a separate `RUN` doesn't reduce size because layers are additive.

---

**17. FROM python:3.11 / USER appuser / COPY requirements.txt / RUN pip install**
*(permission denied errors — what's wrong with the USER/COPY ordering and ownership?)*

**Mistake:** `USER appuser` switches to a non-root user before `COPY` and `RUN pip install`. The user may not exist yet, and even if it does, it lacks permission to write to `/app/` or install packages globally.

**Fix:**
```dockerfile
FROM python:3.11
RUN useradd -m appuser
COPY --chown=appuser:appuser requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
USER appuser
WORKDIR /app
```

Create the user first, do privileged operations (installs) as root, then switch to `USER appuser` as the last step before `CMD`.

---

**18. FROM golang:1.22 / COPY . /src / RUN go build -o app . / CMD ["/src/app"]**
*(the final image is ~1 GB — how would a multi-stage build fix it?)*

**Mistake:** The final image includes the entire Go toolchain, source code, and build cache — all unnecessary at runtime.

**Fix:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o app .

FROM alpine:3.20
COPY --from=builder /src/app /app
CMD ["/app"]
```

The multi-stage build produces a final image of ~10–20MB (alpine + binary) instead of ~1GB because the Go SDK is left behind in the builder stage.

---

**19. A pod is stuck Pending. Walk through the diagnosis.**

```bash
kubectl describe pod <pod-name>
# Check Events section for:
# - "Insufficient cpu/memory" → requests exceed node capacity
# - "no nodes available to schedule" → nodeSelector/affinity mismatch
# - "pod has unbound PersistentVolumeClaims" → PVC not bound
# - "0/N nodes are available: taint..." → tolerations missing
kubectl get events --sort-by=.lastTimestamp
```

The Events section in `describe pod` always reveals the scheduling failure reason. Fix it by adjusting resources, labels, PVCs, or tolerations accordingly.

---

**20. Find a deployment from older tasks, remove a couple of spaces and try to deploy it. Identify the cause in the error messages and fix it.**

```bash
kubectl apply -f broken-deployment.yaml
# Error: error parsing YAML: did not find expected key / mapping values not allowed
```

YAML is indentation-sensitive — removing spaces breaks the document structure. The error message indicates the exact line/column. Fix by restoring proper indentation (2 spaces per level, consistent nesting). Use `kubectl apply --dry-run=server -f file.yaml` to validate before applying.

---

**21. A pod is in ImagePullBackOff. Diagnose the cause and fix it.**

```bash
kubectl describe pod <pod-name>
# Events: "Failed to pull image: rpc error... not found" → typo or missing tag
# Or: "unauthorized: authentication required" → private registry, no pull secret
```

Common causes: misspelled image name, nonexistent tag, or private registry without `imagePullSecrets`. Fix by correcting the image reference or adding the appropriate pull secret. Verify with `podman pull <image>` locally first.

---

**22. A pod is in CrashLoopBackOff. Use kubectl logs --previous to read the last crash and fix the root cause.**

```bash
kubectl logs <pod-name> --previous
# Shows the stdout/stderr from the last crashed instance
kubectl describe pod <pod-name>  # check exit code and reason
```

CrashLoopBackOff means the container starts, crashes, and Kubernetes keeps restarting it with exponential backoff. The `--previous` flag reads logs from the terminated instance. Common causes: missing env vars, wrong command, permission errors, or the app crashing on startup.

---

**23. A pod's last state shows OOMKilled. Confirm and fix it.**

```bash
kubectl get pod <pod-name> -o yaml | grep -A5 lastState
# Shows: reason: OOMKilled, exitCode: 137
kubectl describe pod <pod-name> | grep -i oom
```

OOMKilled (exit code 137) means the container exceeded its memory limit and the kernel killed it. Fix by either raising `resources.limits.memory` to match the app's actual needs, or fixing a memory leak in the application. The limit must accommodate peak usage.

---

**24. A Deployment's pods never become Ready. Trace it to a failing readiness probe and correct the probe or the app.**

```bash
kubectl describe pod <pod-name>
# Events: "Readiness probe failed: HTTP probe failed with statuscode: 404"
# Or: "Readiness probe failed: dial tcp: connection refused"
```

The readiness probe's path, port, or protocol doesn't match what the app serves. Fix by adjusting the probe's `httpGet.path`/`port` to match the app's actual health endpoint, or fix the app to respond correctly on the configured path.

---

**25. A Service returns nothing. Diagnose a Service/pod label mismatch using kubectl get endpoints.**

```bash
kubectl get endpoints <svc-name>
# Shows: <none> — no endpoints!
kubectl get svc <svc-name> -o yaml | grep -A3 selector
kubectl get pods --show-labels
```

If endpoints are empty, the Service's selector doesn't match any pod labels. Fix by aligning the Service's `spec.selector` with the pods' `metadata.labels`. A single character mismatch (e.g., `app: web` vs `app: Web`) breaks the binding.

---

**26. Given a manifest where spec.selector.matchLabels doesn't match spec.template.metadata.labels, explain the error and fix it.**

```
Error: The Deployment "web" is invalid: spec.template.metadata.labels: Invalid value: ... `selector` does not match template `labels`
```

Kubernetes requires that `spec.selector.matchLabels` is a subset of `spec.template.metadata.labels` — otherwise the Deployment would never find its own pods. Fix by making the labels identical or ensuring the selector labels are all present in the template labels.

---

**27. Given a manifest whose resources.requests exceed any node's capacity, explain why the pod is Pending and fix it.**

```bash
kubectl describe pod <pod-name>
# Events: "0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory"
```

The scheduler cannot place the pod because no node has enough allocatable resources to satisfy the requests. Fix by lowering `resources.requests.cpu` and `resources.requests.memory` to values that fit within your cluster's available capacity, or add larger nodes.

---

**28. Given a pod mounting a PVC that doesn't exist, diagnose the FailedMount/Pending and create the PVC.**

```bash
kubectl describe pod <pod-name>
# Events: "persistentvolumeclaim 'my-pvc' not found" → pod stays Pending
```

**Fix:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
```

Once the PVC is created and bound to a PV, the pod can mount it and transitions from Pending to Running.

---

**29. A container based on ubuntu with no long-running command shows Completed/CrashLoopBackOff. Explain why and add a proper command.**

Ubuntu's default command (`bash`) exits immediately when there's no TTY or input, so the container completes with exit code 0. Kubernetes restarts it, it exits again — causing CrashLoopBackOff.

**Fix:**
```yaml
command: ["sleep", "infinity"]
# Or a real application command:
command: ["python3", "server.py"]
```

Every container needs a long-running foreground process as PID 1 to stay alive.

---

**30. Use kubectl get events --sort-by=.lastTimestamp to triage everything that happened to a pod over time.**

```bash
kubectl get events --sort-by=.lastTimestamp --field-selector involvedObject.name=<pod-name>
# Or for all events in namespace:
kubectl get events --sort-by=.lastTimestamp
# Newer clusters:
kubectl events --for pod/<pod-name>
```

Events show the chronological history: scheduling, pulling images, starting containers, probe failures, OOM kills, etc. They are the first place to look when diagnosing any pod issue. Events expire after ~1 hour by default.

---

**31. Use kubectl exec -it to get a shell in a pod and debug DNS with nslookup and cat /etc/resolv.conf.**

```bash
kubectl exec -it <pod-name> -- sh
# Inside:
cat /etc/resolv.conf
# Shows: nameserver 10.96.0.10, search default.svc.cluster.local svc.cluster.local cluster.local
nslookup my-service
nslookup my-service.default.svc.cluster.local
```

`/etc/resolv.conf` shows the cluster DNS server (CoreDNS ClusterIP) and search domains. If nslookup fails, CoreDNS may be down, or the Service doesn't exist.

---

**32. Use an ephemeral debug container (kubectl debug) to troubleshoot a no-shell/distroless container.**

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
# Inside the ephemeral container:
ps aux        # see processes in target container
netstat -tlnp # check listening ports
cat /proc/1/environ  # inspect environment
```

Ephemeral containers attach to a running pod without restarting it, sharing the target container's PID/network namespace. This lets you debug distroless/minimal images that have no shell or utilities.

---

**33. Spin up a throwaway pod to test connectivity to a Service from inside the cluster.**

```bash
kubectl run tmp --rm -it --image=busybox -- sh
# Inside:
wget -qO- http://web.default.svc.cluster.local
nc -zv web 80
nslookup web
```

The `--rm` flag deletes the pod when you exit. This is the standard way to verify in-cluster Service reachability, DNS resolution, and network policies without leaving debug pods behind.

---

**34. A node shows NotReady. List what you'd check and inspect it on minikube.**

```bash
kubectl describe node <node-name>  # check Conditions: Ready, MemoryPressure, DiskPressure
kubectl get events --field-selector involvedObject.kind=Node
sudo minikube logs | grep kubelet
# Check:
# 1. kubelet running? (systemctl status kubelet)
# 2. Disk pressure? (df -h)
# 3. CNI plugin healthy? (ls /etc/cni/net.d/)
# 4. Container runtime responsive? (crictl ps)
```

NotReady typically means the kubelet stopped heartbeating — caused by kubelet crash, disk/memory pressure, or CNI failure. The node's Conditions section reveals which pressure triggered it.

---

**35. A pod is stuck Terminating. Explain finalizers and the grace period, then force-delete it.**

Finalizers are metadata keys that prevent resource deletion until a controller clears them. The grace period (default 30s) gives the container time to shut down gracefully via SIGTERM before SIGKILL.

```bash
# Force-delete:
kubectl delete pod <pod-name> --grace-period=0 --force
```

**Risks:** Force-deleting bypasses graceful shutdown — the app may not flush data, release locks, or deregister from service discovery. The pod may also still be running on the node even though the API object is gone.

---

**36. Given a manifest with the wrong apiVersion/kind pairing (e.g. apps/v1 + Pod), explain the validation error and correct it.**

```
Error: no matches for kind "Pod" in version "apps/v1"
```

Pods belong to the core API group (`apiVersion: v1`), not `apps/v1`. The `apps/v1` group is for Deployments, StatefulSets, DaemonSets, and ReplicaSets. Fix by changing to `apiVersion: v1` for Pod, or if you meant a Deployment, change `kind: Deployment`.

---

**37. A pod fails with CreateContainerConfigError because a configMapKeyRef key is missing. Diagnose and fix.**

```bash
kubectl describe pod <pod-name>
# Events: "couldn't find key 'DB_HOST' in ConfigMap 'app-config'"
kubectl get configmap app-config -o yaml
```

The pod's env var references a key that doesn't exist in the ConfigMap. Fix by either adding the missing key to the ConfigMap (`kubectl edit configmap app-config`) or correcting the `configMapKeyRef.key` in the pod spec. Alternatively, mark it as `optional: true`.

---

**38. A pod can't mount a Secret that lives in a different namespace. Explain namespace scoping and fix it.**

Secrets (and ConfigMaps) are namespace-scoped — a pod in namespace `app` cannot reference a Secret in namespace `infra`. Kubernetes enforces strict namespace isolation for these resources.

**Fix:** Either recreate/copy the Secret into the pod's namespace, or use a tool like `kubernetes-replicator` or External Secrets Operator to sync secrets across namespaces. There is no cross-namespace volume mount.

---

**39. A rollout is stuck with ProgressDeadlineExceeded. Use kubectl describe deployment / rollout status to find the cause and remediate.**

```bash
kubectl rollout status deployment/web  # "has timed out progressing"
kubectl describe deployment web        # Conditions: Progressing=False, reason: ProgressDeadlineExceeded
kubectl get pods -l app=web            # new pods stuck in CrashLoopBackOff or ImagePullBackOff
```

The Deployment couldn't complete within `spec.progressDeadlineSeconds` (default 600s). Find why new pods aren't becoming Ready (bad image, crash, probe failure), fix the issue, then the rollout resumes automatically. Or run `kubectl rollout undo` to revert.

---

**40. Catch indentation/field typos before applying with --dry-run=server / --validate=true, then fix them.**

```bash
kubectl apply -f manifest.yaml --dry-run=server
# Error: unknown field "imagePullpolicy" (should be "imagePullPolicy")
# Error: mapping values not allowed at line X (indentation error)
kubectl apply -f manifest.yaml --validate=true
```

`--dry-run=server` sends the manifest to the API server for full validation without persisting it. This catches typos like `imagePullpolicy` (wrong case), misnested fields (ports under the wrong level), and invalid values. Always validate before applying to production.

---

**41. An app's logs say it can't reach its database Service. Verify in order and identify the broken link.**

```bash
# 1. Is the DB pod running?
kubectl get pods -l app=db
# 2. Does the Service exist?
kubectl get svc db-service
# 3. Are endpoints populated?
kubectl get endpoints db-service
# 4. Does DNS resolve?
kubectl exec <app-pod> -- nslookup db-service
# 5. Is the correct port being targeted?
kubectl get svc db-service -o yaml | grep targetPort
```

Follow this chain sequentially — the first step that fails is your broken link. Most commonly it's empty endpoints (label mismatch) or wrong targetPort.

---

**42. Read a container's state/lastState and restartCount with kubectl get pod -o jsonpath to determine root cause of repeated restarts.**

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

A high restartCount with `reason: OOMKilled` (exit 137) points to memory issues; `reason: Error` (exit 1) points to application crashes. The `lastState` captures why the previous instance died, which is crucial when the current instance is running fine after restart.

---

**43. Compare OpenShift Route to Ingress.**

An **Ingress** is a standard Kubernetes resource that defines HTTP/HTTPS routing rules, requiring a separate Ingress controller (nginx, traefik) to implement them. An OpenShift **Route** is a first-class OpenShift resource handled by the built-in HAProxy router, supporting features like TLS re-encryption, blue-green deployments, and sticky sessions out-of-the-box. Routes were created before Ingress existed and are more feature-rich by default; OpenShift also supports Ingress by translating it into Routes internally. The key difference is that Routes are tightly integrated and require no additional controller installation.

---

# LO6 — Evaluate the use of selected container orchestration systems (theoretical)

**1. Analyze the networking model of Kubernetes versus Docker's network.**

Kubernetes assigns every pod a unique, routable IP address — all pods can communicate directly without NAT, forming a flat network enforced by CNI plugins (Calico, Flannel, Cilium). Docker uses a bridge network by default where containers get private IPs behind the host's NAT, requiring explicit port mapping (`-p`) for external access and user-defined networks for container-to-container DNS. Kubernetes Services provide stable virtual IPs and DNS-based discovery, whereas Docker relies on container names on shared networks or external tools like Docker Compose for service discovery. Kubernetes' model is inherently multi-host and scalable; Docker's default networking is single-host unless using Swarm overlay networks.

---

**2. Evaluate Kubernetes storage abstraction (CSI, StorageClasses, dynamic provisioning) against Podman's storage plugins. Which is more flexible for stateful workloads, and why?**

Kubernetes offers a mature storage ecosystem: CSI drivers abstract any backend (cloud disks, NFS, Ceph), StorageClasses enable dynamic provisioning on demand, and PVCs decouple apps from infrastructure. Podman's storage is limited to local bind mounts, named volumes on the host's filesystem, and basic volume plugins — there's no concept of dynamic provisioning or claim-based abstraction. For stateful workloads, Kubernetes is far more flexible because it supports replication, snapshotting (VolumeSnapshots), access mode enforcement, and topology-aware scheduling. Podman is adequate for single-host development but lacks the multi-node persistence guarantees needed for production stateful services.

---

**3. Evaluate the operator pattern (CRDs + controllers) for managing stateful services versus using only k8s manifests, in terms of control and effort.**

Operators encode domain-specific operational knowledge (backup, failover, scaling) into a controller that automates day-2 tasks that raw manifests cannot express. Plain manifests require manual intervention for operations like database failover, certificate rotation, or version upgrades. Operators provide superior control and consistency but require significant development/maintenance effort (writing Go controllers, testing reconciliation loops). For complex stateful services (databases, message queues), operators drastically reduce operational burden; for simple stateless apps, plain manifests with Deployments are sufficient and less complex.

---

**4. Assess the developer experience of plain Kubernetes (kubectl + manifests) versus OpenShift's Source-to-Image (S2I) and developer console. What does each optimize for?**

Plain Kubernetes optimizes for flexibility and transparency — developers write YAML manifests, build their own container images, and have full control over every resource, but the learning curve is steep. OpenShift S2I optimizes for speed-to-deployment — developers push source code and OpenShift automatically builds images, creates Deployments, and exposes Routes without writing Dockerfiles or manifests. The OpenShift developer console provides a GUI for topology visualization, log streaming, and pipeline management. Kubernetes favors teams wanting maximum control; OpenShift favors organizations wanting guardrails and faster onboarding at the cost of some flexibility.

---

**5. Critically evaluate migrating a workload from OpenShift to Kubernetes.**

OpenShift uses standard Kubernetes resources underneath, so Deployments, Services, ConfigMaps, and Secrets transfer directly. However, OpenShift-specific resources — Routes, DeploymentConfigs, BuildConfigs, S2I — have no direct Kubernetes equivalent and must be replaced with Ingress, Deployments, and external CI/CD pipelines. Security Context Constraints (SCCs) must be translated to PodSecurityStandards/Admission policies, which are less restrictive by default. Image streams need replacement with direct registry references. The migration is feasible but requires rearchitecting the build/deploy pipeline and hardening the security posture manually.

---

**6. Evaluate running CI/CD build agents (Jenkins agents, GitLab runners, Tekton) inside Kubernetes versus on dedicated VMs, considering isolation, autoscaling, and noisy-neighbor effects.**

Kubernetes-based agents scale dynamically via pod autoscaling — agents spin up on demand and terminate after jobs, eliminating idle resource waste. VMs provide stronger isolation (separate kernels) but are slower to provision and costly when idle. Noisy-neighbor effects are more pronounced in Kubernetes if resource requests/limits aren't properly set, as builds share node resources. Kubernetes wins on elasticity and cost efficiency; VMs win on security isolation for untrusted workloads. Tekton, being Kubernetes-native, integrates most naturally, while Jenkins/GitLab runners need plugins to leverage pod-based scaling.

---

**7. How does Kubernetes integrate with cloud environments and dynamic capacity provisioning?**

Kubernetes integrates with clouds via the Cloud Controller Manager, which provisions LoadBalancer Services (ELB, ALB, Cloud LB), attaches persistent disks (EBS, Azure Disk, GCE PD) via CSI drivers, and manages node lifecycle. Cluster Autoscaler dynamically adds/removes nodes based on pending pod resource requests, scaling the underlying VM pool. Managed services (EKS, AKS, GKE) further abstract control-plane management, auto-upgrades, and integrated IAM. This tight integration means Kubernetes can elastically scale infrastructure to match workload demands without manual intervention.

---

**8. Critically evaluate the default security and hardening of OpenShift versus vanilla Kubernetes. Why do enterprises in regulated industries often choose OpenShift?**

OpenShift ships with Security Context Constraints (SCCs) that deny root containers by default, mandatory RBAC, built-in OAuth/identity provider integration, and an integrated image registry with vulnerability scanning. Vanilla Kubernetes runs containers as root by default, has no built-in admission enforcement, and requires manual setup of OPA/Gatekeeper or Pod Security Admission. Regulated industries (finance, healthcare) choose OpenShift because it provides an enterprise-supported, hardened-by-default platform with compliance certifications (FIPS, STIGs), reducing the effort to pass audits. The trade-off is vendor lock-in and licensing cost.

---

**9. Critically evaluate the learning curve and required skill set of OpenShift versus plain Kubernetes. Does OpenShift's added abstraction help or hinder a team new to containers?**

OpenShift's abstractions (S2I, developer console, integrated CI/CD) lower the initial barrier for developers who just want to deploy code without learning Dockerfiles or YAML. However, the added layer of OpenShift-specific concepts (Routes, BuildConfigs, SCCs, ImageStreams) on top of Kubernetes means operators must learn both systems. For teams new to containers, OpenShift helps initially by hiding complexity, but can hinder deeper understanding and troubleshooting when issues arise beneath the abstraction layer. Long-term, teams benefit from understanding plain Kubernetes first, then leveraging OpenShift's conveniences.

---

**10. Compare the resource footprint of full Kubernetes versus k3s for a small on-premise deployment. When is full Kubernetes overkill?**

Full Kubernetes (kubeadm) requires minimum ~2GB RAM per control-plane node, runs etcd as a separate process, and expects dedicated master nodes — overhead is significant for small clusters. k3s bundles everything into a single ~70MB binary, uses SQLite instead of etcd by default, and runs comfortably on 512MB RAM nodes (Raspberry Pi, edge devices). Full Kubernetes is overkill when you have fewer than ~10 nodes, limited hardware, or edge/IoT deployments. k3s provides the same API compatibility with a fraction of the footprint, making it ideal for small on-premise or resource-constrained environments.

---

**11. Defend whether a small team should use Docker Compose on a single host or move straight to Kubernetes, considering complexity, scaling needs, and resilience.**

A small team with a single-host application should start with Docker Compose — it's simple, requires no cluster infrastructure, has minimal operational overhead, and provides sufficient service orchestration for development and small production workloads. Kubernetes adds significant complexity (networking, RBAC, storage, upgrades) that a 2-3 person team cannot justify unless they need multi-node scaling, self-healing across hosts, or zero-downtime rolling updates. Move to Kubernetes when you outgrow one host, need horizontal auto-scaling, or require high availability. Starting with Compose and migrating later is a valid, pragmatic path.

---

**12. Analyze vendor lock-in across Kubernetes (open source / CNCF) and OpenShift (Red Hat). How does the choice affect long-term flexibility?**

Kubernetes is CNCF-governed, fully open-source, and supported by every major cloud — workloads are portable between EKS, AKS, GKE, and self-managed clusters with minimal changes. OpenShift adds proprietary layers (subscription-based support, OperatorHub, specialized build system) that create dependency on Red Hat's ecosystem and pricing. However, OpenShift's core is still Kubernetes, so standard resources remain portable. The lock-in risk with OpenShift lies in using OpenShift-specific features (Routes, SCCs, BuildConfigs) that don't transfer cleanly. For long-term flexibility, plain Kubernetes preserves maximum portability; OpenShift trades some portability for integrated enterprise features.

---

**13. Defend a recommendation of OpenShift over vanilla Kubernetes for a company that wants an integrated, vendor-supported platform with built-in developer and operations tooling.**

OpenShift provides a complete platform out-of-the-box: integrated CI/CD (Tekton/Pipelines), developer console, built-in monitoring (Prometheus/Grafana), log aggregation (EFK), service mesh (Istio), and image registry — all pre-configured and supported by Red Hat. Vanilla Kubernetes requires assembling each component separately, integrating them, and maintaining them independently without vendor backing. For enterprises prioritizing time-to-value, operational consistency, and having a single vendor SLA for the entire stack, OpenShift's premium is justified. The operations team gets a unified upgrade path and 24/7 support rather than community-only help for a patchwork of tools.

---

**14. Compare storage provisioning between Kubernetes and regular VM workloads.**

In VM environments, storage is typically provisioned manually by admins — attaching disks via hypervisor/cloud console, formatting, and mounting them into VMs as fixed block devices. Kubernetes abstracts this via StorageClasses and PVCs: developers request storage declaratively, and the CSI driver dynamically provisions and attaches the appropriate volume without admin intervention. Kubernetes also provides portability (same PVC spec works across clouds), automatic reclaim policies, and snapshot capabilities. VMs offer simpler direct-attach models but lack self-service provisioning, lifecycle automation, and the declarative abstraction that Kubernetes provides.

---

**15. A startup with two engineers must ship a containerized product quickly and cheaply. Recommend container solution and justify your choice on cost and operational overhead.**

I recommend a **managed container service** like AWS Fargate, Google Cloud Run, or Azure Container Apps. These eliminate cluster management entirely — no nodes to patch, no control plane to maintain — the team just pushes container images and defines scaling rules. Cost is pay-per-use (billed per vCPU-second), ideal for startups with variable traffic. Alternatively, Docker Compose on a single VM works for the earliest stage. Full Kubernetes (even managed EKS/GKE) adds operational overhead that two engineers cannot sustain while also building product.

---

**16. Compare Docker and Podman in terms of architecture (daemon vs daemonless) and rootless security. Why might a security-conscious team prefer Podman?**

Docker runs a persistent root daemon (`dockerd`) that all CLI commands communicate with via a socket — a compromise of this daemon grants full root access to the host. Podman is daemonless: each container is a direct child process of the user's session, requiring no privileged background service. Podman's rootless mode runs the entire container lifecycle in a user namespace without ever needing root, reducing the attack surface significantly. Security-conscious teams prefer Podman because it eliminates the root daemon attack vector, supports the same OCI image/runtime standards, and provides better alignment with the principle of least privilege.

---

**17. Compare the day-2 operational overhead (cluster upgrades, scaling nodes, backups) of self-managed Kubernetes versus a managed Kubernetes service.**

Self-managed Kubernetes requires manual control-plane upgrades (etcd backup, version skew management, component-by-component rollout), node OS patching, certificate rotation, and etcd disaster recovery — easily a part-time SRE role. Managed services (EKS, GKE, AKS) automate control-plane upgrades, handle etcd backups transparently, offer one-click node pool scaling and auto-upgrades, and provide integrated monitoring. The operational overhead difference is enormous: self-managed demands deep Kubernetes expertise and ongoing toil, while managed services reduce day-2 to node pool sizing and application-level concerns. Self-managed only makes sense when regulatory/air-gap requirements prevent using cloud-managed offerings.

---

**18. A media-streaming company expects spiky, unpredictable global traffic. Recommend a containerized solution. Elaborate your choice.**

I recommend **managed Kubernetes (GKE/EKS) with Horizontal Pod Autoscaler and Cluster Autoscaler, deployed multi-region behind a global load balancer (e.g., AWS CloudFront + ALB, or GCP Global LB)**. HPA scales pods based on request rate or bandwidth metrics; Cluster Autoscaler provisions nodes within seconds to handle spikes. Multi-region deployment ensures low latency globally and fault tolerance. Kubernetes' declarative scaling, combined with cloud-native CDN caching for static content, handles unpredictable traffic patterns efficiently while scaling to zero cost during lulls with solutions like KEDA.

---

**19. Defend whether a company that has outgrown Docker Swarm (or a single Compose host) should migrate to Kubernetes — weigh the migration effort against the long-term benefits.**

Yes, migrating to Kubernetes is justified when Swarm's limitations become blocking — Swarm lacks robust auto-scaling, has a shrinking ecosystem/community, limited RBAC, no native support for StatefulSets or CRDs, and no major cloud provider actively develops it. The migration effort involves rewriting Compose/Stack files as Kubernetes manifests, adopting a CNI plugin, and training the team — significant but one-time costs. Long-term benefits include a vast ecosystem (Helm, operators, service meshes), multi-cloud portability, active development, and superior scaling. The industry has standardized on Kubernetes; staying on Swarm means increasing technical debt with a deprecated platform.

---

**20. Would you choose Kubernetes as a solution for a microservices architecture, focus on its orchestration, resilience and ecosystem. Elaborate your standing with arguments.**

Yes, Kubernetes is ideal for microservices. **Orchestration:** Deployments, Services, and Ingress natively model independent microservices with independent scaling and release cycles. **Resilience:** Self-healing (restart, reschedule), health probes, circuit-breaking (via service mesh), and PodDisruptionBudgets ensure high availability. **Ecosystem:** Service meshes (Istio/Linkerd) add observability, traffic management, and mTLS; Helm charts package services; operators manage stateful dependencies. The declarative model means each team owns their service's manifests independently. Kubernetes was essentially designed for microservices and its entire architecture reflects that paradigm.

---

**21. Would you choose Kubernetes for enterprise-grade applications, focus on its scalability, community support and feature richness. Elaborate your standing with arguments.**

Yes, Kubernetes excels for enterprise-grade applications. **Scalability:** HPA, VPA, and Cluster Autoscaler handle workloads from 10 to 10,000+ pods; the control plane supports 5,000+ nodes. **Community support:** Backed by CNCF with contributions from Google, Microsoft, Red Hat, and thousands of developers — ensuring rapid bug fixes, security patches, and feature evolution. **Feature richness:** RBAC, NetworkPolicies, PodSecurityAdmission, CRDs, CSI, and Gateway API provide enterprise controls out-of-the-box. The ecosystem (Prometheus, Fluentd, OPA, cert-manager) covers every enterprise requirement. No other orchestrator matches this combination of scale, governance, and extensibility.

---

**22. Would you choose OpenShift as a solution for a microservices architecture, focus on its orchestration, resilience and ecosystem. Elaborate your standing with arguments.**

Yes, OpenShift is excellent for microservices, especially in enterprise settings. It provides all Kubernetes orchestration capabilities plus built-in CI/CD pipelines (Tekton), integrated service mesh (OpenShift Service Mesh/Istio), and a developer console that visualizes microservice topology. Resilience is enhanced by default SCCs preventing risky container configurations and integrated monitoring/alerting (pre-configured Prometheus). The OperatorHub ecosystem provides production-ready operators for databases, message queues, and middleware that microservices depend on. OpenShift is particularly suited when the organization wants an opinionated, pre-integrated microservices platform without assembling individual CNCF tools.

---

**23. Would you choose OpenShift for enterprise-grade applications, focus on its scalability, community support and feature richness. Elaborate your standing with arguments.**

Yes, OpenShift is a strong choice for enterprise-grade applications. **Scalability:** Built on Kubernetes' proven scalability with added multi-cluster management (Advanced Cluster Management/ACM) for global deployments. **Community support:** Backed by Red Hat's enterprise SLA, regular LTS releases, security patches within hours of CVEs, plus the upstream Kubernetes community — enterprises get both commercial and open-source support. **Feature richness:** Includes everything Kubernetes offers plus integrated GitOps (ArgoCD), compliance scanning (Compliance Operator), FIPS-validated cryptography, and automated cluster upgrades. For regulated enterprises needing a certified, supported, and hardened platform with accountability, OpenShift justifies its cost through reduced operational risk and compliance effort.

---
