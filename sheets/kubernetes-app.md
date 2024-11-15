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

# OpenShift
To adapt the previous `httpd` deployment for OpenShift, there are a few key changes required:

1. OpenShift doesn’t allow running containers as the root user by default, so we'll add a `SecurityContext` to ensure the container runs as a non-root user.
2. OpenShift’s `Route` resource will replace the `Ingress` resource for exposing the application externally.

Here’s the updated configuration:

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
          securityContext:
            runAsUser: 1001       # Ensures the container runs as non-root
            allowPrivilegeEscalation: false

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
# Route
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: httpd-route
spec:
  to:
    kind: Service
    name: httpd-service
  port:
    targetPort: 80
  tls:
    termination: edge
  host: httpd-app.example.com  # Replace with your domain or use OpenShift's default subdomain
```

### Explanation of Changes

- **Deployment**:
  - Added `securityContext` to ensure that the container runs as a non-root user (`runAsUser: 1001`).
  - `allowPrivilegeEscalation: false` to adhere to OpenShift’s security constraints.

- **Route**:
  - Replaces the `Ingress` resource in OpenShift.
  - Exposes the `httpd-service` to the outside world.
  - The `host` field specifies the external hostname for the route. OpenShift typically auto-generates a domain if not specified, but you can set it to something like `httpd-app.example.com` if you have a custom domain.

### Accessing the Application

1. **Default Domain**: OpenShift automatically assigns a subdomain if the `host` field is omitted.
2. **Custom Domain**: If you specify a domain in `host`, make sure it points to your OpenShift cluster’s IP.

This configuration will deploy the `httpd` app with an OpenShift-friendly security context and expose it via a Route.

# Minikube Ingress Problems
The issue likely arises because Minikube is attempting to pull images for the NGINX Ingress controller from external container registries (e.g., Docker Hub), which is not possible in your offline environment. To resolve this, you need to ensure that the required images are available locally or within a private registry accessible to your Minikube environment.

Here’s a step-by-step guide to set up NGINX Ingress in an offline Minikube environment:

---

### **1. Find the Required Images**

Check which images the Minikube Ingress addon uses. The typical images are:

- **Controller**: `k8s.gcr.io/ingress-nginx/controller:<tag>`
- **Kube-webhook-certgen**: `k8s.gcr.io/ingress-nginx/kube-webhook-certgen:<tag>`

To verify the specific tags, check the documentation or look at the YAML used by Minikube for the ingress addon:
```bash
minikube addons list --images | grep ingress
```

---

### **2. Preload the Images**

If you're offline, you need to preload these images. Follow these steps:

1. **Download the Images Elsewhere**: Use a machine with internet access to pull the required images.
   ```bash
   docker pull k8s.gcr.io/ingress-nginx/controller:<tag>
   docker pull k8s.gcr.io/ingress-nginx/kube-webhook-certgen:<tag>
   ```

2. **Save the Images**:
   ```bash
   docker save -o controller.tar k8s.gcr.io/ingress-nginx/controller:<tag>
   docker save -o certgen.tar k8s.gcr.io/ingress-nginx/kube-webhook-certgen:<tag>
   ```

3. **Transfer the Images to the Offline Environment**: Copy the `.tar` files to the machine running Minikube.

4. **Load the Images into Minikube**:
   ```bash
   minikube image load controller.tar
   minikube image load certgen.tar
   ```

---

### **3. Enable the Ingress Addon**

Now that the required images are preloaded, enable the addon:
```bash
minikube addons enable ingress
```

Minikube should now create the Ingress controller Pods without needing to pull images from external sources. Verify the Pods:
```bash
kubectl get pods -n kube-system
```

---

### **4. (Optional) Deploy Ingress Manually**

If the `minikube addons enable ingress` method still doesn't work, you can deploy the NGINX Ingress controller manually:

1. **Download the Ingress YAML**:
   - Download the official Ingress controller YAML from a machine with internet access:
     ```bash
     curl -o nginx-ingress.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
     ```
   - Transfer this file to your offline environment.

2. **Edit the YAML to Use Local Images**:
   - Replace the image names with the local image references in the `nginx-ingress.yaml` file.
     Example:
     ```yaml
     image: k8s.gcr.io/ingress-nginx/controller:<tag>
     ```
     becomes:
     ```yaml
     image: localhost/k8s.gcr.io/ingress-nginx/controller:<tag>
     ```

3. **Apply the YAML**:
   ```bash
   kubectl apply -f nginx-ingress.yaml
   ```

---

### **5. Verify the Setup**

Check the status of the Ingress controller Pods:
```bash
kubectl get pods -n ingress-nginx
```

Once they are running, test the ingress configuration as described earlier.

---

### **Key Notes for Offline Setup**

- Always preload images for any additional components you plan to use.
- Consider setting up a private registry to simplify image management in your offline environment.
- Minikube’s `--cache-images` flag can help cache required images locally when setting up Minikube in a connected environment.

Let me know if you need further assistance!
