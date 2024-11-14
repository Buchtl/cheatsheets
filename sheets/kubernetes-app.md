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
