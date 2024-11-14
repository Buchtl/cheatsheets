# Example App
Here's a simple YAML file to deploy a single container in a Pod on Kubernetes. This example uses an `httpd` container image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
  labels:
    app: httpd
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
      ports:
        - containerPort: 80
```

### Explanation:

- `apiVersion: v1`: Specifies the Kubernetes API version.
- `kind: Pod`: Defines the resource type as a Pod.
- `metadata`: Provides metadata for the Pod, including its `name` and `labels`.
- `spec`: Specifies the Pod's configuration.
  - `containers`: Defines the list of containers for this Pod.
    - `name`: The name of the container within the Pod.
    - `image`: The container image to use; here, we use the latest `httpd` image.
    - `ports`: Specifies the ports exposed by the container; in this case, port 80 for HTTP.

You can apply this YAML file with:

```bash
kubectl apply -f pod.yaml
```

This will create a Pod with a single container running `httpd`.

# Expose Port

To expose the container's port 80 to your localhost on port 8080, you can use `kubectl port-forward`. This command allows you to forward a local port to a port on the container within the Pod.

Here's how you can do it:

```bash
kubectl port-forward pod/httpd-pod 8080:80
```

### Explanation

- `pod/httpd-pod`: Specifies the Pod by name (`httpd-pod` in this case).
- `8080:80`: Maps your local port 8080 to the Pod's container port 80.

Once you run this command, you should be able to access the application on `http://localhost:8080`.

This port-forwarding command will stay active until you stop it (e.g., by pressing `Ctrl+C`). Note that this approach is typically used for local testing and debugging. For a production setup, you may want to create a Kubernetes `Service` to expose the Pod's port.

Yes, you can also use `kubectl expose` to expose the Pod’s port. This command creates a Kubernetes `Service` to expose the Pod, which is more suitable for long-term access compared to `kubectl port-forward`.

Here’s how you can do it:

```bash
kubectl expose pod httpd-pod --type=NodePort --port=80 --target-port=80 --name=httpd-service
```

### Explanation

- `pod httpd-pod`: Specifies the Pod to expose (in this case, `httpd-pod`).
- `--type=NodePort`: Sets the Service type to `NodePort`, which exposes the Service on a port on each node.
- `--port=80`: The port that the Service will expose.
- `--target-port=80`: The port on the container that the Service will forward traffic to.
- `--name=httpd-service`: Names the Service as `httpd-service`.

### Accessing the Pod

After running this command, you can get the NodePort assigned to the Service by running:

```bash
kubectl get service httpd-service
```

You’ll see an output like this:

```
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
httpd-service   NodePort   10.96.222.197    <none>        80:<NodePort>    5s
```

The `NodePort` value is typically in the range 30000–32767. You can access your application on `http://<NodeIP>:<NodePort>`, where `<NodeIP>` is the IP address of the Kubernetes node.

For local clusters (like Minikube), you can get the Node IP with:

```bash
minikube ip
```

If you want it accessible specifically on `localhost:8080`, you can use a `Service` of type `ClusterIP` combined with port-forwarding:

```bash
kubectl expose pod httpd-pod --type=ClusterIP --port=80 --target-port=80 --name=httpd-service
kubectl port-forward service/httpd-service 8080:80
``` 

This way, you get a persistent `Service` and can forward it to `localhost:8080`.

# External IP

The `EXTERNAL-IP` shows as `<none>` because the Service type `NodePort` does not allocate an external IP address. Here’s a quick breakdown of Service types and when an external IP might appear:

1. **ClusterIP** (default): Only accessible within the cluster, with no external IP.
2. **NodePort**: Exposes the Service on each node’s IP at a high port (30000–32767) but does not assign an external IP. You access the Service using `<NodeIP>:<NodePort>`, but the `EXTERNAL-IP` field is `<none>`.
3. **LoadBalancer**: Automatically provisions an external IP by creating a cloud load balancer, provided you're on a cloud provider that supports this (like AWS, GCP, or Azure). For LoadBalancer, an external IP should appear in that field once the load balancer is provisioned.

### Options to Access the Service Externally

1. **Using `minikube`** (for local testing):
   - If you are using Minikube, you can access a `NodePort` Service by running:
     ```bash
     minikube service httpd-service
     ```
   - This command opens the Service in your browser and forwards the correct IP and NodePort.

2. **Using a LoadBalancer Service** (on a supported cloud provider):
   - If you're on a cloud provider, you can change the Service type to `LoadBalancer`, which would provision an external IP.
   - To do this, run:
     ```bash
     kubectl expose pod httpd-pod --type=LoadBalancer --port=80 --target-port=80 --name=httpd-service
     ```
   - After a few seconds, you should see an external IP assigned to the Service.

For local setups without Minikube, `NodePort` with direct access to `NodeIP:NodePort` or `port-forward` are the primary options.

# Ingress
Here’s a sample Kubernetes YAML configuration to expose the `httpd` app using an NGINX Ingress. This configuration assumes that you have an NGINX Ingress controller installed in your cluster.

We'll create:
1. A `Deployment` to manage the Pod.
2. A `Service` to expose the Pod internally.
3. An `Ingress` resource to route external traffic to the Service.

### Full Configuration

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
  labels:
    app: httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
        - name: httpd
          image: httpd:latest
          ports:
            - containerPort: 80

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: httpd-service
spec:
  selector:
    app: httpd
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpd-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: httpd.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: httpd-service
                port:
                  number: 80
```

### Explanation

- **Deployment**:
  - Manages a Pod with the `httpd` container.
  - Ensures one replica is always running.
  
- **Service**:
  - Exposes the `httpd` container internally on port 80.
  - Type `ClusterIP` means it is only accessible within the cluster.

- **Ingress**:
  - Routes external traffic from the specified hostname (`httpd.local`) to the `httpd-service`.
  - Uses the `nginx.ingress.kubernetes.io/rewrite-target: /` annotation to rewrite requests to the root path.
  - The `pathType: Prefix` and `path: /` settings route all paths to the `httpd-service`.

### Accessing the Application

1. **DNS or Hosts File**: Add `httpd.local` to your `/etc/hosts` file with your cluster’s IP if you’re testing locally:
   ```plaintext
   <Cluster IP> httpd.local
   ```

2. **Check Ingress**: Run `kubectl get ingress httpd-ingress` to verify the ingress is configured correctly and see the external IP.

This setup should allow you to access your `httpd` app by visiting `http://httpd.local` once the Ingress is set up and the DNS or hosts file is configured.
