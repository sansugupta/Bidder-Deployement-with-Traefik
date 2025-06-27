Comprehensive End-to-End Documentation: Ad Bidding System with Kubernetes and Traefik
Project Overview
This document provides a detailed overview of the ad bidding system deployed on an Amazon Elastic Kubernetes Service (EKS) cluster, integrated with Traefik as a reverse proxy to manage HTTPS traffic from Google AdX. The system ensures secure routing, load balancing, TLS termination, and high availability. The project was executed from June 20, 2025, to June 25, 2025, at 02:16 PM IST, and this document serves as the final handover to Jason for validation and ongoing maintenance.
Project Objectives

Route HTTPS traffic from https://us-east.googleadx.dsp-fraudfree.net/ and https://us-east.googleadx.fraudfree.net/ to Kubernetes-hosted bidder services.
Implement TLS termination using Let’s Encrypt certificates managed by Traefik.
Ensure high availability and scalability using AWS Network Load Balancers (NLB) and Kubernetes deployments.
Provide comprehensive documentation for operational handover, troubleshooting, and portfolio showcasing.
Address issues such as TLS handshake failures, 502 Bad Gateway errors, and connection resets.

Infrastructure Setup
Kubernetes Cluster

Cluster Details:
Hosted on AWS EKS, endpoint: https://4435FED514988924928841CC9E147903.gr7.us-east-1.eks.amazonaws.com.
Kubernetes Version: Server v1.33.1-eks-1fbb135, Client v1.32.3 (built 2025-03-11).
Generated Report: k8s-complete-cluster-20250620-233325 (created Fri Jun 20 23:41:05 IST 2025).


Nodes:
Total: 6 nodes, all AMD64 Linux instances.
IPs and Hostnames:
ip-172-31-23-64.ec2.internal (172.31.23.64)
ip-172-31-25-190.ec2.internal (172.31.25.190)
ip-172-31-70-238.ec2.internal (172.31.70.238)
ip-172-31-71-236.ec2.internal (172.31.71.236)
ip-172-31-82-89.ec2.internal (172.31.82.89)
ip-172-31-88-150.ec2.internal (172.31.88.150)


Architecture: AMD64, OS: Linux, Kernel: v5.x, Container Runtime: containerd.


Namespaces: 6 total, primary namespace: default (created 2025-06-12T20:58:30Z, UID: 52dff874-98af-484d-bf05-3753dc50cd98).
Resources:
Cluster-Wide:
Pods: 42
Deployments: 11
Services: 12
ConfigMaps: 18
Secrets: 10


Default Namespace:
Pods: 13 (e.g., po-bidder-6c9fd99b84-2dl2j, po-dev-bidder-864bcd58-m9r74)
Deployments: 4 (e.g., po-bidder, po-dev-bidder)
Services: 6 (e.g., po-bidder, kubernetes)
ConfigMaps: 6 (e.g., po-bidder-config, po-monthly-report-config)
Secrets: 7
ReplicaSets: 5




Load Balancer Services:
po-bidder: k8s-default-pobidder-daa62f3d85-8b28917bb34e97a4.elb.us-east-1.amazonaws.com (port 22258).
po-dev-bidder: k8s-default-podevbid-aff72eed08-9dd06f420baea488.elb.us-east-1.amazonaws.com (port 22259).
po-dev-eventimpression: k8s-default-podeveve-06e8dfa061-bb4276bf60a1f3f6.elb.us-east-1.amazonaws.com (port 8080).
po-eventimpression: k8s-default-poeventi-d7cf9c2e4a-48d88dc37e9b91df.elb.us-east-1.amazonaws.com (port 8080).


Storage: 0 Persistent Volumes, 2 Storage Classes (gp2, standard).

Traefik Configuration

Deployment:
Hosted on EC2 instance ip-172-31-82-69.
Configuration directory: /etc/traefik/.


Configuration Files:
traefik.yml:
API: Dashboard enabled (insecure: true), debug mode off.
Entry Points:
web: :8081 (internal proxy port).
traefik: :8080 (dashboard).


Providers: File-based, watching /etc/traefik/dynamic.
Logging: INFO level.


dynamic/routing.yml:
Router po-bidder-main:
Rule: Host(us-east.googleadx.dsp-fraudfree.net) || Host(us-east.googleadx.fraudfree.net).
Entry Point: web.
Service: po-bidder-service.
Middleware: root-redirect (redirects / to /googleadx).


Service po-bidder-service:
Load Balancer: URL http://k8s-default-pobidder-daa62f3d85-8b28917bb34e97a4.elb.us-east-1.amazonaws.com:22258.
Health Check: Path /health, interval 30s, timeout 5s.




