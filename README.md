# K8 Local Debug Setup
How to Debug and Develop services deployed in Kubernetes environment with the speed of local dev environment 

Congratulations! if you have reached this point in K8 journey.  While K8 is fantastic for running your applications, as a software developer I know they require constant enhancement and support.  

## Here I am going to talk about how you can make your life easier when debugging, developing services for K8.
My service is a sample hello world application in nodejs.

        const http = require('http');

        const hostname = '127.0.0.1';
        const port = 10000;

        const server = http.createServer((req, res) => {
          res.statusCode = 200;
          res.setHeader('Content-Type', 'text/plain');
          res.end('Hello World');
        });

        server.listen(port, hostname, () => {
          console.log(`Server running at http://${hostname}:${port}/`);
        });


## Assumptions:
You are running VS Code, best IDE out there (100% biased opinion), and you have a lunch configuration that you can use to debug hello world application.
You can access K8 application from your dev machine, and you can access your dev machine from a K8 pod/container.
You are familiar with ingress controllers and ingress rules but it not really required.

k8 flow is like 
k8 ingress controller -> ingress rule -> service -> Statful Set / Deployment -> pod -> container

# The idea is very simple.  You all know nginx proxy.  I am going to alter the flow that looks below:

k8 ingress controller -> ingress rule -> nginx proxy service -> Stateful Set / Deployment -> pod -> nginx proxy container -> your dev box / port -> VS code debug configuration running locally on your dev box.

### Step 1. Assuming you have a Virtual Host (alice.example.local) that you will use that resolves to your k8 cluster, create an ingress rule for your nginx proxy service:

          # Source: nginx-proxy/ingress.yaml
          apiVersion: networking.k8s.io/v1
          kind: Ingress
          metadata:
            name: nginx-ingress-serviceToDebug
            namespace: default
            annotations:
              nginx.ingress.kubernetes.io/rewrite-target: /$1
          spec:
            ingressClassName: nginx-serviceToDebug
            rules:
              - host: alice.example.local
                http:
                  paths:
                    - path: "/"
                      pathType: Prefix
                      backend:
                        service:
                          name: serviceToDebug
                          port:
                            number: 10000
                  
  
### Step 2:  Define the k8 nginx proxy service

          ---
          # Source: nginx-proxy/service.yaml
          apiVersion: v1
          kind: Service
          metadata:
            name: serviceToDebug
            labels:
              app.kubernetes.io/name: serviceToDebug
              app.kubernetes.io/instance: serviceToDebug
          spec:
            externalIPs:
            ports:
            - protocol: TCP
              port: 10000
              targetPort: 10000
            selector:
              app: serviceToDebug-pod
            sessionAffinity: None
            type: ClusterIP
            
 ### Step 3: Create a Deployment/stateful set that deploys the nginx proxy container
 
           ---
          # Source: nginx-proxy/configMap.yaml
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: confnginx
          data:
            nginx.conf: |
              user  nginx;
              worker_processes  1;
              error_log  /var/log/nginx/error.log warn;
              pid        /var/run/nginx.pid;
              events {
                  worker_connections  1024;
              }
              http {
                  include       /etc/nginx/mime.types;
                  default_type  application/octet-stream;
                  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                                      '$status $body_bytes_sent "$http_referer" '
                                      '"$http_user_agent" "$http_x_forwarded_for"';
                  access_log  /var/log/nginx/access.log  main;
                  sendfile        on;
                  keepalive_timeout  65;
                  server {
                      listen 10000;
                      server_name ~^(?<subdomain>.*?)\.;
                      resolver kube-dns.kube-system.svc.cluster.local valid=5s;
                      location /healthz {
                          return 200;
                      }
                      location / {
                          proxy_set_header Upgrade $http_upgrade;
                          proxy_set_header Connection "Upgrade";
                          proxy_pass http://alice-dev-local.example.local:11000;
                          proxy_set_header Host $host;
                          proxy_http_version 1.1;
                      }
                  }
              }
            ---
            # Source: nginx-proxy/statefulset.yaml
            apiVersion: apps/v1
            kind: StatefulSet
            metadata:
              name: serviceToDebug
            spec:
              replicas: 1
              selector:
                matchLabels:
                  app: serviceToDebug-pod
              serviceName: serviceToDebug
              template:
                metadata:
                  labels:
                    app: serviceToDebug-pod
                spec:
                  containers:
                  - image: nginx:alpine
                    imagePullPolicy: IfNotPresent
                    name: serviceToDebug-docker-container
                    env: []
                    volumeMounts:
                    - name: nginx-config
                      mountPath: /etc/nginx/nginx.conf
                      subPath: nginx.conf
                  volumes:
                        - name: nginx-config
                          configMap:
                            name: confnginx
                            
 ### Step 4: Deploy K8 using kubectl apply -f ...
 
 ### Step 5: On your local box (alice-dev-local.example.local) create the VS Code debug configuration:
 
             {
                "version": "0.2.0",
                "configurations": [
                    {
                        "type": "node",
                        "request": "launch",
                        "name": "Launch Hello World",
                        "skipFiles": [
                            "<node_internals>/**"
                        ],
                        "program": "${workspaceFolder}/helloworld.js"
                    }
                ]
            }

### Step 6: Modify local helloworld application to listen on port 11000 (optional, you may run it on same k8 port if you like

        const http = require('http');

        const hostname = '127.0.0.1';
        const port = 11000;

        const server = http.createServer((req, res) => {
          res.statusCode = 200;
          res.setHeader('Content-Type', 'text/plain');
          res.end('Hello World');
        });

        server.listen(port, hostname, () => {
          console.log(`Server running at http://${hostname}:${port}/`);
        });
        
### Step 7: Use postman and fire a request to http://alice.example.local:10000/
 
 You are now ready to debug a k8 application locally.  You can set break points, modify code and relaunch the local application.
 
## Extra

If your application consists of multiple micro-services and let's say the hello world needs to talk to one such service, you have to make sure that they are accessible from outside at cluster level.  You can consume the service in you hello world by reading environment variables (process.env.MY_GREAT_SERVICE_HOST, process.env.MY_GREAT_SERVICE_PORT).  And you can define the env variable in launch config
 
              {
                "version": "0.2.0",
                "configurations": [
                    {
                        "type": "node",
                        "request": "launch",
                        "name": "Launch Hello World",
                        "skipFiles": [
                            "<node_internals>/**"
                        ],
                        "program": "${workspaceFolder}/helloworld.js",
                        "env: [
                          "MY_GREAT_SERVICE_HOST": "http://alice.example.local",
                          "MY_GREAT_SERVICE_PORT": "10001",
                        ]
                    }
                ]
            }
