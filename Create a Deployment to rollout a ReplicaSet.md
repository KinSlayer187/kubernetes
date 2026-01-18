The following is an example of a Deployment. It creates a ReplicaSet to bring up three nginx Pods:

```controllers/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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

In this example:
- A Deployment named ```nginx-deployment``` is created, indicated by the ```.metadata.name``` field. This name will become the basis for the ReplicaSets and Pods which are created later. See [[Writing a Deployment Spec]] for more details.
- The Deployment create a ReplicaSet that creates three replicated Pods, indicated by ```.spec.replicas``` field.
- 