
Let's say that the goal it's to generate the following manifest

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: kuard-test
  # labels:
    # pod-security.kubernetes.io/enforce: restricted
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
  namespace: kuard-test
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
        - image: gcr.io/kuar-demo/kuard-amd64:1
          imagePullPolicy: Always
          name: kuard
          ports:
            - containerPort: 8080
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
            allowPrivilegeEscalation: false
            seccompProfile:
              type: RuntimeDefault
            capabilities:
              drop:
                - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: kuard
  namespace: kuard-test

spec:
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  selector:
    app: kuard
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kuard
  namespace: kuard-test
  annotations:
    kubernetes.io/ingress.class: "nginx"
    #cert-manager.io/issuer: "letsencrypt-staging"

spec:
  tls:
  - hosts:
    - kuard.rubenlaguna.com
    secretName: quickstart-example-tls
  rules:
  - host: kuard.rubenlaguna.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kuard
            port:
              number: 80
```


First we need a `.kapitan` file that sets up some options 
* `.compile.fetch` is set to `true` so that the dependencies are downloaded. The dependencies are setup in the `.parameters.kapitan.dependencies`
* `.compile.prune` will remove keys with `null` values , the kadet's kubernetes generator for example will generate
```
kind: Namespace
metadata:
    name: xxxx
    namespace: null
```
by setting `prune` we ensure that `namespace: null` is removed from the output



You need a target in this case `demo` which is defined at `inventory/targets/demo.yml`. 

That target defines some stuff
* `.parameters.kapitan.var.target`. This is mandatory to set
* `.parameters.kapitan.dependencies`
  * will dowload the kadet's kubernetes generator when you use `./kapitan compile --fetch`

* `.parameters.kapitan.compile`, runs the kadet kubernetes generators that where downloaded (as a dependencie) in `system/generators/kubernetes`

* `.parameters.generators`, sets some options for the generator that will be used by the generator as defaults, 