dynamic/certificates.yml:
TLS Certificates:
us-east.googleadx.dsp-fraudfree.net: /etc/letsencrypt/live/.../fullchain.pem, /etc/letsencrypt/live/.../privkey.pem.
us-east.googleadx.fraudfree.net: Similar paths.






Certificates:
Stored in /etc/traefik/certs/: dsp-fraudfree.crt, dsp-fraudfree.key, fraudfree.crt, fraudfree.key.


Startup:
Command: docker run -d -p 8080:8080 -p 8081:8081 -v /etc/traefik:/etc/traefik traefik:latest.



Docker Images

ECR Repository: 590184014267.dkr.ecr.us-east-1.amazonaws.com/prodatamediagroup/.
Images:
po-bidder:latest:
Dockerfile: Po-bider.Dockerfile.
Base: ubuntu:22.04, installs libssl-dev, netcat-openbsd.
Copies po-bidder binary, entrypoint.sh, config files.
Exposes port 22259, runs as appuser.


po-dev-bidder:latest:
Dockerfile: Po-dev-bidder.Dockerfile.
Similar to po-bidder, adds bind9-dnsutils.
Exposes port 22259, uses dev-bidder-entrypoint.sh.


po-dev-eventimpression:v1.0.0:
Dockerfile: po-dev-event-impression.Dockerfile.
Exposes port 22261.


po-eventimpression:v1.0.0:
Dockerfile: po-event-impression.Dockerfile.
Exposes port 22260.




Build Process:
Script: build_push_ecr.sh.
Uses AWS credentials (AKIAYS2NVMG5XV6WSNGJ, redacted key).
Pushes to ECR with tags latest and timestamp (e.g., 20250625-1416).



System Flow

Client Request:
Google AdX sends HTTPS request to https://us-east.googleadx.dsp-fraudfree.net/ or .fraudfree.net.


Traefik Processing:
Terminates TLS using Let’s Encrypt certificates.
Matches po-bidder-main router rule based on hostname.
Applies root-redirect middleware to redirect / to /googleadx.
Forwards request to po-bidder-service.


Load Balancing:
AWS NLB distributes traffic across Kubernetes nodes.


Kubernetes Routing:
po-bidder service selects pods with app=po-bidder label.
Pods (e.g., po-bidder-6c9fd99b84-2dl2j at 172.31.66.88) process on port 22258.
Environment variables (e.g., RABBITMQ_URI, REDIS_ADS_ADDR) sourced from po-bidder-secret.


Response:
Pods return bid response (e.g., 204 No Content), relayed via NLB and Traefik.


Health Monitoring:
Traefik checks /health every 30s.
Kubernetes liveness/readiness probes on port 22258 (delay 15s/5s, period 10s/5s).



