Project Handover: Google AdX Bidder on EKS with Traefik
Date: June 22, 2025
To: Jason Korkin
From: Sanskar
Status: Complete & Fully Operational
1. Executive Summary
This document outlines the final architecture and configuration for the deployment of the po-bidder application suite onto a new Amazon EKS (Elastic Kubernetes Service) cluster. The infrastructure is fronted by a dedicated Traefik reverse proxy to handle ingress traffic, TLS termination, and path-based routing.
The primary objective—to receive live Google AdX bid request traffic, rewrite the request path, and forward it to the po-bidder application running in Kubernetes—has been successfully implemented, tested, and verified. The system is stable, healthy, and correctly processing live traffic from Google's servers.
Final Status: The infrastructure is production-ready. The po-bidder application is successfully receiving and processing requests. The appearance of metrics on the Grafana dashboard is now dependent on the application's internal business logic and its ability to connect to the production metrics collector.
2. System Architecture
The solution is composed of two main environments: the Ingress/Proxy Layer and the Compute/Application Layer.
2.1. High-Level Architecture Diagram

![image](https://github.com/user-attachments/assets/bdd04ddb-9264-42b6-b5f9-b543fe1f3c30)

2.2. Component Breakdown
Component	Technology	Location	Responsibility
Ingress Proxy	Traefik v3.0	EC2 Instance	Handles public traffic, terminates TLS (SSL), rewrites request paths, and forwards to the backend.
Compute Platform	Amazon EKS v1.33	Kubernetes Cluster	Hosts the application containers (po-bidder, etc.), manages their lifecycle, and provides internal networking.
Application	po-bidder	Docker Containers	The core bidder application that receives, processes, and responds to bid requests.
Internal Bridge	AWS NLB	AWS VPC	A high-performance load balancer that bridges traffic from the Traefik proxy into the EKS cluster.
Certificate Authority	Let's Encrypt	External Service	Automatically provides and renews SSL/TLS certificates for all public-facing domains.
3. End-to-End Traffic Flow for Google AdX
This section details the precise journey of a single bid request from Google to the application.
Requirement: An incoming request to https://us-east.googleadx.fraudfree.net/ must be processed by the po-bidder application on the path /googleadx.
3.1. Flowchart Diagram
Generated mermaid
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
    Note over Traefik: 2. Match Router: po-bidder-google-root
    Note over Traefik: 3. Apply Middleware:<br/>add-googleadx-prefix
    Note over Traefik: 4. Path is now /googleadx
    
    Traefik->>+NLB: POST /googleadx (HTTP, Port 22258)
    NLB->>+BidderPod: POST /googleadx (HTTP, Port 22258)
    
    BidderPod-->>-NLB: 204 No Content (or Bid Response)
    NLB-->>-Traefik: 204 No Content (or Bid Response)
    Traefik-->>-Google: 204 No Content (or Bid Response)
    
    BidderPod->>Jason's Monitoring: Send Metrics (e.g., Request Count, No-Bid Reason)
Use code with caution.
Mermaid
3.2. Detailed Flow Explanation
DNS Resolution: A Google server queries the A record for us-east.googleadx.fraudfree.net and receives the IP address of the Traefik server (3.229.15.229).
Initial Connection: Google initiates an HTTPS connection to the Traefik server on port 443.
TLS Termination: Traefik presents the valid Let's Encrypt certificate for the domain, and a secure connection is established.
Routing: Traefik inspects the request. The Host header matches us-east.googleadx.fraudfree.net and the Path is exactly /. This matches the high-priority po-bidder-google-root router.
Path Rewrite: The add-googleadx-prefix middleware, associated with this router, is executed. It modifies the request's path from / to /googleadx.
Forwarding to Backend: Traefik forwards the modified request (now POST /googleadx) to the po-bidder-service URL: http://k8s-default-pobidder-....elb.amazonaws.com:22258.
Kubernetes Networking: The AWS Network Load Balancer receives the request on port 22258 and forwards it to a healthy po-bidder pod, also on port 22258.
Application Processing: The po-bidder application receives the request on the correct /googleadx path. Its internal logic is triggered, it makes a decision (e.g., "no bid, missing site info"), and sends back a response (e.g., 204 No Content).
Metrics Reporting: After processing the request, the application is responsible for making a separate, outbound connection to the production metrics server to report its activity. This is what populates the Grafana dashboards.
4. Final Configuration Details
4.1. Traefik Static Configuration (traefik.yml)
This file sets up the fundamental listeners and providers. The configuration is stable and healthy.
Generated yaml
# Enable API and Dashboard, and Ping for healthchecks
api:
  dashboard: true
  insecure: true
ping: {}

# Define all network entrypoints
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
  ping:
    address: ":8082"

# Configure Logging and Providers
log:
  level: DEBUG
providers:
  file:
    directory: "/configs"
    watch: true

# Configure Let's Encrypt for automatic SSL certificates
certificatesResolvers:
  letsencrypt:
    acme:
      email: "jasper@prodata.media"
      storage: "/acme.json"
      httpChallenge:
        entryPoint: web
Use code with caution.
Yaml
4.2. Traefik Dynamic Configuration (dynamic.yml)
This file contains the core routing logic. It has been simplified to ensure clarity and correctness.
Generated yaml
http:
  routers:
    # Router for Google's live traffic. It has the HIGHEST priority.
    po-bidder-google-root:
      rule: "Host(`us-east.googleadx.fraudfree.net`) && Path(`/`)"
      priority: 200
      service: po-bidder-service
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
      middlewares:
        - add-googleadx-prefix

    # A separate router for your health checks.
    po-bidder-health:
      rule: "Host(`us-east.googleadx.fraudfree.net`, `us-east.googleadx.dsp-fraudfree.net`) && Path(`/health`)"
      priority: 199
      service: po-bidder-service
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt

  services:
    po-bidder-service:
      loadBalancer:
        servers:
          - url: "http://k8s-default-pobidder-daa62f3d85-8b28917bb34e97a4.elb.us-east-1.amazonaws.com:22258"
        serversTransport: bidder-transport

  middlewares:
    add-googleadx-prefix:
      addPrefix:
        prefix: "/googleadx"

# Top-level block for backend transport configurations
serversTransports:
  bidder-transport:
    forwardingTimeouts:
      responseHeaderTimeout: "2s"
Use code with caution.
Yaml
4.3. Kubernetes Deployment (po-bidder)
The Kubernetes deployment ensures the application is running, scalable, and resilient. The resource requests and limits have been set according to the latest requirements.
Replicas: 6
Container Image: 590184014267.dkr.ecr.us-east-1.amazonaws.com/prodatamediagroup/po-bidder:latest
CPU Request/Limit: 500m / 1000m
Memory Request/Limit: 512Mi / 1Gi
Configuration: All necessary environment variables for Redis, RabbitMQ, and MySQL are securely injected from Kubernetes Secrets and ConfigMaps.
5. Final Verification & Conclusion
The system is live and processing traffic. Log analysis has confirmed that Google AdX requests are successfully traversing the entire infrastructure stack and are being processed by the po-bidder application's correct route handler.
