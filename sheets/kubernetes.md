Here's an example of an RKE Kubernetes deployment YAML that sets up a simple app with two pods: one running an `httpd` container for the front-end and the other running a Java JDK container for the back-end. These two components will communicate within the Kubernetes cluster:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-namespace

---
# Deployment for the HTTPD (Frontend) Container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
  namespace: app-namespace
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
          env:
            - name: BACKEND_URL
              value: "http://backend-service:8080" # URL of backend service in cluster

---
# Service for HTTPD to expose it within the cluster and externally
apiVersion: v1
kind: Service
metadata:
  name: httpd-service
  namespace: app-namespace
spec:
  selector:
    app: httpd
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer # Allows external access

---
# Deployment for the Backend (Java JDK) Container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: app-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: openjdk:latest
          ports:
            - containerPort: 8080
          command: ["java", "-jar", "/path/to/your-backend-app.jar"] # Customize this

---
# Service for Backend to expose it within the cluster for frontend to communicate
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: app-namespace
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP # Internal access only for the front-end
```

### Explanation

- **Namespace**: Creates a separate namespace for the app's components.
- **HTTPD Deployment and Service**:
  - The `httpd-deployment` runs the HTTPD server, and the service `httpd-service` exposes it externally.
- **Backend Deployment and Service**:
  - The `backend-deployment` runs a Java JDK container that serves the backend application.
  - The `backend-service` is a `ClusterIP` service, exposing it only within the cluster. This allows the HTTPD container to reach it through the `BACKEND_URL` environment variable.

This deployment allows for scaling and networking flexibility, making it easier to manage the frontend and backend separately.


# Ingress
Adding an `Ingress` resource will allow external access to the frontend through an HTTP/HTTPS endpoint. Here’s how to include it in the YAML:

1. The `Ingress` will route external traffic to the `httpd-service` based on specified paths or hostnames.
2. Ensure that an Ingress controller (like NGINX or Traefik) is installed in your Kubernetes cluster, as it is required to manage the Ingress resources.

Here's the updated YAML with the Ingress configuration:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-namespace

---
# HTTPD Deployment (Frontend)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
  namespace: app-namespace
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
          env:
            - name: BACKEND_URL
              value: "http://backend-service:8080" # Internal backend service URL

---
# HTTPD Service
apiVersion: v1
kind: Service
metadata:
  name: httpd-service
  namespace: app-namespace
spec:
  selector:
    app: httpd
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP # Internal service for Ingress

---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: app-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: openjdk:latest
          ports:
            - containerPort: 8080
          command: ["java", "-jar", "/path/to/your-backend-app.jar"] # Customize path

---
# Backend Service
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: app-namespace
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP

---
# Ingress for HTTPD (Frontend)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpd-ingress
  namespace: app-namespace
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: / # Adjust based on Ingress controller type
spec:
  rules:
    - host: your-app.example.com # Replace with your domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: httpd-service
                port:
                  number: 80
  tls:
    - hosts:
        - your-app.example.com
      secretName: your-tls-secret # Name of TLS secret for HTTPS
```

### Explanation of the Ingress Configuration

- **Rules**:
  - Specifies the hostname (`your-app.example.com`) for accessing the frontend. Replace this with your domain.
  - Routes requests on `/` to the `httpd-service`.
- **TLS**:
  - Configures HTTPS access. Make sure to create a TLS secret (`your-tls-secret`) in the same namespace for HTTPS support.

This `Ingress` setup will allow external access to the frontend app over HTTP and HTTPS. The `httpd-service` is configured as a `ClusterIP` because only the `Ingress` will route traffic to it from outside the cluster.