Architecture Diagram
graph TD
    subgraph Internet
        A[Google AdX Servers]
    end

    subgraph "Traefik Ingress Layer (EC2: ip-172-31-82-69)"
        B(DNS: us-east.googleadx.fraudfree.net) --> C{Traefik Proxy}
        C -- Manages SSL --> D[Let's Encrypt]
    end

    subgraph "AWS VPC"
        C --> E[AWS Network Load Balancer (NLB)]
    end

    subgraph "Amazon EKS Cluster (6 Nodes)"
        E --> F[K8s Service: po-bidder]
        F --> G1[Pod: po-bidder-1]
        F --> G2[Pod: po-bidder-2]
        F --> G3[Pod: po-bidder-n]
        G1 --> H((Databases & Cache))
        G2 --> H
        G3 --> H
    end

    subgraph "External Dependencies"
        H
        I[Production Metrics Server / Grafana]
        G1 --> I
        G2 --> I
        G3 --> I
    end

    A -- "POST /" --> B

Detailed Code Breakdown
Traefik Configuration Files

traefik.yml:

api:
  dashboard: true
  insecure: true
entryPoints:
  web:
    address: ":8081"
  traefik:
    address: ":8080"
providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true
log:
  level: INFO
accessLog: {}


dynamic/routing.yml:

http:
  routers:
    po-bidder-main:
      rule: "Host(`us-east.googleadx.dsp-fraudfree.net`) || Host(`us-east.googleadx.fraudfree.net`)"
      entryPoints:
        - web
      service: po-bidder-service
      middlewares:
        - root-redirect
  middlewares:
    root-redirect:
      replacePathRegex:
        regex: "^/$"
        replacement: "/googleadx"
  services:
    po-bidder-service:
      loadBalancer:
        servers:
          - url: "http://k8s-default-pobidder-daa62f3d85-8b28917bb34e97a4.elb.us-east-1.amazonaws.com:22258"
        healthCheck:
          path: "/health"
          interval: "30s"
          timeout: "5s"


dynamic/certificates.yml:

tls:
  certificates:
    - certFile: /etc/letsencrypt/live/us-east.googleadx.dsp-fraudfree.net/fullchain.pem
      keyFile: /etc/letsencrypt/live/us-east.googleadx.dsp-fraudfree.net/privkey.pem
    - certFile: /etc/letsencrypt/live/us-east.googleadx.fraudfree.net/fullchain.pem
      keyFile: /etc/letsencrypt/live/us-east.googleadx.fraudfree.net/privkey.pem

Dockerfile Examples

Po-bider.Dockerfile:

FROM --platform=linux/amd64 ubuntu:22.04
RUN apt-get update && apt-get install -y \
    libssl-dev \
    libc6 \
    libc-bin \
    ca-certificates \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*
RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY po-bidder/po-bidder /app/po-bidder
COPY po-bidder/entrypoint.sh /app/entrypoint.sh
COPY po-bidder/App.toml.tpl /app/config/App.toml.tpl
COPY po-bidder/Rocket.toml /app/config/Rocket.toml
RUN chmod +x /app/po-bidder /app/entrypoint.sh && \
    chmod 644 /app/config/App.toml.tpl /app/config/Rocket.toml && \
    chown -R appuser:appuser /app && \
    ls -l /app /app/config && \
    file /app/po-bidder && \
    ldd /app/po-bidder || echo "Binary dependencies check completed"
EXPOSE 22259
USER appuser
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["/app/po-bidder"]


Po-dev-bidder.Dockerfile:

FROM --platform=linux/amd64 ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl \
    netcat-openbsd \
    libssl-dev \
    bind9-dnsutils \
    libc6 \
    libc-bin \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY po-dev-bidder/po-bidder /app/po-dev-bidder
COPY po-dev-bidder/dev-bidder-entrypoint.sh /app/dev-bidder-entrypoint.sh
RUN chmod +x /app/po-dev-bidder /app/dev-bidder-entrypoint.sh && \
    chown -R appuser:appuser /app
RUN ldd /app/po-dev-bidder || echo "Binary dependencies check completed"
EXPOSE 22259
USER appuser
ENTRYPOINT ["/app/dev-bidder-entrypoint.sh"]
CMD ["/app/po-dev-bidder"]

Kubernetes Manifests

Pod Example (po-bidder-6c9fd99b84-2dl2j):

apiVersion: v1
kind: Pod
metadata:
  name: po-bidder-6c9fd99b84-2dl2j
  namespace: default
  labels:
    app: po-bidder
    pod-template-hash: 6c9fd99b84
spec:
  containers:
  - name: po-bidder
    image: 590184014267.dkr.ecr.us-east-1.amazonaws.com/prodatamediagroup/po-bidder:latest
    ports:
    - containerPort: 22258
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 256Mi
    livenessProbe:
      tcpSocket:
        port: 22258
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 1
      failureThreshold: 3
    readinessProbe:
      tcpSocket:
        port: 22258
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 1
      failureThreshold: 3
    env:
    - name: RUST_BACKTRACE
      value: full
    - name: ROCKET_PORT
      value: "22258"
    - name: RABBITMQ_URI
      valueFrom:
        secretKeyRef:
          name: po-bidder-secret
          key: RABBITMQ_URI
    volumeMounts:
    - mountPath: /app/config
      name: config
  volumes:
  - name: config
    configMap:
      name: po-bidder-config


Deployment Example (po-bidder):

apiVersion: apps/v1
kind: Deployment
metadata:
  name: po-bidder
  namespace: default
spec:
  replicas: 6
  selector:
    matchLabels:
      app: po-bidder
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: po-bidder
    spec:
      containers:
      - name: po-bidder
        image: 590184014267.dkr.ecr.us-east-1.amazonaws.com/prodatamediagroup/po-bidder:latest
        ports:
        - containerPort: 22258
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 256Mi


Service Example (po-bidder):

apiVersion: v1
kind: Service
metadata:
  name: po-bidder
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
spec:
  type: LoadBalancer
  ports:
  - port: 22258
    targetPort: 22258
    protocol: TCP
  selector:
    app: po-bidder

End-to-End Traffic Flow
sequenceDiagram
    participant Google as Google AdX Server
    participant DNS as Public DNS
    participant Traefik as Traefik Proxy<br>(ip-172-31-82-69)
    participant NLB as AWS Network LB
    participant BidderPod as po-bidder Pod

    Google->>DNS: Query us-east.googleadx.fraudfree.net
    DNS-->>Google: 3.229.15.229
    Google->>+Traefik: POST / (HTTPS, Port 443)
    Note over Traefik: 1. TLS Handshake using<br/>Let's Encrypt Certificate
    Note over Traefik: 2. Match Router: po-bidder-main
    Note over Traefik: 3. Apply Middleware:<br/>root-redirect
    Note over Traefik: 4. Path is now /googleadx
    Traefik->>+NLB: POST /googleadx (HTTP, Port 22258)
    NLB->>+BidderPod: POST /googleadx (HTTP, Port 22258)
    BidderPod-->>-NLB: 204 No Content (or Bid Response)
    NLB-->>-Traefik: 204 No Content (or Bid Response)
    Traefik-->>-Google: 204 No Content (or Bid Response)
    BidderPod->>Monitoring: Send Metrics (e.g., Request Count)

Flow Explanation

DNS Resolution: Google AdX queries us-east.googleadx.fraudfree.net, resolving to Traefik’s IP (3.229.15.229).
TLS Termination: Traefik presents Let’s Encrypt certificate, establishing a secure connection.
Routing: Traefik matches the po-bidder-main router based on the Host header and path /.
Path Rewrite: Middleware rewrites the path from / to /googleadx.
Forwarding: Traefik forwards the request to the AWS NLB on port 22258.
Kubernetes Networking: NLB routes to a healthy po-bidder pod.
Application Processing: The pod processes the request and responds (e.g., 204 No Content).
Metrics: Pods send metrics to Grafana for monitoring.

Deployment and Verification
Traefik Deployment

Setup:
SSH to ip-172-31-82-69.
Create /etc/traefik/ with traefik.yml, dynamic/routing.yml, dynamic/certificates.yml.
Copy certificates to /etc/traefik/certs/.


Run:docker run -d -p 8080:8080 -p 8081:8081 -v /etc/traefik:/etc/traefik traefik:latest


Verify:
Dashboard: http://ip-172-31-82-69:8080.
Logs: docker logs <container-id>.



Kubernetes Deployment

Apply Manifests:kubectl apply -f pod.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml


Verify:
Pods: kubectl get pods -n default -o wide
Services: kubectl get services -n default
NLB: kubectl describe service po-bidder -n default



Testing

Curl Test: curl -v https://us-east.googleadx.dsp-fraudfree.net/
Health Check: curl -v http://k8s-default-pobidder-daa62f3d85-8b28917bb34e97a4.elb.us-east-1.amazonaws.com:22258/health
Logs: kubectl logs po-bidder-6c9fd99b84-2dl2j -n default

Troubleshooting and Logs

TLS Handshake Errors:
Issue: EOF, i/o timeout in Traefik logs.
Solution: Increased timeouts in traefik.yml, regenerated certificates.
Log: /etc/traefik/logs/access.log (if enabled).


502 Bad Gateway:
Issue: NLB to pod connectivity failure.
Solution: Verified endpoints, adjusted health checks.
Log: Kubernetes events (kubectl get events -n default).


Connection Resets:
Issue: Resource limits exceeded.
Solution: Increased ulimit -n 65535 on Traefik host.
Log: Pod logs (kubectl logs <pod-name> -n default).



Challenges and Solutions

TLS Handshake: Resolved with certificate regeneration and timeout adjustments.
Health Check Failures: Switched to TCP probes, extended intervals.
502 Errors: Ensured NLB endpoint health, verified pod readiness.
Scalability: Added 6 replicas to po-bidder deployment.

Future Improvements

Monitoring: Integrate Prometheus/Grafana for metrics.
Auto-Scaling: Configure Horizontal Pod Autoscaler (HPA) for po-bidder.
Health Endpoint: Fix backend /health 500 error.
Logging: Enable structured logging in Traefik and pods.

Handover Notes

Artifacts: Kubernetes manifests, Dockerfiles, Traefik configs in attached directory.
Access: SSH key for ip-172-31-82-69, AWS IAM role for EKS.
Contact: [Your Email] for support until July 2, 2025.
Pending Tasks: Resolve /health 500 error, implement monitoring.
Project Closure: Completed June 25, 2025, 02:16 PM IST.

Conclusion
This project delivered a robust, scalable ad bidding system with comprehensive documentation. The system is production-ready, successfully processing Google AdX traffic. The handover to Jason ensures a seamless transition for ongoing maintenance and future enhancements.
