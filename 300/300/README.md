# 100 - Azure Application Gateway

This is a comprehensive reverse engineering breakdown of Azure Application Gateway following the exact same pattern. 

## What’s Inside

Following the same 8-layer structure:

**Layer 1: Portal View** - What you see vs the full reverse proxy infrastructure  
**Layer 2: Components** - Listeners, backend pools, routing rules, URL path maps, WAF, SSL certificates, rewrite rules  
**Layer 3: How It Works** - Two-connection model, HTTP parsing, URL routing, cookie affinity, WAF processing  
**Layer 4: Technology** - NGINX reverse proxy, ModSecurity WAF, OpenSSL, connection multiplexing  
**Layer 5: TCP/IP Mapping** - Layer 7 operations, HTTP/HTTPS processing, TLS termination  
**Layer 6: Request Flow** - Complete HTTP journey with TLS handshake, WAF inspection, routing decisions  
**Layer 7: Physical Implementation** - Instance architecture, zone-redundant deployment, configuration sync  
**Layer 8: Practical Examples** - Simple web app with WAF, microservices routing, multi-site hosting, App Service integration

## Key Topics Covered

- **V1 vs V2** and **Standard vs WAF** SKU comparisons
- **Multi-site hosting** with SNI and multiple SSL certificates
- **Path-based routing** for microservices architectures
- **URL path maps** with complex routing rules
- **Cookie-based session affinity** (sticky sessions)
- **SSL/TLS termination** vs **End-to-end TLS**
- **WAF configuration** (OWASP CRS, custom rules, detection/prevention modes)
- **Backend HTTP settings** (connection draining, timeout, host headers)
- **Rewrite rules** for headers and URLs
- **Health probes** (TCP, HTTP, HTTPS with body matching)
- **Connection multiplexing** and pooling
- **Autoscaling** configuration
- **FQDN backends** for App Service/Functions

## Critical Differences vs Load Balancer

The document clearly shows how Application Gateway differs from Azure Load Balancer:

- **Layer 4 vs Layer 7**: LB forwards packets, AppGW terminates connections
- **Two-connection model**: Client ←→ AppGW ←→ Backend (separate TCP connections)
- **Full HTTP awareness**: Can route based on URL paths, host headers, HTTP methods
- **Content inspection**: WAF can inspect and block malicious requests
- **SSL termination**: Handles TLS/SSL encryption/decryption
- **Request transformation**: Can modify headers, rewrite URLs, inject cookies

## Architecture Patterns

Four detailed examples show real-world usage:

1. **Simple web app** with WAF protection
1. **Microservices** with path-based routing (/api/auth, /api/orders, etc.)
1. **Multi-site hosting** (multiple domains on one gateway)
1. **End-to-end TLS** with Azure App Service

This completes your trio of Azure networking deep dives! You now understand:

- **NSG** (Layer 3/4 firewall)
- **Load Balancer** (Layer 4 load balancing)
- **Application Gateway** (Layer 7 reverse proxy with WAF)

# Reverse Engineering Azure Application Gateway

## From Azure Portal to HTTP Processing - A Complete Breakdown

A comprehensive deconstruction of how Azure Application Gateway works, from what you click in the portal down to how HTTP requests are processed, routed, and secured at Layer 7.

-----

## Table of Contents

- [Layer 1: What You See (Azure Portal)](#layer-1-what-you-see-azure-portal)
- [Layer 2: Application Gateway Components & Structure](#layer-2-application-gateway-components--structure)
- [Layer 3: How Application Gateway Works](#layer-3-how-application-gateway-works)
- [Layer 4: Under the Hood - The Technology](#layer-4-under-the-hood---the-technology)
- [Layer 5: Mapping to TCP/IP Model](#layer-5-mapping-to-tcpip-model)
- [Layer 6: Request Processing Flow](#layer-6-request-processing-flow)
- [Layer 7: Physical Implementation](#layer-7-physical-implementation)
- [Layer 8: Practical Examples](#layer-8-practical-examples)

-----

## Layer 1: What You See (Azure Portal)

### The User Interface View

When you navigate to an Application Gateway in the Azure Portal, here’s what you see:

```
Azure Portal → Resource Groups → Your Application Gateway
├── Overview
│   ├── Essentials (Name, Resource Group, Location, SKU, Tier)
│   ├── Frontend IP configuration
│   ├── Backend pools
│   └── HTTP settings
├── Configuration
├── Backend pools
├── Backend health
├── HTTP settings
├── Frontend IP configurations
├── Listeners
├── Rules
├── Rewrites
├── Health probes
├── Web application firewall (WAF)
├── SSL certificates
├── Diagnostics settings
├── Insights (monitoring)
└── Properties
```

### Creating an Application Gateway: The Simple View

**Portal Steps**:

1. Click “Create a resource”
1. Search “Application Gateway”
1. **Basics Tab**:
- **Name**: `MyAppGateway`
- **Region**: `West Europe`
- **Tier**: `Standard V2` or `WAF V2`
- **Enable autoscaling**: Yes/No
- **Instance count**: 2 (if manual)
- **Availability zones**: Select zones
- **Virtual network**: Select or create
- **Subnet**: Dedicated subnet for App Gateway
1. **Frontends Tab**:
- **Frontend IP type**: Public, Private, or Both
- **Public IP**: Create or select
1. **Backends Tab**:
- Add backend pool(s)
- Add backend targets (VMs, IPs, FQDN, App Service)
1. **Configuration Tab**:
- Add routing rule
- Configure listener (HTTP/HTTPS)
- Configure backend settings
- Configure health probe
1. **Tags** and **Review + Create**

**What Just Happened?**

- Azure deployed virtual appliances in dedicated subnet
- Created Layer 7 load balancer with HTTP awareness
- Configured SSL termination capability
- Set up URL routing and path-based routing
- Enabled Web Application Firewall (if WAF tier)
- Configured health monitoring
- Enabled autoscaling (V2 SKU)

### The Abstraction

**What Azure Hides From You**:

```
Simple Portal View:
┌──────────────────────────────────────┐
│  Application Gateway                  │
│  - Name: MyAppGateway                │
│  - Tier: WAF V2                      │
│  - Frontend IP: 20.x.x.x             │
│  - Backend: 3 web servers            │
│  - Listener: HTTPS port 443          │
└──────────────────────────────────────┘

What's Actually Running:
┌────────────────────────────────────────────────────┐
│ Application Gateway Infrastructure                  │
│ - Multiple VM instances (autoscaling)              │
│ - NGINX-based reverse proxy cluster                │
│ - SSL/TLS termination engine                       │
│ - HTTP/2 and WebSocket support                     │
│ - Layer 7 content inspection                       │
│ - URL path routing engine                          │
│ - HTTP header manipulation                         │
│ - Cookie-based session affinity                    │
│ - Web Application Firewall (WAF engine)            │
│ - Connection multiplexing to backends              │
│ - Health monitoring service                        │
│ - Zone-redundant deployment (V2)                   │
│ - End-to-end TLS encryption capable                │
│ - Request/Response buffering                       │
│ - Metrics and diagnostics collection               │
└────────────────────────────────────────────────────┘
```

**Key Concept**: Azure Application Gateway is NOT just a load balancer—it’s a full reverse proxy that terminates client connections, inspects HTTP content, makes routing decisions based on URLs, modifies requests/responses, and then creates new connections to backend servers.

### Application Gateway vs Load Balancer

**Critical Difference**:

```
Azure Load Balancer (Layer 4):
Client ←→ LB ←→ Backend
        ↑
    Transparent packet forwarding
    LB sees: IP, Port, Protocol
    LB doesn't see: HTTP content

Application Gateway (Layer 7):
Client → AppGW → Backend
    Connection 1 | Connection 2
    
    Two separate TCP connections:
    1. Client → Application Gateway (terminated)
    2. Application Gateway → Backend (new connection)
    
    AppGW sees everything:
    - Full HTTP request (method, URL, headers, body)
    - Cookies
    - TLS/SSL data
    - Makes routing decisions based on content
```

-----

## Layer 2: Application Gateway Components & Structure

### Anatomy of an Application Gateway

An Application Gateway consists of many interconnected components:

#### 1. Application Gateway Resource

```json
{
  "name": "MyAppGateway",
  "location": "westeurope",
  "type": "Microsoft.Network/applicationGateways",
  "sku": {
    "name": "WAF_v2",
    "tier": "WAF_v2",
    "capacity": 2
  },
  "properties": {
    "gatewayIPConfigurations": [],
    "frontendIPConfigurations": [],
    "frontendPorts": [],
    "backendAddressPools": [],
    "backendHttpSettingsCollection": [],
    "httpListeners": [],
    "requestRoutingRules": [],
    "probes": [],
    "sslCertificates": [],
    "urlPathMaps": [],
    "redirectConfigurations": [],
    "rewriteRuleSets": [],
    "webApplicationFirewallConfiguration": {},
    "autoscaleConfiguration": {}
  }
}
```

**SKU Comparison**:

|Feature               |Standard V2  |WAF V2              |
|----------------------|-------------|--------------------|
|Layer 7 load balancing|✓            |✓                   |
|SSL termination       |✓            |✓                   |
|Autoscaling           |✓            |✓                   |
|Zone redundancy       |✓            |✓                   |
|Static VIP            |✓            |✓                   |
|WAF protection        |✗            |✓                   |
|OWASP core rules      |✗            |✓                   |
|Bot protection        |✗            |✓                   |
|Custom WAF rules      |✗            |✓                   |
|Cost                  |Lower        |Higher              |
|Use case              |Internal apps|Internet-facing apps|

**V1 vs V2 Comparison**:

|Feature           |V1     |V2                 |
|------------------|-------|-------------------|
|Autoscaling       |✗      |✓                  |
|Zone redundancy   |✗      |✓                  |
|Static VIP        |✗      |✓                  |
|Header rewrite    |✗      |✓                  |
|WAF custom rules  |Limited|Advanced           |
|Performance       |Good   |Better (35% faster)|
|Deployment time   |~30 min|~6 min             |
|**Recommendation**|Legacy |**Use this**       |

#### 2. Gateway IP Configuration

**Subnet Requirement**: Application Gateway needs a **dedicated subnet**.

```json
{
  "name": "appGatewayIpConfig",
  "properties": {
    "subnet": {
      "id": "/subscriptions/.../subnets/AppGatewaySubnet"
    }
  }
}
```

**Subnet Requirements**:

```
Minimum Subnet Size:
- V2 SKU: /24 or larger recommended
- At least /27 (32 IPs) absolute minimum
- Dedicated subnet (no other resources)

Why Dedicated Subnet?
- App Gateway deploys multiple instances
- Each instance needs IP address
- Autoscaling needs room to grow
- Updates require additional capacity

Example:
Subnet: 10.0.1.0/24 (256 IPs)
- Azure reserves: 5 IPs (.0, .1, .2, .3, .255)
- Available: 251 IPs
- Max instances: 125 (V2 can scale to 125)
- Recommendation: Don't exceed 80% utilization
```

**IP Allocation**:

```
Application Gateway Instances:
┌────────────────────────────────────────┐
│ AppGateway Subnet: 10.0.1.0/24         │
│                                        │
│ ├─ Instance 1: 10.0.1.4                │
│ ├─ Instance 2: 10.0.1.5                │
│ ├─ Instance 3: 10.0.1.6 (autoscaled)   │
│ └─ Instance N: 10.0.1.X                │
│                                        │
│ Frontend (what users connect to):      │
│ - Public IP: 20.50.60.70               │
│ - Private IP: 10.0.1.100 (optional)    │
└────────────────────────────────────────┘

Client connects to: 20.50.60.70
Internally routed to: Any instance (10.0.1.4-X)
All instances share same configuration
```

#### 3. Frontend IP Configuration

The **Frontend IP** is what clients connect to—your application gateway’s address.

**Public Frontend**:

```json
{
  "name": "appGatewayFrontendIP",
  "properties": {
    "publicIPAddress": {
      "id": "/subscriptions/.../publicIPAddresses/AppGatewayPublicIP"
    }
  }
}
```

**Private Frontend** (Internal Application Gateway):

```json
{
  "name": "appGatewayPrivateFrontendIP",
  "properties": {
    "privateIPAddress": "10.0.1.100",
    "privateIPAllocationMethod": "Static",
    "subnet": {
      "id": "/subscriptions/.../subnets/AppGatewaySubnet"
    }
  }
}
```

**Dual-Stack** (Both Public and Private):

```
Single Application Gateway:
├─ Frontend 1: Public IP (20.50.60.70)
│  └─ Listener: External traffic
│
└─ Frontend 2: Private IP (10.0.1.100)
   └─ Listener: Internal traffic

Use Cases:
- Public: Internet users
- Private: Internal corporate users
- Same backends, different frontends
```

#### 4. Frontend Ports

**Frontend Ports** define which ports the gateway listens on:

```json
{
  "name": "port_80",
  "properties": {
    "port": 80
  }
},
{
  "name": "port_443",
  "properties": {
    "port": 443
  }
},
{
  "name": "port_8080",
  "properties": {
    "port": 8080
  }
}
```

**Common Port Configurations**:

```
Standard Web Application:
├─ Port 80 (HTTP)
│  └─ Redirect to HTTPS
└─ Port 443 (HTTPS)
   └─ Main application

Multiple Applications:
├─ Port 443 → app1.example.com
├─ Port 8443 → app2.example.com
└─ Port 9443 → admin.example.com

Port Limits:
- V2 SKU: Up to 100 listeners
- Each listener needs a port
- Ports 65200-65535 reserved for health probes
```

#### 5. Backend Pools

**Backend Pools** contain the targets that serve requests:

```json
{
  "name": "WebServersPool",
  "properties": {
    "backendAddresses": [
      {
        "fqdn": "webserver1.contoso.com"
      },
      {
        "ipAddress": "10.0.2.4"
      },
      {
        "fqdn": "myapp.azurewebsites.net"
      }
    ]
  }
}
```

**Backend Target Types**:

```
Supported Backend Types:
├── Virtual Machine IP addresses
├── Virtual Machine NIC names
├── Virtual Machine Scale Sets (VMSS)
├── IP addresses (any routable IP)
├── Fully Qualified Domain Names (FQDN)
├── Azure App Service
├── Azure Container Instances
└── Azure Kubernetes Service (AKS) pods

Example Mix:
Backend Pool: "ApplicationServers"
├─ VM1: 10.0.2.4 (Azure VM)
├─ VM2: 10.0.2.5 (Azure VM)
├─ AppService: myapp.azurewebsites.net
└─ External: api.partner.com (external API)

Flexibility:
✓ Mix different types in same pool
✓ Backends can be in different VNets (with peering)
✓ Backends can be on-premises (via VPN/ExpressRoute)
✓ Backends can be external (internet)
```

**FQDN Backends** (Special Feature):

```
FQDN Backend Resolution:

Configuration:
Backend: "api.example.com"

Application Gateway Behavior:
1. DNS lookup: api.example.com → 93.184.216.34
2. Create connection to: 93.184.216.34
3. Send HTTP request with Host header: api.example.com
4. If IP changes (DNS update):
   → App Gateway re-resolves automatically
   → Follows DNS TTL
   → Adapts to IP changes

Use Cases:
✓ Azure App Service (IP changes frequently)
✓ External APIs
✓ Multi-region backends with failover
✓ CDN origins
```

#### 6. Backend HTTP Settings

**Backend HTTP Settings** define HOW Application Gateway talks to backends:

```json
{
  "name": "appGatewayBackendHttpSettings",
  "properties": {
    "port": 80,
    "protocol": "Http",
    "cookieBasedAffinity": "Disabled",
    "connectionDraining": {
      "enabled": true,
      "drainTimeoutInSec": 60
    },
    "requestTimeout": 30,
    "probe": {
      "id": "/subscriptions/.../probes/customProbe"
    },
    "pickHostNameFromBackendAddress": false,
    "hostName": "www.contoso.com",
    "path": "/api/",
    "affinityCookieName": "ApplicationGatewayAffinity"
  }
}
```

**Backend Settings Properties**:

|Property                      |Purpose             |Values              |Example        |
|------------------------------|--------------------|--------------------|---------------|
|port                          |Backend port        |1-65535             |80, 443, 8080  |
|protocol                      |HTTP or HTTPS       |Http, Https         |Https          |
|cookieBasedAffinity           |Session persistence |Enabled, Disabled   |Enabled        |
|connectionDraining            |Graceful shutdown   |true/false + timeout|true, 60s      |
|requestTimeout                |Backend timeout     |1-86400 seconds     |30             |
|pickHostNameFromBackendAddress|Use FQDN as Host    |true/false          |true           |
|hostName                      |Override Host header|FQDN                |www.contoso.com|
|path                          |Prepend path        |URL path            |/api/          |
|probe                         |Custom health probe |Probe ID            |customProbe    |

**Protocol Options**:

```
HTTP to Backend:
Client → HTTPS → AppGW → HTTP → Backend
        (encrypted)      (unencrypted)

Use when:
✓ Backends are trusted (same VNet)
✓ Don't need end-to-end encryption
✓ Better performance (no backend SSL overhead)

HTTPS to Backend (End-to-End TLS):
Client → HTTPS → AppGW → HTTPS → Backend
        (encrypted)      (encrypted)

Use when:
✓ Compliance requires encryption
✓ Backends in different network
✓ Zero-trust architecture
✓ Sensitive data in transit

Configuration:
{
  "protocol": "Https",
  "trustedRootCertificates": [
    {"id": ".../trustedRootCertificates/BackendCert"}
  ]
}

Backend certificate must:
- Match hostname or be trusted
- Not be self-signed (unless uploaded as trusted root)
```

**Cookie-Based Affinity** (Sticky Sessions):

```
Session Affinity Enabled:

Request 1 from Client A:
Client A → AppGW
  ↓ (Selects backend: Server1)
AppGW → Server1

Response:
Server1 → AppGW
  ↓ (Injects cookie)
AppGW → Client A
Set-Cookie: ApplicationGatewayAffinity=abc123; Path=/

Request 2 from Client A:
Client A → AppGW
Cookie: ApplicationGatewayAffinity=abc123
  ↓ (Reads cookie, routes to same backend)
AppGW → Server1 (same server!)

Benefits:
✓ Stateful applications work correctly
✓ Shopping carts preserved
✓ User sessions maintained

Considerations:
- Cookie name customizable
- Applies per backend pool
- Survives backend instance changes (graceful)
```

**Connection Draining**:

```
Scenario: Backend VM being updated/removed

Without Connection Draining:
1. Admin removes VM from backend pool
2. Active connections: TERMINATED immediately
3. Users: See errors, broken connections
4. User experience: BAD

With Connection Draining (60 seconds):
1. Admin removes VM from backend pool
2. App Gateway marks VM as "draining"
3. New connections: Go to other backends
4. Existing connections: Continue for 60 seconds
5. After 60s: Remaining connections closed
6. User experience: SMOOTH

Configuration:
{
  "connectionDraining": {
    "enabled": true,
    "drainTimeoutInSec": 60
  }
}

Typical timeouts:
- Web apps: 30-60 seconds
- API calls: 10-30 seconds
- Long-polling: 120-300 seconds
```

#### 7. HTTP Listeners

**Listeners** define what Application Gateway listens for:

```json
{
  "name": "appGatewayHttpsListener",
  "properties": {
    "frontendIPConfiguration": {
      "id": ".../frontendIPConfigurations/appGatewayFrontendIP"
    },
    "frontendPort": {
      "id": ".../frontendPorts/port_443"
    },
    "protocol": "Https",
    "sslCertificate": {
      "id": ".../sslCertificates/appGatewaySslCert"
    },
    "hostName": "www.contoso.com",
    "requireServerNameIndication": true,
    "customErrorConfigurations": [
      {
        "statusCode": "HttpStatus502",
        "customErrorPageUrl": "https://storage.../error502.html"
      }
    ]
  }
}
```

**Listener Types**:

```
1. Basic Listener:
   - Single site
   - One hostname or *
   - Example: HTTP on port 80

2. Multi-Site Listener:
   - Multiple hostnames
   - Host-based routing
   - Example: app1.com, app2.com both on port 443

Configuration Comparison:

Basic Listener:
{
  "protocol": "Http",
  "frontendPort": 80,
  "hostName": null  // Accepts all hostnames
}

Multi-Site Listener:
{
  "protocol": "Https",
  "frontendPort": 443,
  "hostName": "www.contoso.com",  // Specific hostname
  "requireServerNameIndication": true
}
```

**Multi-Site Hosting**:

```
Single Application Gateway:
Frontend IP: 20.50.60.70

Listener 1:
├─ Port: 443
├─ Hostname: www.contoso.com
├─ SSL Cert: contoso-cert
└─ Routes to: Backend Pool 1 (Contoso servers)

Listener 2:
├─ Port: 443
├─ Hostname: www.fabrikam.com
├─ SSL Cert: fabrikam-cert
└─ Routes to: Backend Pool 2 (Fabrikam servers)

Listener 3:
├─ Port: 443
├─ Hostname: api.adventure-works.com
├─ SSL Cert: adventure-cert
└─ Routes to: Backend Pool 3 (API servers)

DNS Configuration:
www.contoso.com → 20.50.60.70
www.fabrikam.com → 20.50.60.70
api.adventure-works.com → 20.50.60.70

How Routing Works:
1. All three domains point to same IP
2. Client connects and sends SNI (Server Name Indication)
3. App Gateway matches SNI to listener hostname
4. Correct certificate presented
5. Request routed to correct backend pool

Cost Savings:
- One Application Gateway
- Multiple websites
- Single public IP
- Consolidated management
```

**Server Name Indication (SNI)**:

```
TLS Handshake with SNI:

Client → AppGW:
ClientHello {
  TLS Version: 1.3
  Cipher Suites: [...]
  SNI Extension: "www.contoso.com"  ← Client tells server which hostname
}

AppGW:
1. Reads SNI: "www.contoso.com"
2. Matches to listener with hostname: "www.contoso.com"
3. Selects SSL certificate for contoso.com
4. Responds with correct certificate

AppGW → Client:
ServerHello {
  Selected Cipher: TLS_AES_256_GCM_SHA384
  Certificate: CN=www.contoso.com
}

Without SNI:
- Can't host multiple HTTPS sites on same IP:port
- Would only work with wildcard cert
- Or single hostname

With SNI (requireServerNameIndication: true):
✓ Multiple HTTPS sites on same IP:port
✓ Each with own certificate
✓ Modern standard (supported by all browsers)
```

#### 8. Request Routing Rules

**Routing Rules** connect listeners to backend pools:

**Basic Rule**:

```json
{
  "name": "rule1",
  "properties": {
    "ruleType": "Basic",
    "httpListener": {
      "id": ".../httpListeners/appGatewayHttpsListener"
    },
    "backendAddressPool": {
      "id": ".../backendAddressPools/WebServersPool"
    },
    "backendHttpSettings": {
      "id": ".../backendHttpSettingsCollection/appGatewayBackendHttpSettings"
    },
    "priority": 100
  }
}
```

**Path-Based Rule** (URL Routing):

```json
{
  "name": "rule2",
  "properties": {
    "ruleType": "PathBasedRouting",
    "httpListener": {
      "id": ".../httpListeners/appGatewayHttpListener"
    },
    "urlPathMap": {
      "id": ".../urlPathMaps/urlPathMap"
    },
    "priority": 200
  }
}
```

**Rule Types Explained**:

```
Basic Rule:
Listener → Backend Pool
Simple 1:1 mapping

Example:
www.contoso.com:443 → WebServers backend pool

Path-Based Rule:
Listener → URL Path Map → Multiple Backend Pools
Routes based on URL path

Example:
www.contoso.com/images/* → ImageServers
www.contoso.com/api/* → APIServers
www.contoso.com/* → WebServers (default)
```

#### 9. URL Path Maps (Path-Based Routing)

**URL Path Maps** enable routing based on URL paths:

```json
{
  "name": "urlPathMap",
  "properties": {
    "defaultBackendAddressPool": {
      "id": ".../backendAddressPools/WebServersPool"
    },
    "defaultBackendHttpSettings": {
      "id": ".../backendHttpSettingsCollection/appGatewayBackendHttpSettings"
    },
    "pathRules": [
      {
        "name": "ImagesRule",
        "properties": {
          "paths": [
            "/images/*"
          ],
          "backendAddressPool": {
            "id": ".../backendAddressPools/ImageServersPool"
          },
          "backendHttpSettings": {
            "id": ".../backendHttpSettingsCollection/imageBackendSettings"
          }
        }
      },
      {
        "name": "APIRule",
        "properties": {
          "paths": [
            "/api/*"
          ],
          "backendAddressPool": {
            "id": ".../backendAddressPools/APIServersPool"
          },
          "backendHttpSettings": {
            "id": ".../backendHttpSettingsCollection/apiBackendSettings"
          }
        }
      }
    ]
  }
}
```

**Path-Based Routing Example**:

```
Application: www.contoso.com

URL Path Routing:
┌──────────────────────────────────────────┐
│ www.contoso.com/images/logo.png          │
│ → Path: /images/*                        │
│ → Backend: ImageServers (CDN, storage)   │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ www.contoso.com/api/users                │
│ → Path: /api/*                           │
│ → Backend: APIServers (API microservice) │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ www.contoso.com/about                    │
│ → Path: /* (default)                     │
│ → Backend: WebServers (web app)          │
└──────────────────────────────────────────┘

Path Matching Rules:
- Most specific path wins
- /api/users/1 matches /api/* (not /*)
- Case-sensitive matching
- Wildcards: /path/* or /path*
- Query strings ignored for matching
```

**Complex Path-Based Routing**:

```
Real-World Microservices Architecture:

www.contoso.com/auth/*
  → AuthenticationService (10.0.3.0/24)
  
www.contoso.com/api/orders/*
  → OrderService (10.0.4.0/24)
  
www.contoso.com/api/inventory/*
  → InventoryService (10.0.5.0/24)
  
www.contoso.com/api/payment/*
  → PaymentService (10.0.6.0/24)
  
www.contoso.com/static/*
  → StaticContentCDN (Azure Storage)
  
www.contoso.com/*
  → WebFrontend (10.0.2.0/24)

Benefits:
✓ Single public IP for all services
✓ Centralized SSL termination
✓ Each microservice scales independently
✓ Path-based routing to correct service
✓ WAF protection for all services
```

#### 10. Health Probes

**Health Probes** continuously monitor backend health:

**Default Probe** (Automatic):

```
Application Gateway automatically creates default probe:
- Protocol: Same as backend setting
- Path: /
- Interval: 30 seconds
- Timeout: 30 seconds
- Unhealthy threshold: 3 consecutive failures
```

**Custom Probe**:

```json
{
  "name": "customHealthProbe",
  "properties": {
    "protocol": "Http",
    "host": "www.contoso.com",
    "path": "/health",
    "interval": 30,
    "timeout": 30,
    "unhealthyThreshold": 3,
    "pickHostNameFromBackendHttpSettings": false,
    "minServers": 0,
    "match": {
      "statusCodes": [
        "200-399"
      ],
      "body": "healthy"
    }
  }
}
```

**Probe Properties**:

|Property          |Purpose                  |Example        |
|------------------|-------------------------|---------------|
|protocol          |HTTP or HTTPS            |Http           |
|host              |Host header in probe     |www.contoso.com|
|path              |URL path                 |/health        |
|interval          |Seconds between probes   |30             |
|timeout           |Max wait for response    |30             |
|unhealthyThreshold|Failures before unhealthy|3              |
|match.statusCodes |Expected HTTP codes      |200-399        |
|match.body        |Expected body substring  |“healthy”      |

**Health Probe Flow**:

```
Probe Configuration:
Path: /health
Interval: 30 seconds
Unhealthy threshold: 3

Timeline:

T+0s: Backend: Server1 (Healthy)
  Probe → Server1 → 200 OK ✓
  Status: Healthy

T+30s: Backend: Server1
  Probe → Server1 → 200 OK ✓
  Status: Healthy

T+60s: Backend: Server1 (Application issue)
  Probe → Server1 → 503 Service Unavailable ✗
  Failure count: 1
  Status: Still Healthy (needs 3 failures)

T+90s: Backend: Server1
  Probe → Server1 → Timeout ✗
  Failure count: 2
  Status: Still Healthy

T+120s: Backend: Server1
  Probe → Server1 → Timeout ✗
  Failure count: 3
  Status: Marked UNHEALTHY
  Action: Stop sending traffic to Server1

T+150s: Backend: Server1 (Application recovered)
  Probe → Server1 → 200 OK ✓
  Status: Marked HEALTHY
  Action: Resume sending traffic to Server1

Detection Time:
Minimum: unhealthyThreshold * interval
Example: 3 * 30s = 90 seconds to detect failure
```

**Advanced Health Probes**:

```
Health Check Best Practices:

❌ Bad: Probe root path
GET / HTTP/1.1
- Might be cached
- Doesn't verify backend health
- Could return 200 even if broken

✓ Good: Probe dedicated health endpoint
GET /health HTTP/1.1
- Checks application health
- Verifies database connectivity
- Returns status of dependencies

Example Health Endpoint (Node.js):
app.get('/health', async (req, res) => {
  try {
    // Check database
    await db.ping();
    
    // Check critical dependency
    const apiHealthy = await checkExternalAPI();
    
    // Check disk space
    const diskSpace = await checkDiskSpace();
    
    if (apiHealthy && diskSpace > 10) {
      res.status(200).json({
        status: 'healthy',
        database: 'connected',
        api: 'reachable',
        diskSpace: diskSpace + 'GB'
      });
    } else {
      res.status(503).json({
        status: 'unhealthy',
        reason: 'dependency failure'
      });
    }
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

Probe Configuration:
{
  "path": "/health",
  "match": {
    "statusCodes": ["200"],
    "body": "healthy"  // Must contain this string
  }
}
```

#### 11. SSL Certificates

**SSL/TLS Certificates** for HTTPS:

**Certificate Upload**:

```json
{
  "name": "appGatewaySslCert",
  "properties": {
    "data": "MIIKcQIBAzCCCi0GCSqGSIb3DQEHAaCCCh...",
    "password": "P@ssw0rd"
  }
}
```

**Certificate Types**:

```
Frontend Certificates (Client → AppGW):
- PFX/PKCS12 format
- Contains private key
- Used for SSL termination
- One cert per hostname (or wildcard)

Backend Certificates (AppGW → Backend):
- CER/PEM format (public key only)
- For end-to-end TLS
- Uploaded as "Trusted Root Certificates"
- Validates backend certificates
```

**SSL Certificate Management**:

```
Scenario 1: Single Domain
www.contoso.com

Certificate:
CN: www.contoso.com
- Standard SSL certificate
- Covers one hostname

Scenario 2: Multiple Subdomains
www.contoso.com
app.contoso.com
api.contoso.com

Certificate (Wildcard):
CN: *.contoso.com
- Covers all subdomains
- Single certificate
- Easier management

Scenario 3: Multiple Domains
www.contoso.com
www.fabrikam.com
www.adventure-works.com

Certificates:
- Cert 1: CN=www.contoso.com
- Cert 2: CN=www.fabrikam.com
- Cert 3: CN=www.adventure-works.com

Or Multi-Domain Certificate (SAN):
CN: www.contoso.com
SAN: www.fabrikam.com, www.adventure-works.com
```

**SSL Policy**:

```json
{
  "sslPolicy": {
    "policyType": "Predefined",
    "policyName": "AppGwSslPolicy20220101",
    "minProtocolVersion": "TLSv1_2",
    "cipherSuites": [
      "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
      "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
    ]
  }
}
```

**Predefined SSL Policies**:

```
AppGwSslPolicy20220101 (Recommended):
- Min TLS: 1.2
- Strong ciphers only
- Modern security

AppGwSslPolicy20170401S (Strict):
- Min TLS: 1.2
- Very limited ciphers
- Maximum security
- May break old clients

AppGwSslPolicy20150501 (Legacy):
- Min TLS: 1.0
- Weak ciphers allowed
- Compatibility mode
- ⚠️ Not recommended
```

#### 12. Rewrite Rules

**Rewrite Rules** modify HTTP headers and URLs:

**HTTP Header Rewrite**:

```json
{
  "name": "headerRewriteRule",
  "properties": {
    "ruleSequence": 100,
    "conditions": [],
    "actionSet": {
      "requestHeaderConfigurations": [
        {
          "headerName": "X-Forwarded-For",
          "headerValue": "{var_client_ip}"
        },
        {
          "headerName": "X-Original-Host",
          "headerValue": "{var_host}"
        }
      ],
      "responseHeaderConfigurations": [
        {
          "headerName": "Strict-Transport-Security",
          "headerValue": "max-age=31536000; includeSubDomains"
        },
        {
          "headerName": "X-Content-Type-Options",
          "headerValue": "nosniff"
        }
      ]
    }
  }
}
```

**URL Rewrite**:

```json
{
  "name": "urlRewriteRule",
  "properties": {
    "ruleSequence": 100,
    "conditions": [
      {
        "variable": "http_req_uri",
        "pattern": "/old-path/(.*)",
        "ignoreCase": true
      }
    ],
    "actionSet": {
      "urlConfiguration": {
        "modifiedPath": "/new-path/{http_req_uri_1}",
        "reroute": false
      }
    }
  }
}
```

**Common Rewrite Scenarios**:

```
1. Add Security Headers:
Response Headers:
+ Strict-Transport-Security: max-age=31536000
+ X-Frame-Options: DENY
+ X-Content-Type-Options: nosniff
+ X-XSS-Protection: 1; mode=block

2. Preserve Client IP:
Request Headers:
+ X-Forwarded-For: {client_ip}
+ X-Forwarded-Proto: https
+ X-Original-Host: www.contoso.com

Backend sees:
X-Forwarded-For: 1.2.3.4 (original client)
Instead of: Application Gateway IP

3. Remove Server Identity:
Response Headers:
- Server: nginx/1.18.0 (removed)
- X-Powered-By: PHP/7.4 (removed)

Security benefit: Don't reveal backend technology

4. URL Rewriting:
Client requests: /api/v1/users
Backend receives: /users

Or:
Client requests: /blog/post-title
Rewrite to: /blog.php?slug=post-title

5. Header-Based Routing:
Condition: X-Version header = "beta"
Action: Route to beta backend pool
```

**Server Variables**:

```
Available Variables:

Client Variables:
{var_client_ip} → Client's IP address
{var_client_port} → Client's source port
{var_client_user} → Authenticated username

Request Variables:
{var_uri_path} → Request URI path
{var_uri_query} → Query string
{var_host} → Host header
{var_http_method} → HTTP method (GET, POST, etc.)

SSL Variables:
{var_ssl_protocol} → TLS version (TLSv1.2, TLSv1.3)
{var_ssl_cipher} → Cipher suite used

Example Usage:
Add header to backend request:
X-Client-Info: {var_client_ip}:{var_client_port}

Backend receives:
X-Client-Info: 1.2.3.4:52000
```

#### 13. Web Application Firewall (WAF)

**WAF Configuration** (WAF V2 SKU only):

```json
{
  "webApplicationFirewallConfiguration": {
    "enabled": true,
    "firewallMode": "Prevention",
    "ruleSetType": "OWASP",
    "ruleSetVersion": "3.2",
    "disabledRuleGroups": [],
    "requestBodyCheck": true,
    "maxRequestBodySizeInKb": 128,
    "fileUploadLimitInMb": 100,
    "exclusions": []
  }
}
```

**WAF Modes**:

```
Detection Mode:
- Logs threats
- Does NOT block
- Good for testing/tuning
- No impact on legitimate users

Prevention Mode:
- Logs threats
- BLOCKS malicious requests
- Production mode
- Protects application

Mode Transition:
1. Start with Detection
2. Monitor logs for false positives
3. Tune rules (add exclusions)
4. Switch to Prevention
5. Monitor and adjust
```

**OWASP Core Rule Sets**:

```
OWASP CRS 3.2 (Recommended):
Protection against:
├─ SQL Injection
├─ Cross-Site Scripting (XSS)
├─ Local File Inclusion (LFI)
├─ Remote File Inclusion (RFI)
├─ Command Injection
├─ Session Fixation
├─ HTTP Protocol Violations
├─ Scanner Detection
├─ Malicious User Agents
└─ Information Leakage

Rule Groups:
- REQUEST-911-METHOD-ENFORCEMENT
- REQUEST-913-SCANNER-DETECTION
- REQUEST-920-PROTOCOL-ENFORCEMENT
- REQUEST-921-PROTOCOL-ATTACK
- REQUEST-930-APPLICATION-ATTACK-LFI
- REQUEST-931-APPLICATION-ATTACK-RFI
- REQUEST-932-APPLICATION-ATTACK-RCE
- REQUEST-933-APPLICATION-ATTACK-PHP
- REQUEST-941-APPLICATION-ATTACK-XSS
- REQUEST-942-APPLICATION-ATTACK-SQLI
- REQUEST-943-APPLICATION-ATTACK-SESSION-FIXATION
```

**WAF Custom Rules**:

```json
{
  "customRules": [
    {
      "name": "BlockCountry",
      "priority": 1,
      "ruleType": "MatchRule",
      "matchConditions": [
        {
          "matchVariables": [
            {
              "variableName": "RemoteAddr"
            }
          ],
          "operator": "GeoMatch",
          "matchValues": [
            "CN",
            "RU"
          ]
        }
      ],
      "action": "Block"
    },
    {
      "name": "RateLimitPerIP",
      "priority": 2,
      "ruleType": "RateLimitRule",
      "rateLimitDuration": "OneMin",
      "rateLimitThreshold": 100,
      "matchConditions": [
        {
          "matchVariables": [
            {
              "variableName": "RemoteAddr"
            }
          ],
          "operator": "IPMatch",
          "matchValues": [
            "0.0.0.0/0"
          ]
        }
      ],
      "action": "Block"
    },
    {
      "name": "BlockSpecificUserAgent",
      "priority": 3,
      "ruleType": "MatchRule",
      "matchConditions": [
        {
          "matchVariables": [
            {
              "variableName": "RequestHeaders",
              "selector": "User-Agent"
            }
          ],
          "operator": "Contains",
          "matchValues": [
            "badbot",
            "scanner"
          ]
        }
      ],
      "action": "Block"
    }
  ]
}
```

**WAF Protection Example**:

```
Attack Scenario: SQL Injection

Malicious Request:
GET /users?id=1' OR '1'='1 HTTP/1.1
Host: www.contoso.com

WAF Analysis:
1. Parse request
2. Check against SQL injection rules (REQUEST-942)
3. Detect SQL injection pattern: ' OR '1'='1
4. Match Rule: 942100 (SQL Injection Attack)
5. Anomaly Score: +5 (critical severity)
6. Total Anomaly Score: 5 (exceeds threshold)

WAF Action (Prevention Mode):
7. Block request
8. Return HTTP 403 Forbidden
9. Log incident

Response to Client:
HTTP/1.1 403 Forbidden
Content-Type: text/html

<html>
<body>
<h1>403 Forbidden</h1>
<p>Your request has been blocked.</p>
</body>
</html>

Backend Impact:
✓ Backend never receives malicious request
✓ Database protected from injection
✓ Application remains secure

Logs:
{
  "action": "Blocked",
  "ruleId": "942100",
  "message": "SQL Injection Attack Detected",
  "clientIp": "1.2.3.4",
  "requestUri": "/users?id=1' OR '1'='1",
  "severity": "Critical"
}
```

#### 14. Autoscaling Configuration

**Autoscaling** (V2 SKU only):

```json
{
  "autoscaleConfiguration": {
    "minCapacity": 2,
    "maxCapacity": 10
  }
}
```

**How Autoscaling Works**:

```
Capacity Units (CU):
Metric that combines:
- Throughput (Mbps)
- Connections per second
- Compute (CPU)

Each instance provides:
- 10 Compute Units
- 2,500 persistent connections
- 2.22 Mbps throughput

Scaling Triggers:
If current utilization > 75% for 5 minutes:
  → Scale out (add instance)

If current utilization < 25% for 15 minutes:
  → Scale in (remove instance)

Example:
Min: 2 instances (always running)
Max: 10 instances (ceiling)

Load increases:
T+0min: 2 instances, 70% utilization
T+5min: Load spike, 85% utilization
T+6min: Scale out triggered, adding instance
T+8min: 3 instances, 60% utilization (distributed)

Load decreases:
T+30min: Load drops, 20% utilization
T+45min: Scale in triggered (after 15 min)
T+47min: 2 instances (at minimum)

Cost Impact:
- Only pay for what you use
- Scale automatically with traffic
- No manual intervention
```

-----

## Layer 3: How Application Gateway Works

### Request Processing Overview

**High-Level Flow**:

```
1. Client Request:
   User → Application Gateway
   - TCP connection established
   - TLS handshake (if HTTPS)
   - HTTP request sent

2. Application Gateway Processing:
   - Terminates client connection
   - Parses HTTP request fully
   - Matches listener (host + port)
   - Applies WAF rules (if enabled)
   - Matches routing rule
   - Selects backend (path-based routing)
   - Applies rewrite rules
   - Selects healthy backend server
   - Creates new connection to backend

3. Backend Processing:
   Application Gateway → Backend
   - New TCP connection
   - TLS handshake (if HTTPS backend)
   - Modified HTTP request sent
   - Backend processes request

4. Backend Response:
   Backend → Application Gateway
   - HTTP response received
   - Application Gateway processes response
   - Applies response rewrite rules
   - Buffers response

5. Client Response:
   Application Gateway → User
   - Sends response on original connection
   - Connection kept alive (HTTP keep-alive)
```

### Connection Handling

**Two-Connection Model**:

```
Traditional Load Balancer (Layer 4):
┌────────┐         ┌────────┐         ┌────────┐
│ Client │←═══════→│   LB   │←═══════→│Backend │
└────────┘         └────────┘         └────────┘
         Single logical connection
         (LB just forwards packets)

Application Gateway (Layer 7):
┌────────┐         ┌──────────┐         ┌────────┐
│ Client │←═══════→│  AppGW   │←═══════→│Backend │
└────────┘         └──────────┘         └────────┘
    Connection 1       ↑            Connection 2
                   Terminates &
                   Re-establishes

Key Difference:
- AppGW terminates client connection
- Fully parses HTTP
- Makes intelligent routing decisions
- Creates separate backend connection
```

**Connection Multiplexing**:

```
Multiple Clients → Fewer Backend Connections

Without Connection Multiplexing:
Client 1 → AppGW → Backend Connection 1
Client 2 → AppGW → Backend Connection 2
Client 3 → AppGW → Backend Connection 3
...
100 clients = 100 backend connections

With Connection Multiplexing (AppGW does this):
Client 1 ┐
Client 2 ├→ AppGW → Backend Connection 1
Client 3 ┘                (reused)

Client 4 ┐
Client 5 ├→ AppGW → Backend Connection 2
Client 6 ┘                (reused)

100 clients = ~10-20 backend connections

Benefits:
✓ Reduced backend load
✓ Fewer TCP connections
✓ Better resource utilization
✓ Improved performance

How It Works:
1. Client 1 request arrives
2. AppGW establishes backend connection
3. Sends Client 1 request
4. Client 2 request arrives
5. AppGW reuses same backend connection (HTTP/1.1 pipelining or HTTP/2)
6. Backend sees sequential requests on same connection
```

### Backend Selection Algorithm

**Default Algorithm** (Round Robin with Health):

```
Backend Pool: [Server1, Server2, Server3]

Request Flow:
Request 1 → Server1 (healthy) ✓
Request 2 → Server2 (healthy) ✓
Request 3 → Server3 (healthy) ✓
Request 4 → Server1 (cycles back)
Request 5 → Server2
Request 6 → Server3

Server2 Fails Health Probe:
Backend Pool: [Server1, Server3] (Server2 removed)

Request 7 → Server1 ✓
Request 8 → Server3 ✓
Request 9 → Server1 ✓
(Server2 skipped automatically)

Server2 Recovers:
Backend Pool: [Server1, Server2, Server3] (Server2 added back)

Request 10 → Server2 ✓
Request 11 → Server3 ✓
```

**With Cookie-Based Affinity**:

```
Client A - First Request:
Request → AppGW
  ↓ (Selects backend: Server1)
AppGW → Server1

Response:
Server1 → AppGW
  ↓ (Injects affinity cookie)
AppGW → Client A
Set-Cookie: ApplicationGatewayAffinity=server1-abc123

Client A - Subsequent Requests:
Request + Cookie: ApplicationGatewayAffinity=server1-abc123
  ↓ (Reads cookie, maintains affinity)
AppGW → Server1 (always same server)

Client B - First Request:
Request (no cookie) → AppGW
  ↓ (Selects backend: Server2 - different from Client A)
AppGW → Server2

Session Persistence:
✓ Client A always → Server1
✓ Client B always → Server2
✓ Survives AppGW instance changes
✓ Cookie expires with session
```

### URL-Based Routing Decision Tree

```
Request: GET /api/users/123 HTTP/1.1
         Host: www.contoso.com

Step 1: Listener Matching
┌─────────────────────────────────────┐
│ Which listener matches?             │
│ - Port: 443 ✓                       │
│ - Hostname: www.contoso.com ✓       │
│ → Matched: "ContosoListener"        │
└─────────────────────────────────────┘

Step 2: Rule Type
┌─────────────────────────────────────┐
│ Rule: Path-based routing            │
│ URL Path Map: "ContosoPathMap"      │
└─────────────────────────────────────┘

Step 3: Path Matching
┌─────────────────────────────────────┐
│ Request Path: /api/users/123        │
│                                     │
│ Check Paths:                        │
│ /images/* → No match                │
│ /api/* → MATCH! ✓                   │
│ /* (default) → Not evaluated        │
│                                     │
│ → Backend: APIServersPool           │
│ → Settings: APIBackendSettings      │
└─────────────────────────────────────┘

Step 4: Backend Selection
┌─────────────────────────────────────┐
│ Backend Pool: APIServersPool        │
│ Healthy servers:                    │
│ - Server1: 10.0.3.4 ✓               │
│ - Server2: 10.0.3.5 ✓               │
│ - Server3: 10.0.3.6 ✗ (unhealthy)   │
│                                     │
│ Selection: Round robin              │
│ → Selected: Server1 (10.0.3.4)      │
└─────────────────────────────────────┘

Step 5: Request Transformation
┌─────────────────────────────────────┐
│ Original Request:                   │
│ GET /api/users/123 HTTP/1.1         │
│ Host: www.contoso.com               │
│                                     │
│ Apply Rewrites:                     │
│ + X-Forwarded-For: 1.2.3.4          │
│ + X-Forwarded-Proto: https          │
│                                     │
│ Backend Receives:                   │
│ GET /api/users/123 HTTP/1.1         │
│ Host: www.contoso.com               │
│ X-Forwarded-For: 1.2.3.4            │
│ X-Forwarded-Proto: https            │
└─────────────────────────────────────┘

Result: Request sent to 10.0.3.4:8080
```

### WAF Processing Flow

```
Request Flow Through WAF:

1. Request Arrives:
   POST /login HTTP/1.1
   Host: www.contoso.com
   Content-Type: application/x-www-form-urlencoded
   
   username=admin&password=' OR '1'='1

2. WAF Inspection Begins:
   ┌─────────────────────────────────────┐
   │ WAF Engine                          │
   │ Mode: Prevention                    │
   │ Rule Set: OWASP 3.2                 │
   └─────────────────────────────────────┘

3. Rule Evaluation:
   Check HTTP Method: POST ✓ (allowed)
   Check Protocol: HTTP/1.1 ✓
   Check Content-Type: application/x-www-form-urlencoded ✓
   
   Parse Request Body:
   username=admin
   password=' OR '1'='1
   
   SQL Injection Detection:
   Pattern: ' OR '1'='1
   Match: Rule 942100 (SQL Injection Attack)
   Severity: Critical
   Anomaly Score: +5

4. Anomaly Scoring:
   Total Score: 5
   Threshold: 5
   Result: Score >= Threshold → BLOCK

5. Action Taken:
   ┌─────────────────────────────────────┐
   │ Request BLOCKED                     │
   │ - Do not forward to backend         │
   │ - Return 403 Forbidden              │
   │ - Log incident                      │
   └─────────────────────────────────────┘

6. Response to Client:
   HTTP/1.1 403 Forbidden
   Content-Type: text/html
   
   <!DOCTYPE html>
   <html>
   <body>
   <h1>Access Denied</h1>
   <p>Your request was blocked by security policy.</p>
   </body>
   </html>

7. Logging:
   {
     "timestamp": "2025-11-08T12:30:45Z",
     "action": "Blocked",
     "ruleId": "942100",
     "message": "SQL Injection Attack Detected",
     "clientIp": "1.2.3.4",
     "requestUri": "/login",
     "matchedData": "' OR '1'='1",
     "severity": "Critical"
   }

Legitimate Request for Comparison:

1. Request:
   POST /login HTTP/1.1
   username=john.doe@contoso.com
   password=MySecureP@ssw0rd!

2. WAF Inspection:
   No SQL patterns detected ✓
   No XSS patterns detected ✓
   No malicious patterns detected ✓
   Anomaly Score: 0

3. Action:
   ALLOW - Forward to backend

4. Backend Processing:
   AppGW → Backend Server
   Normal authentication flow
```

### SSL/TLS Termination

**SSL Offloading** (HTTPS → HTTP to backend):

```
Client ←→ Application Gateway ←→ Backend

Step 1: Client TLS Handshake
Client → AppGW:
  ClientHello {
    TLS Version: 1.3
    SNI: www.contoso.com
    Cipher Suites: [...]
  }

AppGW → Client:
  ServerHello {
    Certificate: CN=www.contoso.com
    Selected Cipher: TLS_AES_256_GCM_SHA384
  }

TLS Connection Established (encrypted)

Step 2: Client Sends HTTPS Request
Client → AppGW (encrypted):
  [Encrypted] GET /api/data HTTP/1.1

AppGW:
  - Decrypts request
  - Parses HTTP
  - Routes based on URL

Step 3: AppGW → Backend (Unencrypted)
AppGW → Backend (HTTP, not HTTPS):
  GET /api/data HTTP/1.1
  Host: www.contoso.com
  X-Forwarded-For: 1.2.3.4
  X-Forwarded-Proto: https

Backend:
  - Receives plain HTTP
  - No SSL overhead
  - Faster processing

Step 4: Backend Response
Backend → AppGW (HTTP):
  HTTP/1.1 200 OK
  Content-Type: application/json
  {"data": "value"}

Step 5: AppGW → Client (Encrypted)
AppGW → Client (encrypted):
  [Encrypted] HTTP/1.1 200 OK
  [Encrypted] {"data": "value"}

Benefits:
✓ Backend doesn't handle SSL (simpler)
✓ Centralized certificate management
✓ Better backend performance
✓ AppGW handles SSL complexity

Security Consideration:
⚠️ Traffic between AppGW and backend is unencrypted
✓ OK if in same VNet (trusted network)
✗ Not OK if regulatory compliance requires end-to-end encryption
```

**End-to-End TLS** (HTTPS → HTTPS to backend):

```
Client ←→ Application Gateway ←→ Backend
   (TLS Connection 1)        (TLS Connection 2)

Step 1: Client → AppGW TLS
Client → AppGW:
  [Encrypted TLS 1.3]

AppGW:
  - Terminates TLS
  - Decrypts request
  - Parses HTTP

Step 2: AppGW → Backend TLS
AppGW → Backend:
  ClientHello (new TLS connection)

Backend:
  ServerHello {
    Certificate: CN=backend.contoso.internal
  }

AppGW:
  - Validates backend certificate against trusted root
  - Establishes TLS connection

Step 3: Encrypted Request to Backend
AppGW → Backend (encrypted TLS):
  [Encrypted] GET /api/data HTTP/1.1

Backend:
  - Decrypts request
  - Processes
  - Returns encrypted response

Benefits:
✓ End-to-end encryption
✓ Compliance requirements met
✓ Zero-trust architecture
✓ Backend certificate validation

Trade-offs:
✗ Backend must handle SSL (CPU overhead)
✗ Need to manage backend certificates
✗ Slightly higher latency
```

-----

## Layer 4: Under the Hood - The Technology

### Application Gateway Architecture

**Core Components**:

```
Application Gateway Infrastructure:

┌──────────────────────────────────────────────┐
│ Azure Management Plane                        │
│ - ARM APIs                                   │
│ - Configuration storage                      │
│ - Instance orchestration                     │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Application Gateway Instances                 │
│                                              │
│ Instance 1 (10.0.1.4):                       │
│ ┌──────────────────────────────────────────┐│
│ │ NGINX-based Reverse Proxy                ││
│ │ - HTTP parser                            ││
│ │ - SSL/TLS engine (OpenSSL)               ││
│ │ - Load balancing logic                   ││
│ │ - Connection multiplexing                ││
│ └──────────────────────────────────────────┘│
│ ┌──────────────────────────────────────────┐│
│ │ WAF Engine (ModSecurity)                 ││
│ │ - OWASP CRS rules                        ││
│ │ - Custom rules                           ││
│ │ - Anomaly scoring                        ││
│ └──────────────────────────────────────────┘│
│ ┌──────────────────────────────────────────┐│
│ │ Health Probe Agent                       ││
│ │ - Monitors backends                      ││
│ │ - Reports health status                  ││
│ └──────────────────────────────────────────┘│
│                                              │
│ Instance 2-N: (same components)              │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Backend Servers                               │
│ - Your application servers                    │
└──────────────────────────────────────────────┘
```

### NGINX Reverse Proxy

**Application Gateway uses NGINX** as its core HTTP engine:

```
Why NGINX?
✓ High-performance HTTP server
✓ Proven reverse proxy capabilities
✓ Efficient connection handling
✓ Low memory footprint
✓ Event-driven architecture
✓ Supports HTTP/1.1, HTTP/2, WebSocket

NGINX Configuration (Conceptual):

upstream backend_pool_1 {
    least_conn;  # Or round_robin
    server 10.0.2.4:80 weight=1;
    server 10.0.2.5:80 weight=1;
    server 10.0.2.6:80 weight=1;
}

server {
    listen 443 ssl http2;
    server_name www.contoso.com;
    
    ssl_certificate /certs/contoso.pem;
    ssl_certificate_key /certs/contoso.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # WAF processing here (ModSecurity module)
    
    location /api/ {
        proxy_pass http://backend_pool_1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
    
    location / {
        proxy_pass http://backend_pool_2;
    }
}
```

### ModSecurity WAF Engine

**WAF powered by ModSecurity**:

```
ModSecurity Integration:

┌────────────────────────────────────────┐
│ NGINX                                   │
│ ├─ Receives HTTP request               │
│ └─ Passes to ModSecurity module        │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ ModSecurity Engine                      │
│ ├─ Parses HTTP request                 │
│ ├─ Loads OWASP CRS rules               │
│ ├─ Evaluates request against rules     │
│ ├─ Calculates anomaly score            │
│ └─ Returns: ALLOW or BLOCK             │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ Decision                                │
│ ALLOW: Forward to backend               │
│ BLOCK: Return 403 Forbidden             │
└────────────────────────────────────────┘

ModSecurity Rule Processing:

Request: GET /search?q=<script>alert('XSS')</script>

Phase 1: Request Headers
Rule: Check User-Agent for known bad bots ✓

Phase 2: Request Body
Rule: Check for XSS patterns
Pattern: /<script[^>]*>.*?<\/script>/
Match: FOUND ✓
Anomaly Score: +5

Phase 3: Response Headers
(Skipped, request blocked)

Phase 4: Response Body
(Skipped, request blocked)

Phase 5: Logging
Log severity: CRITICAL
Log message: XSS Attack Detected
Action: Block request

Result: 403 Forbidden returned to client
```

### Instance Architecture

**Each Application Gateway Instance**:

```
VM Instance (Azure-managed):
┌────────────────────────────────────────────┐
│ Operating System: Linux (custom)            │
│                                            │
│ Network Interfaces:                        │
│ ├─ Frontend NIC (receives client traffic) │
│ │  IP: 10.0.1.4                            │
│ │  Connected to: AppGW Subnet              │
│ │                                          │
│ └─ Backend NIC (connects to backends)     │
│    IP: 10.0.1.5                            │
│    Connected to: Backend subnets           │
│                                            │
│ Software Stack:                            │
│ ├─ NGINX (reverse proxy)                  │
│ ├─ ModSecurity (WAF)                      │
│ ├─ OpenSSL (TLS/SSL)                      │
│ ├─ Health monitoring agent                │
│ └─ Azure management agent                 │
│                                            │
│ Resources:                                 │
│ - CPU: Multiple cores                     │
│ - Memory: Several GB                      │
│ - Storage: For logs, config              │
└────────────────────────────────────────────┘

Instance Responsibilities:
1. Accept client connections
2. Terminate TLS
3. Parse HTTP requests
4. Apply WAF rules
5. Route to backend
6. Multiplex connections
7. Monitor backend health
8. Return responses to clients
9. Log all activities
```

### Load Distribution Across Instances

**How Traffic Distributes to AppGW Instances**:

```
Frontend IP: 20.50.60.70 (Public IP)

Azure Load Balancer (Internal):
┌────────────────────────────────────────┐
│ Distributes to AppGW instances         │
│ - Uses standard Azure Load Balancer    │
│ - 5-tuple hash distribution            │
│ - Health monitoring                    │
└────────────────────────────────────────┘
        ↓           ↓           ↓
┌─────────────┬─────────────┬─────────────┐
│ Instance 1  │ Instance 2  │ Instance 3  │
│ 10.0.1.4    │ 10.0.1.5    │ 10.0.1.6    │
│ (Active)    │ (Active)    │ (Active)    │
└─────────────┴─────────────┴─────────────┘

Client Request Flow:
1. Client connects to: 20.50.60.70
2. DNS resolves to public IP
3. Request hits Azure edge
4. Internal LB distributes to AppGW instance
5. Selected instance handles full request/response
6. Session affinity maintained by LB (if configured)

Instance Failure:
Instance 2 fails health probe
→ Internal LB stops routing to Instance 2
→ Only Instance 1 and 3 receive traffic
→ Azure automatically replaces Instance 2
→ New instance added to pool
→ Traffic resumes to 3 instances

Autoscaling:
Load increases:
→ Azure adds Instance 4, 5, etc.
→ Internal LB distributes across all
→ Each instance independently processes requests

Load decreases:
→ Azure removes instances
→ Connection draining applied
→ Graceful shutdown
```

### Configuration Synchronization

**How Config Updates Propagate**:

```
Configuration Change Timeline:

T+0s: You update routing rule in portal
      ↓
T+0.1s: ARM API receives request
        - Validates JSON
        - Checks permissions
      ↓
T+0.5s: Configuration stored in database
        - Geo-replicated storage
        - Version controlled
      ↓
T+1s: AppGW controller notified
      - Processes configuration
      - Generates NGINX config
      - Validates rules
      ↓
T+2s: Config pushed to all instances
      - Instance 1: Receives new config
      - Instance 2: Receives new config
      - Instance 3: Receives new config
      ↓
T+3s: Each instance applies config
      - NGINX reload (graceful)
      - No dropped connections
      - Hot configuration update
      ↓
T+5s: All instances updated
      - New config active
      - Portal shows "Succeeded"
      ↓
T+6s: Verification
      - Health probes confirm
      - Traffic flowing normally

Total time: ~5-10 seconds typical

NGINX Graceful Reload:
1. New worker processes started with new config
2. Old worker processes finish existing requests
3. Old workers shut down after completion
4. No connection drops
5. Zero downtime
```

### Health Monitoring System

**Distributed Health Probing**:

```
Health Probe Architecture:

┌────────────────────────────────────────┐
│ AppGW Instance 1                        │
│ ├─ Probe Agent                         │
│ │  └─ Probes: Server1, Server2, Server3│
│ └─ Reports results locally             │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ AppGW Instance 2                        │
│ ├─ Probe Agent                         │
│ │  └─ Probes: Server1, Server2, Server3│
│ └─ Reports results locally             │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ AppGW Instance 3                        │
│ ├─ Probe Agent                         │
│ │  └─ Probes: Server1, Server2, Server3│
│ └─ Reports results locally             │
└────────────────────────────────────────┘

Each instance independently:
1. Sends health probes to all backends
2. Tracks health status locally
3. Makes routing decisions based on health
4. Reports aggregate health to portal

Probe Schedule:
Instance 1: Probes at T+0s, T+30s, T+60s
Instance 2: Probes at T+0s, T+30s, T+60s
Instance 3: Probes at T+0s, T+30s, T+60s

Backend Server1 receives:
- 3 probes every 30 seconds (one from each instance)
- Total: 6 probes per minute

Backend Failure Detection:
T+0s: Server2 healthy (all instances agree)
T+30s: Instance 1 probe fails
       Instance 2 probe fails
       Instance 3 probe succeeds (intermittent)
T+60s: Instance 1 probe fails (2nd failure)
       Instance 2 probe fails (2nd failure)
       Instance 3 probe fails (1st failure)
T+90s: All instances mark Server2 unhealthy
       Server2 removed from all instance pools
       No traffic sent to Server2 by any instance
```

-----

## Layer 5: Mapping to TCP/IP Model

Let’s map Application Gateway operations to the TCP/IP model:

### TCP/IP Layer Overview

```
┌─────────────────────────────────────────────────┐
│ Layer 4: Application Layer                      │
│ - HTTP, HTTPS, WebSocket                        │
│ AppGW: ✓✓✓ PRIMARY OPERATION LAYER             │
│     ✓ Full HTTP parsing and inspection         │
│     ✓ URL-based routing                         │
│     ✓ Header manipulation                       │
│     ✓ Cookie injection                          │
│     ✓ HTTP method filtering                     │
│     ✓ Content-based decisions                   │
│     ✓ WAF content inspection                    │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 3: Transport Layer                        │
│ - TCP, connection management                    │
│ AppGW: ✓ CONNECTION TERMINATION                │
│     ✓ Terminates client TCP connection          │
│     ✓ Creates new backend TCP connection        │
│     ✓ Connection pooling/multiplexing           │
│     ✓ TCP optimization                          │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 2: Internet Layer                         │
│ - IP addressing, routing                        │
│ AppGW: ✓ IP OPERATIONS                         │
│     ✓ Source IP preservation (headers)          │
│     ✓ Routing to backend IPs                    │
│     ✓ DNS resolution for FQDN backends          │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 1: Network Access Layer                   │
│ - Ethernet, physical transmission               │
│ AppGW: Standard Azure networking                │
└─────────────────────────────────────────────────┘
```

### Layer 4: Application Layer (HTTP/HTTPS)

**This is where Application Gateway shines!**

```
HTTP Request Structure:
┌─────────────────────────────────────────┐
│ HTTP Request Line                       │
│ GET /api/users/123 HTTP/1.1             │ ← AppGW parses this
├─────────────────────────────────────────┤
│ HTTP Headers                            │
│ Host: www.contoso.com                   │ ← AppGW uses for routing
│ User-Agent: Mozilla/5.0                 │ ← AppGW can inspect
│ Cookie: session=abc123                  │ ← AppGW can read/inject
│ Content-Type: application/json          │ ← AppGW aware of type
│ Authorization: Bearer token123          │ ← WAF can inspect
├─────────────────────────────────────────┤
│ HTTP Body                               │
│ {"username": "admin",                   │ ← WAF inspects this
│  "password": "password"}                │ ← Checks for attacks
└─────────────────────────────────────────┘

Application Gateway Processing:

1. Parse Request Line:
   Method: GET ✓ (WAF checks method)
   Path: /api/users/123 → Matches /api/* rule
   HTTP Version: 1.1 ✓

2. Parse Headers:
   Host: www.contoso.com → Multi-site routing
   Cookie: session=abc123 → Session affinity check
   
3. URL-Based Routing Decision:
   Path /api/* → Route to APIBackendPool
   
4. Header Manipulation:
   Add: X-Forwarded-For: 1.2.3.4
   Add: X-Forwarded-Proto: https
   
5. WAF Content Inspection:
   Check request against OWASP rules
   Scan for SQL injection, XSS, etc.
   
6. Forward to Backend:
   Modified HTTP request sent

Compare to Layer 4 Load Balancer:
Load Balancer sees:
- IP: 1.2.3.4
- Port: 443
- Protocol: TCP
- That's IT!

Application Gateway sees:
- HTTP Method: GET
- URL: /api/users/123
- Host: www.contoso.com
- All headers and cookies
- Request body content
- Can make intelligent decisions!
```

**HTTP Response Processing**:

```
Backend Response:
┌─────────────────────────────────────────┐
│ HTTP/1.1 200 OK                         │
│ Content-Type: application/json          │
│ Server: nginx/1.18.0                    │ ← AppGW can remove
│ Set-Cookie: auth=xyz                    │ ← AppGW can modify
│                                         │
│ {"users": [{"id": 123, "name": "John"}]}│
└─────────────────────────────────────────┘

Application Gateway Processing:

1. Receive response from backend
2. Parse HTTP response
3. Apply response rewrite rules:
   - Remove: Server header (security)
   - Add: Strict-Transport-Security
   - Add: X-Content-Type-Options
   - Modify: Set-Cookie (if needed)
   
4. Modified Response to Client:
┌─────────────────────────────────────────┐
│ HTTP/1.1 200 OK                         │
│ Content-Type: application/json          │
│ Strict-Transport-Security: max-age=...  │ ← Added
│ X-Content-Type-Options: nosniff         │ ← Added
│ Set-Cookie: auth=xyz; Secure; HttpOnly  │ ← Modified
│                                         │
│ {"users": [{"id": 123, "name": "John"}]}│
└─────────────────────────────────────────┘

Layer 7 Capabilities:
✓ Read HTTP status code
✓ Inspect response headers
✓ Can buffer and inspect body
✓ Add security headers
✓ Remove server identity headers
✓ Modify cookies
```

**HTTP/2 Support**:

```
HTTP/2 Features:

Client → AppGW:
Protocol: HTTP/2
- Multiplexing (multiple requests on single connection)
- Header compression (HPACK)
- Server push
- Binary protocol

AppGW → Backend:
Protocol: HTTP/1.1 or HTTP/2
- AppGW translates if needed
- Can use HTTP/1.1 to backend (simpler)
- Or HTTP/2 if backend supports

Benefit:
Client gets HTTP/2 performance
Backend can use simpler HTTP/1.1
Best of both worlds!

Example:
Client sends (HTTP/2):
:method = GET
:path = /api/data
:authority = www.contoso.com
:scheme = https

AppGW translates to backend (HTTP/1.1):
GET /api/data HTTP/1.1
Host: www.contoso.com
```

### Layer 3: Transport Layer (TCP)

**Connection Termination**:

```
TCP Three-Way Handshake (Client → AppGW):

Step 1: Client → AppGW
┌─────────────────────────────────────────┐
│ TCP Header                              │
│ Src Port: 52000 (client ephemeral)     │
│ Dst Port: 443 (HTTPS)                   │
│ Flags: SYN                              │
│ Seq: 1000                               │
└─────────────────────────────────────────┘

Step 2: AppGW → Client
┌─────────────────────────────────────────┐
│ TCP Header                              │
│ Src Port: 443                           │
│ Dst Port: 52000                         │
│ Flags: SYN, ACK                         │
│ Seq: 5000, Ack: 1001                    │
└─────────────────────────────────────────┘

Step 3: Client → AppGW
┌─────────────────────────────────────────┐
│ TCP Header                              │
│ Flags: ACK                              │
│ Ack: 5001                               │
└─────────────────────────────────────────┘

Connection 1: Client ←→ AppGW ESTABLISHED

Then Separately:

TCP Three-Way Handshake (AppGW → Backend):

Step 1: AppGW → Backend
┌─────────────────────────────────────────┐
│ TCP Header                              │
│ Src Port: 50000 (AppGW ephemeral)      │
│ Dst Port: 80 (backend HTTP)             │
│ Flags: SYN                              │
│ Seq: 2000                               │
└─────────────────────────────────────────┘

Step 2: Backend → AppGW
┌─────────────────────────────────────────┐
│ Flags: SYN, ACK                         │
│ Seq: 6000, Ack: 2001                    │
└─────────────────────────────────────────┘

Step 3: AppGW → Backend
┌─────────────────────────────────────────┐
│ Flags: ACK                              │
│ Ack: 6001                               │
└─────────────────────────────────────────┘

Connection 2: AppGW ←→ Backend ESTABLISHED

Two Independent TCP Connections:
Connection 1: Client ←→ AppGW (ports: 52000 ← 443)
Connection 2: AppGW ←→ Backend (ports: 50000 → 80)

AppGW manages both:
- Accepts on Connection 1
- Initiates Connection 2
- Proxies data between them
- Can optimize each independently
```

**Connection Pooling**:

```
Client Connections → AppGW → Backend Connections

Traditional (No Pooling):
Client 1 → AppGW → Backend Conn 1
Client 2 → AppGW → Backend Conn 2
Client 3 → AppGW → Backend Conn 3
100 clients = 100 backend connections

With Connection Pooling (AppGW does this):
Client 1 ┐
Client 2 ├→ AppGW ─→ Backend Conn 1 (reused)
Client 3 ┘

Client 4 ┐
Client 5 ├→ AppGW ─→ Backend Conn 2 (reused)
Client 6 ┘

100 clients = ~5-10 backend connections

How It Works:
1. Client 1 request arrives, AppGW checks pool
2. No available backend connection, creates new
3. Sends Client 1 request, gets response
4. Keeps backend connection open (HTTP keep-alive)
5. Client 2 request arrives
6. Backend connection available in pool, reuses it
7. Sends Client 2 request on same TCP connection
8. Much more efficient!

Benefits:
✓ Fewer backend TCP connections
✓ Reduced backend CPU (fewer handshakes)
✓ Better resource utilization
✓ Faster request processing (no connection setup)
```

### Layer 2: Internet Layer (IP)

**IP-Level Operations**:

```
Client Request Packet:
┌─────────────────────────────────────────┐
│ IP Header                               │
│ Src IP: 1.2.3.4 (client)                │
│ Dst IP: 20.50.60.70 (AppGW public IP)   │
│ Protocol: TCP (6)                       │
│ TTL: 50                                 │
└─────────────────────────────────────────┘

AppGW to Backend Packet:
┌─────────────────────────────────────────┐
│ IP Header                               │
│ Src IP: 10.0.1.4 (AppGW instance)       │
│ Dst IP: 10.0.2.5 (backend server)       │
│ Protocol: TCP (6)                       │
│ TTL: 64                                 │
└─────────────────────────────────────────┘

Source IP Preservation:
Backend sees AppGW IP, not client IP!
BUT: Client IP preserved in HTTP header
X-Forwarded-For: 1.2.3.4

Backend can parse HTTP headers to get:
- Original client IP: X-Forwarded-For
- Original protocol: X-Forwarded-Proto (https)
- Original host: X-Original-Host
```

**FQDN Backend Resolution**:

```
Backend Pool Configuration:
Backend: api.example.com (FQDN, not IP)

AppGW Behavior:

1. Configuration loaded:
   Backend: api.example.com
   
2. DNS Resolution:
   AppGW → DNS: Query api.example.com
   DNS → AppGW: A record: 93.184.216.34
   
3. Connection:
   AppGW connects to: 93.184.216.34
   
4. HTTP Request:
   AppGW sends HTTP with Host header:
   GET /data HTTP/1.1
   Host: api.example.com  ← Important!
   
5. Periodic Re-resolution:
   Every 30 seconds (configurable):
   AppGW → DNS: Query api.example.com
   DNS → AppGW: A record: 93.184.216.35 (IP changed!)
   
6. Adaptation:
   New connections go to: 93.184.216.35
   Existing connections drain gracefully
   
Benefits:
✓ Backends can change IPs
✓ Azure App Service works (dynamic IPs)
✓ DNS-based failover
✓ Geographic routing via DNS
```

-----

## Layer 6: Request Processing Flow

### Complete HTTP Request Journey

Let’s trace a complete request through Application Gateway:

#### Scenario Setup

```
Client: 1.2.3.4 (Internet user)
    ↓
Application Gateway: "ProductionAppGW"
  - Public IP: 20.50.60.70
  - Instance 1: 10.0.1.4
  - Instance 2: 10.0.1.5
  - Listener: www.contoso.com:443 (HTTPS)
  - WAF: Enabled (Prevention mode)
    ↓
Backend Pool: "APIServers"
  - Server1: 10.0.2.4:8080 (Healthy)
  - Server2: 10.0.2.5:8080 (Healthy)
  - Server3: 10.0.2.6:8080 (Unhealthy)
```

### Inbound Request Flow

**Step 1: Client Initiates HTTPS Request**

```
User Action:
Browser navigates to: https://www.contoso.com/api/users

DNS Resolution:
1. Browser queries DNS: www.contoso.com
2. DNS returns: 20.50.60.70 (AppGW public IP)
3. Browser initiates connection to 20.50.60.70:443

TCP Connection (Client → AppGW):
SYN → 
   ← SYN-ACK
ACK →
Connection established
```

**Step 2: TLS Handshake**

```
Client → AppGW:
ClientHello {
  TLS Version: 1.3
  SNI: www.contoso.com
  Cipher Suites: [
    TLS_AES_256_GCM_SHA384,
    TLS_CHACHA20_POLY1305_SHA256,
    ...
  ]
}

AppGW Processing:
1. Reads SNI: "www.contoso.com"
2. Matches to listener with hostname: www.contoso.com
3. Loads SSL certificate: CN=www.contoso.com
4. Selects cipher suite: TLS_AES_256_GCM_SHA384

AppGW → Client:
ServerHello {
  TLS Version: 1.3
  Cipher: TLS_AES_256_GCM_SHA384
  Certificate Chain: [
    Certificate: CN=www.contoso.com
    Intermediate CA cert
    Root CA cert
  ]
}

Certificate Verification:
Client verifies:
1. Certificate matches hostname ✓
2. Certificate not expired ✓
3. Certificate signed by trusted CA ✓
4. Certificate chain valid ✓

TLS Handshake Complete:
- Encrypted channel established
- All further data encrypted with TLS 1.3
```

**Step 3: Client Sends HTTP Request**

```
Encrypted over TLS:
┌─────────────────────────────────────────┐
│ GET /api/users HTTP/1.1                 │
│ Host: www.contoso.com                   │
│ User-Agent: Mozilla/5.0 (Windows NT 10) │
│ Accept: application/json                │
│ Cookie: sessionId=abc123def456          │
│ Authorization: Bearer jwt_token_here    │
└─────────────────────────────────────────┘

Packet on Network:
[Encrypted TLS Data]
- Observer can see: Source IP, Dest IP, Port 443
- Observer CANNOT see: URL, headers, body
```

**Step 4: Application Gateway Receives and Decrypts**

```
AppGW Instance 1 (10.0.1.4) receives:

1. TLS Decryption:
   - Uses private key
   - Decrypts TLS payload
   - Extracts HTTP request

2. HTTP Parsing:
   ┌─────────────────────────────────────┐
   │ Parsed HTTP Request:                │
   │ ├─ Method: GET                      │
   │ ├─ Path: /api/users                 │
   │ ├─ HTTP Version: 1.1                │
   │ ├─ Headers:                         │
   │ │  ├─ Host: www.contoso.com         │
   │ │  ├─ User-Agent: Mozilla/5.0       │
   │ │  ├─ Accept: application/json      │
   │ │  ├─ Cookie: sessionId=abc123...   │
   │ │  └─ Authorization: Bearer jwt...  │
   │ └─ Body: (none for GET)             │
   └─────────────────────────────────────┘

3. Extract Client Information:
   Client IP: 1.2.3.4
   Client Port: 52000
   TLS Version: 1.3
   Cipher: TLS_AES_256_GCM_SHA384
```

**Step 5: WAF Inspection**

```
WAF Engine (ModSecurity) Analysis:

Phase 1: Request Headers Inspection
┌─────────────────────────────────────────┐
│ Check User-Agent:                       │
│ "Mozilla/5.0" ✓ (not a known bad bot)  │
│                                         │
│ Check for HTTP Protocol Violations:    │
│ HTTP/1.1 ✓ (valid)                      │
│ Host header present ✓                   │
│                                         │
│ Anomaly Score: 0                        │
└─────────────────────────────────────────┘

Phase 2: Request URI Inspection
┌─────────────────────────────────────────┐
│ URI: /api/users                         │
│                                         │
│ Check for Path Traversal:              │
│ Pattern: \.\./ or \.\.\\               │
│ Match: None ✓                           │
│                                         │
│ Check for SQL Injection in URI:        │
│ Pattern: ' OR 1=1, UNION SELECT, etc.  │
│ Match: None ✓                           │
│                                         │
│ Check for XSS in URI:                   │
│ Pattern: <script>, onerror=, etc.      │
│ Match: None ✓                           │
│                                         │
│ Anomaly Score: 0                        │
└─────────────────────────────────────────┘

Phase 3: Request Headers Deep Inspection
┌─────────────────────────────────────────┐
│ Authorization Header:                   │
│ Value: Bearer jwt_token_here            │
│ Check: JWT format valid ✓               │
│                                         │
│ Cookie Header:                          │
│ Value: sessionId=abc123def456           │
│ Check: No malicious patterns ✓          │
│                                         │
│ Anomaly Score: 0                        │
└─────────────────────────────────────────┘

Phase 4: Request Body Inspection
(No body for GET request, skipped)

WAF Decision:
┌─────────────────────────────────────────┐
│ Total Anomaly Score: 0                  │
│ Threshold: 5                            │
│ Decision: ALLOW                         │
│                                         │
│ Action: Forward request to backend      │
└─────────────────────────────────────────┘

If malicious request (for comparison):
GET /api/users?id=' OR '1'='1 HTTP/1.1

WAF would detect:
Pattern: ' OR '1'='1 (SQL Injection)
Rule: 942100
Anomaly Score: +5
Total Score: 5 (>= threshold)
Decision: BLOCK
Action: Return 403 Forbidden
```

**Step 6: Listener and Rule Matching**

```
Listener Matching:
┌─────────────────────────────────────────┐
│ Incoming Request:                       │
│ - Frontend IP: 20.50.60.70              │
│ - Port: 443                             │
│ - SNI/Host: www.contoso.com             │
│                                         │
│ Available Listeners:                    │
│ Listener 1:                             │
│   Name: "ContosoHTTPSListener"          │
│   Port: 443 ✓                           │
│   Hostname: www.contoso.com ✓           │
│   Protocol: HTTPS ✓                     │
│   → MATCH!                              │
│                                         │
│ Listener 2:                             │
│   Name: "FabrikamHTTPSListener"         │
│   Port: 443 ✓                           │
│   Hostname: www.fabrikam.com ✗          │
│   → No match                            │
│                                         │
│ Selected: ContosoHTTPSListener          │
└─────────────────────────────────────────┘

Routing Rule Selection:
┌─────────────────────────────────────────┐
│ Listener: ContosoHTTPSListener          │
│ Associated Rule: "PathBasedRoutingRule" │
│ Rule Type: Path-based routing           │
│ URL Path Map: "ContosoPathMap"          │
└─────────────────────────────────────────┘

Path Matching:
┌─────────────────────────────────────────┐
│ Request Path: /api/users                │
│                                         │
│ Path Rules:                             │
│ /images/* → ImageServersPool ✗          │
│ /api/* → APIServersPool ✓ MATCH!       │
│ /* → DefaultPool (not evaluated)        │
│                                         │
│ Selected Backend Pool: APIServersPool   │
│ Selected Backend Settings: APISettings  │
└─────────────────────────────────────────┘
```

**Step 7: Backend Health Check and Selection**

```
Backend Pool: APIServersPool
┌─────────────────────────────────────────┐
│ Backend Health Status:                  │
│                                         │
│ Server1: 10.0.2.4:8080                  │
│ Status: Healthy ✓                       │
│ Last Probe: 5 seconds ago (200 OK)     │
│                                         │
│ Server2: 10.0.2.5:8080                  │
│ Status: Healthy ✓                       │
│ Last Probe: 8 seconds ago (200 OK)     │
│                                         │
│ Server3: 10.0.2.6:8080                  │
│ Status: Unhealthy ✗                     │
│ Last Probe: 10 seconds ago (Timeout)   │
│                                         │
│ Available Backends: [Server1, Server2]  │
└─────────────────────────────────────────┘

Backend Selection (Round Robin):
Previous request went to: Server1
Next request goes to: Server2
Selected: 10.0.2.5:8080

Cookie-Based Affinity Check:
Cookie: sessionId=abc123def456
Affinity Cookie: ApplicationGatewayAffinity=server1-xyz
Previous Backend: Server1 (10.0.2.4)
Override Selection: Use Server1 instead!
Final Selection: 10.0.2.4:8080
```

**Step 8: Request Transformation and Rewrite**

```
Original Request (from client):
┌─────────────────────────────────────────┐
│ GET /api/users HTTP/1.1                 │
│ Host: www.contoso.com                   │
│ User-Agent: Mozilla/5.0 (Windows NT 10) │
│ Accept: application/json                │
│ Cookie: sessionId=abc123def456          │
│ Authorization: Bearer jwt_token_here    │
└─────────────────────────────────────────┘

Apply Backend HTTP Settings:
- Protocol: HTTP (not HTTPS)
- Port: 8080
- Add standard headers

Apply Rewrite Rules:
1. Add X-Forwarded-For header
2. Add X-Forwarded-Proto header
3. Add X-Original-Host header
4. Add custom tracking header

Transformed Request (to backend):
┌─────────────────────────────────────────┐
│ GET /api/users HTTP/1.1                 │
│ Host: www.contoso.com                   │
│ User-Agent: Mozilla/5.0 (Windows NT 10) │
│ Accept: application/json                │
│ Cookie: sessionId=abc123def456          │
│ Authorization: Bearer jwt_token_here    │
│ X-Forwarded-For: 1.2.3.4 ←New           │
│ X-Forwarded-Proto: https ←New           │
│ X-Original-Host: www.contoso.com ←New   │
│ X-Request-Id: req-12345-67890 ←New      │
│ Connection: keep-alive                  │
└─────────────────────────────────────────┘
```

**Step 9: Backend Connection**

```
AppGW checks connection pool:
┌─────────────────────────────────────────┐
│ Connection Pool to 10.0.2.4:8080:       │
│ Connection 1: Active (in use)           │
│ Connection 2: Idle (available) ✓        │
│ Connection 3: Active (in use)           │
└─────────────────────────────────────────┘

Reuse Connection 2:
- No new TCP handshake needed
- Connection already established
- Faster request delivery

If no idle connection:
1. Create new TCP connection
   SYN → Backend
   ← SYN-ACK
   ACK →
   
2. If backend is HTTPS, TLS handshake
   
3. Add connection to pool

Send Request:
AppGW (10.0.1.4) → Backend (10.0.2.4:8080)
[HTTP request over TCP connection]
```

**Step 10: Backend Processing**

```
Backend Server1 (10.0.2.4:8080):

1. Receives HTTP request
   
2. Application (Node.js/Java/Python) processes:
   - Authenticates JWT token
   - Queries database for users
   - Builds JSON response
   
3. Generates HTTP response:
   ┌─────────────────────────────────────┐
   │ HTTP/1.1 200 OK                     │
   │ Content-Type: application/json      │
   │ Server: nginx/1.18.0                │
   │ X-Backend-Server: server1           │
   │ Content-Length: 524                 │
   │                                     │
   │ {                                   │
   │   "users": [                        │
   │     {"id": 1, "name": "Alice"},     │
   │     {"id": 2, "name": "Bob"}        │
   │   ]                                 │
   │ }                                   │
   └─────────────────────────────────────┘
   
4. Sends response back to AppGW
```

**Step 11: Response Processing**

```
AppGW Receives Response:

1. Parse HTTP response
   
2. Apply Response Rewrite Rules:
   - Remove "Server" header (security)
   - Add security headers
   - Add custom headers
   
Modified Response:
┌─────────────────────────────────────────┐
│ HTTP/1.1 200 OK                         │
│ Content-Type: application/json          │
│ Strict-Transport-Security: max-age=...  │ ←Added
│ X-Content-Type-Options: nosniff         │ ←Added
│ X-Frame-Options: DENY                   │ ←Added
│ X-XSS-Protection: 1; mode=block         │ ←Added
│ X-Backend-Server: server1               │
│ Content-Length: 524                     │
│                                         │
│ {                                       │
│   "users": [                            │
│     {"id": 1, "name": "Alice"},         │
│     {"id": 2, "name": "Bob"}            │
│   ]                                     │
│ }                                       │
└─────────────────────────────────────────┘

3. Buffer response (if needed)
   
4. Prepare to send to client
```

**Step 12: Response to Client**

```
AppGW → Client (over TLS):

1. Encrypt response with TLS 1.3
   
2. Send encrypted data:
   [Encrypted TLS Data containing HTTP response]
   
3. Client receives and decrypts
   
4. Browser renders JSON data

Connection State:
- Connection kept alive (HTTP keep-alive)
- Client can send more requests on same connection
- AppGW can reuse backend connection
- Efficient for multiple requests

Total Time:
- TLS handshake: ~50ms
- WAF inspection: ~5ms
- Routing decision: <1ms
- Backend processing: ~100ms (depends on query)
- Response time: ~5ms
- Total: ~160ms
```

### Error Scenarios

**Scenario: All Backends Unhealthy**

```
Request: GET /api/users HTTP/1.1

Backend Health:
- Server1: Unhealthy ✗
- Server2: Unhealthy ✗
- Server3: Unhealthy ✗

AppGW Decision:
No healthy backends available

Response to Client:
HTTP/1.1 502 Bad Gateway
Content-Type: text/html

<!DOCTYPE html>
<html>
<body>
<h1>502 Bad Gateway</h1>
<p>No healthy backend servers available.</p>
</body>
</html>

Custom Error Page (if configured):
Redirect to: https://storage.../error502.html
```

**Scenario: Backend Timeout**

```
Request: GET /api/slow-query HTTP/1.1

Backend Processing:
Server1 processes request...
Takes 35 seconds (slow database query)

AppGW Backend Settings:
Request Timeout: 30 seconds

At T+30 seconds:
AppGW times out waiting for backend
Closes backend connection

Response to Client:
HTTP/1.1 504 Gateway Timeout
Content-Type: text/html

<!DOCTYPE html>
<html>
<body>
<h1>504 Gateway Timeout</h1>
<p>The server took too long to respond.</p>
</body>
</html>

Backend continues processing (wasted work)
Backend eventually sends response (discarded by AppGW)
```

**Scenario: WAF Blocks Request**

```
Malicious Request:
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin&password=' OR '1'='1

WAF Detection:
Pattern: ' OR '1'='1 (SQL Injection)
Rule: 942100
Severity: Critical
Anomaly Score: 5
Decision: BLOCK

Response to Client (immediate):
HTTP/1.1 403 Forbidden
Content-Type: text/html

<!DOCTYPE html>
<html>
<body>
<h1>403 Forbidden</h1>
<p>Your request has been blocked by security policy.</p>
<p>Incident ID: abc123-def456</p>
</body>
</html>

Backend Impact:
✓ Backend never sees request
✓ Database not queried
✓ No SQL injection possible
✓ Attack logged for analysis

WAF Log:
{
  "timestamp": "2025-11-08T12:30:45Z",
  "action": "Blocked",
  "ruleId": "942100",
  "severity": "Critical",
  "clientIp": "1.2.3.4",
  "requestUri": "/login",
  "matchedData": "' OR '1'='1"
}
```

-----

## Layer 7: Physical Implementation

### Datacenter Deployment

**Application Gateway Physical Architecture**:

```
Azure Region: West Europe

┌────────────────────────────────────────────┐
│ Availability Zone 1                         │
│ ├─ AppGW Instance 1 (VM)                   │
│ │  IP: 10.0.1.4                             │
│ │  Role: Active, serving traffic            │
│ └─ Resources: vCPU, RAM, Storage            │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ Availability Zone 2                         │
│ ├─ AppGW Instance 2 (VM)                   │
│ │  IP: 10.0.1.5                             │
│ │  Role: Active, serving traffic            │
│ └─ Resources: vCPU, RAM, Storage            │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ Availability Zone 3                         │
│ ├─ AppGW Instance 3 (VM)                   │
│ │  IP: 10.0.1.6                             │
│ │  Role: Active, serving traffic            │
│ └─ Resources: vCPU, RAM, Storage            │
└────────────────────────────────────────────┘

Frontend Distribution:
┌────────────────────────────────────────────┐
│ Azure Internal Load Balancer               │
│ - Public IP: 20.50.60.70                   │
│ - Distributes to AppGW instances           │
│ - Health monitors all instances            │
│ - ECMP routing                             │
└────────────────────────────────────────────┘
```

### Instance Specifications

**Application Gateway Instance (V2 SKU)**:

```
Virtual Machine Specifications:

OS: Azure-managed Linux
VM Size: Varies by load and autoscaling

Small Workload:
- vCPU: 2 cores
- RAM: 4 GB
- Network: 2x NICs (frontend/backend)
- Storage: 30 GB SSD

Medium Workload:
- vCPU: 4 cores
- RAM: 8 GB
- Network: 2x NICs
- Storage: 50 GB SSD

Large Workload:
- vCPU: 8+ cores
- RAM: 16+ GB
- Network: 2x NICs (25 Gbps+)
- Storage: 100 GB SSD

Software Components per Instance:
├─ NGINX (reverse proxy)
├─ ModSecurity (WAF engine)
├─ OpenSSL (TLS/SSL)
├─ Health monitoring daemon
├─ Metrics collection agent
├─ Azure management agent
└─ Logging service
```

### Networking Architecture

**Network Flow**:

```
Internet (Client)
    ↓
┌────────────────────────────────────────┐
│ Azure Edge Network                      │
│ - DDoS Protection                       │
│ - Public IP endpoint                    │
│ - 20.50.60.70                           │
└────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────┐
│ Internal Load Balancer                  │
│ - Distributes to AppGW instances        │
│ - 5-tuple hash                          │
└────────────────────────────────────────┘
    ↓   ↓   ↓
┌─────┬─────┬─────┐
│ AG1 │ AG2 │ AG3 │ Application Gateway Instances
└─────┴─────┴─────┘ in AppGW Subnet (10.0.1.0/24)
    ↓   ↓   ↓
┌────────────────────────────────────────┐
│ Virtual Network Routing                 │
│ - Routes to backend subnets             │
└────────────────────────────────────────┘
    ↓   ↓   ↓
┌─────┬─────┬─────┐
│ BE1 │ BE2 │ BE3 │ Backend Servers
└─────┴─────┴─────┘ in Backend Subnet (10.0.2.0/24)
```

### Storage and State Management

**Configuration Storage**:

```
Configuration Components:

┌────────────────────────────────────────┐
│ Azure Resource Manager Database         │
│ - Stores AppGW configuration            │
│ - Geo-replicated                        │
│ - Version controlled                    │
└────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────┐
│ AppGW Control Plane                     │
│ - Processes configuration               │
│ - Generates instance configs            │
│ - Distributes to instances              │
└────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────┐
│ AppGW Instances                         │
│ - Load configuration locally            │
│ - NGINX config files                    │
│ - SSL certificates (encrypted)          │
│ - WAF rules                             │
└────────────────────────────────────────┘

Configuration Synchronization:
1. You update rule in portal
2. ARM validates and stores
3. Control plane notified
4. Control plane generates configs
5. Configs pushed to all instances
6. Instances reload gracefully
7. New config active

Time: ~5-10 seconds
```

**Session State**:

```
Session Affinity (Cookie-Based):

Cookie Stored:
- Location: Client browser
- Name: ApplicationGatewayAffinity
- Value: Encoded backend server ID
- Properties: HttpOnly, Secure, SameSite

NOT Stored by AppGW:
✗ Application session data
✗ User authentication state
✗ Shopping cart contents

Application State Storage:
Must use external storage:
├─ Azure Redis Cache
├─ Azure Cosmos DB
├─ Azure SQL Database
├─ Azure Storage (Table/Blob)
└─ Any distributed cache

Why?
- AppGW instances are stateless
- Instances can be replaced anytime
- Autoscaling adds/removes instances
- Must not rely on instance state
```

### Monitoring and Diagnostics

**Telemetry Collection**:

```
Metrics Collected (per instance):

Network Metrics:
├─ Total Requests
├─ Failed Requests
├─ Response Status Distribution (2xx, 3xx, 4xx, 5xx)
├─ Throughput (bytes/sec)
├─ Connection Count
└─ Current Connections

Performance Metrics:
├─ Backend Response Time
├─ Application Gateway Total Time
├─ Healthy/Unhealthy Backend Count
└─ Backend Connect Time

WAF Metrics (if enabled):
├─ Blocked Requests
├─ Matched Rules
├─ Blocked Request Count by Rule
└─ Top Blocked IPs

Collection Flow:
┌────────────────────────────────────┐
│ AppGW Instance                      │
│ ├─ Metrics Agent                   │
│ │  - Collects every 60 seconds    │
│ │  - Aggregates locally            │
│ └─ Sends to Azure Monitor          │
└────────────────────────────────────┘
    ↓
┌────────────────────────────────────┐
│ Azure Monitor                       │
│ ├─ Ingests metrics                 │
│ ├─ Stores time-series data         │
│ └─ Provides query interface        │
└────────────────────────────────────┘
    ↓
┌────────────────────────────────────┐
│ Azure Portal / Dashboards           │
│ - Visualizes metrics                │
│ - Alerts on thresholds              │
└────────────────────────────────────┘
```

**Diagnostics Logging**:

```
Diagnostic Logs:

Access Logs:
{
  "time": "2025-11-08T12:30:45Z",
  "clientIP": "1.2.3.4",
  "clientPort": 52000,
  "httpMethod": "GET",
  "requestUri": "/api/users",
  "httpStatus": 200,
  "userAgent": "Mozilla/5.0...",
  "timeTaken": 0.156,
  "sslEnabled": true,
  "backendServerIP": "10.0.2.4",
  "backendServerPort": 8080,
  "backendResponseTime": 0.120
}

Firewall Logs (WAF):
{
  "time": "2025-11-08T12:31:00Z",
  "clientIP": "5.6.7.8",
  "action": "Blocked",
  "ruleId": "942100",
  "message": "SQL Injection Attack",
  "matchedData": "' OR '1'='1",
  "severity": "Critical",
  "requestUri": "/login"
}

Performance Logs:
{
  "time": "2025-11-08T12:30:45Z",
  "instanceId": "appgw-instance-1",
  "healthyBackendCount": 2,
  "unhealthyBackendCount": 1,
  "cpuUtilization": 45.2,
  "memoryUtilization": 62.8
}

Log Destinations:
├─ Storage Account (long-term retention)
├─ Log Analytics (queries and analysis)
├─ Event Hub (streaming to SIEM)
└─ Partner Solutions (Splunk, etc.)
```

### High Availability Implementation

**Zone-Redundant Deployment**:

```
Failure Scenarios and Resilience:

Scenario 1: Single Instance Failure
Initial: 3 instances (Zone 1, 2, 3)
Instance 2 fails (hardware issue)

T+0s: Instance 2 stops responding
T+5s: Health probe failures detected
T+10s: Internal LB marks Instance 2 unhealthy
T+11s: Traffic redistributed to Instance 1, 3
T+15s: Azure detects failure
T+60s: New Instance 2 deployed
T+120s: Instance 2 online, health probes succeed
T+125s: Instance 2 added back to pool

Downtime: 0 seconds (other instances handled traffic)

Scenario 2: Availability Zone Failure
Initial: Instance 1 (Zone 1), Instance 2 (Zone 2), Instance 3 (Zone 3)
Zone 1 fails (datacenter power issue)

T+0s: Zone 1 loses power
T+0s: Instance 1 stops responding
T+5s: Health probes fail
T+10s: Internal LB marks Instance 1 unhealthy
T+11s: All traffic to Instance 2, 3
T+15s: Azure autoscaling triggered (if enabled)
T+60s: New instances added in Zone 2, 3
T+120s: Back to 3+ instances across 2 zones

Capacity Impact:
- 33% reduction temporarily
- Autoscaling compensates
- Performance maintained

Downtime: 0 seconds (zone-redundant design)

Scenario 3: Maintenance/Updates
Azure needs to update instance OS

Graceful update process:
1. Instance 1 marked for update
2. Connection draining enabled (300s)
3. No new connections to Instance 1
4. Existing connections complete
5. After 300s, Instance 1 updated
6. Instance 1 returns to service
7. Repeat for Instance 2, 3

No downtime, zero impact to users
```

-----

## Layer 8: Practical Examples

### Example 1: Simple Web Application with WAF

**Scenario**: Secure a public-facing web application

```
Requirements:
- Public HTTPS website
- SSL termination
- WAF protection against attacks
- 3 web servers in backend
- Health monitoring
```

**Architecture**:

```
Internet (Users)
    ↓
Public IP: 40.50.60.80 (HTTPS)
    ↓
Application Gateway (WAF V2)
  ├─ Frontend: 40.50.60.80:443
  ├─ SSL Certificate: *.contoso.com
  ├─ WAF: Prevention mode, OWASP 3.2
  ├─ Listener: www.contoso.com
  └─ Backend Pool: WebServers
      ├─ Web1: 10.0.2.4:80 (HTTP)
      ├─ Web2: 10.0.2.5:80 (HTTP)
      └─ Web3: 10.0.2.6:80 (HTTP)
```

**Configuration** (simplified JSON):

```json
{
  "name": "WebAppGateway",
  "sku": {
    "name": "WAF_v2",
    "tier": "WAF_v2"
  },
  "autoscaleConfiguration": {
    "minCapacity": 2,
    "maxCapacity": 10
  },
  "webApplicationFirewallConfiguration": {
    "enabled": true,
    "firewallMode": "Prevention",
    "ruleSetType": "OWASP",
    "ruleSetVersion": "3.2"
  },
  "frontendIPConfigurations": [{
    "name": "PublicFrontend",
    "publicIPAddress": {"id": ".../WebPublicIP"}
  }],
  "frontendPorts": [
    {"name": "port_443", "port": 443}
  ],
  "backendAddressPools": [{
    "name": "WebServers",
    "backendAddresses": [
      {"ipAddress": "10.0.2.4"},
      {"ipAddress": "10.0.2.5"},
      {"ipAddress": "10.0.2.6"}
    ]
  }],
  "backendHttpSettingsCollection": [{
    "name": "httpSettings",
    "port": 80,
    "protocol": "Http",
    "cookieBasedAffinity": "Disabled",
    "requestTimeout": 30
  }],
  "httpListeners": [{
    "name": "httpsListener",
    "frontendIPConfiguration": {"id": ".../PublicFrontend"},
    "frontendPort": {"id": ".../port_443"},
    "protocol": "Https",
    "sslCertificate": {"id": ".../contosoWildcard"},
    "hostName": "www.contoso.com"
  }],
  "requestRoutingRules": [{
    "name": "rule1",
    "ruleType": "Basic",
    "httpListener": {"id": ".../httpsListener"},
    "backendAddressPool": {"id": ".../WebServers"},
    "backendHttpSettings": {"id": ".../httpSettings"},
    "priority": 100
  }]
}
```

**Traffic Flow**:

```
User Request:
https://www.contoso.com/products

1. DNS resolves to: 40.50.60.80
2. Client → AppGW HTTPS connection
3. TLS termination at AppGW
4. WAF inspects request
5. AppGW → Backend HTTP connection (10.0.2.4:80)
6. Backend processes and responds
7. AppGW → Client HTTPS response

Security Benefits:
✓ TLS encryption from client to AppGW
✓ WAF blocks SQL injection, XSS, etc.
✓ Backend simplified (no SSL management)
✓ Centralized certificate management
✓ DDoS protection (Azure platform)
```

### Example 2: Microservices with Path-Based Routing

**Scenario**: Route different API paths to different microservices

```
Requirements:
- Single public IP for all services
- /api/auth → Authentication service
- /api/orders → Order service
- /api/inventory → Inventory service
- /api/payment → Payment service
- Everything else → Web frontend
```

**Architecture**:

```
Internet
    ↓
Application Gateway: 40.50.60.90
    ↓
Path-Based Routing:
├─ /api/auth/* → AuthService (10.0.3.0/24)
├─ /api/orders/* → OrderService (10.0.4.0/24)
├─ /api/inventory/* → InventoryService (10.0.5.0/24)
├─ /api/payment/* → PaymentService (10.0.6.0/24)
└─ /* → WebFrontend (10.0.2.0/24)
```

**URL Path Map Configuration**:

```json
{
  "urlPathMaps": [{
    "name": "MicroservicesPathMap",
    "defaultBackendAddressPool": {"id": ".../WebFrontendPool"},
    "defaultBackendHttpSettings": {"id": ".../httpSettings"},
    "pathRules": [
      {
        "name": "AuthRule",
        "paths": ["/api/auth/*"],
        "backendAddressPool": {"id": ".../AuthServicePool"},
        "backendHttpSettings": {"id": ".../authSettings"}
      },
      {
        "name": "OrderRule",
        "paths": ["/api/orders/*"],
        "backendAddressPool": {"id": ".../OrderServicePool"},
        "backendHttpSettings": {"id": ".../orderSettings"}
      },
      {
        "name": "InventoryRule",
        "paths": ["/api/inventory/*"],
        "backendAddressPool": {"id": ".../InventoryServicePool"},
        "backendHttpSettings": {"id": ".../inventorySettings"}
      },
      {
        "name": "PaymentRule",
        "paths": ["/api/payment/*"],
        "backendAddressPool": {"id": ".../PaymentServicePool"},
        "backendHttpSettings": {"id": ".../paymentSettings"}
      }
    ]
  }]
}
```

**Request Routing Examples**:

```
Request 1: GET /api/auth/login
→ Matches: /api/auth/*
→ Routed to: AuthService (10.0.3.4)

Request 2: POST /api/orders/12345
→ Matches: /api/orders/*
→ Routed to: OrderService (10.0.4.5)

Request 3: GET /api/inventory/items
→ Matches: /api/inventory/*
→ Routed to: InventoryService (10.0.5.6)

Request 4: POST /api/payment/process
→ Matches: /api/payment/*
→ Routed to: PaymentService (10.0.6.4)

Request 5: GET /about
→ No specific match
→ Routed to: WebFrontend (default, 10.0.2.4)

Request 6: GET /products
→ No specific match
→ Routed to: WebFrontend (default, 10.0.2.5)
```

**Benefits**:

```
✓ Single public IP for all services
✓ Centralized SSL termination
✓ Each microservice scales independently
✓ WAF protection for all services
✓ Path-based segmentation
✓ Easy to add new services (add new path rule)
✓ Backend services use simple HTTP
✓ Simplified certificate management
```

### Example 3: Multi-Site Hosting

**Scenario**: Host multiple domains on single Application Gateway

```
Requirements:
- www.contoso.com
- www.fabrikam.com
- api.adventure-works.com
- Each with own SSL certificate
- Each routing to different backends
```

**Architecture**:

```
Application Gateway: 40.50.60.100

Listener 1:
├─ Hostname: www.contoso.com
├─ Port: 443
├─ Certificate: contoso-cert
└─ Backend: ContosoWebServers

Listener 2:
├─ Hostname: www.fabrikam.com
├─ Port: 443
├─ Certificate: fabrikam-cert
└─ Backend: FabrikamWebServers

Listener 3:
├─ Hostname: api.adventure-works.com
├─ Port: 443
├─ Certificate: adventure-cert
└─ Backend: AdventureAPIServers
```

**Multi-Site Listener Configuration**:

```json
{
  "httpListeners": [
    {
      "name": "ContosoListener",
      "frontendIPConfiguration": {"id": ".../PublicFrontend"},
      "frontendPort": {"id": ".../port_443"},
      "protocol": "Https",
      "hostName": "www.contoso.com",
      "requireServerNameIndication": true,
      "sslCertificate": {"id": ".../contosoCert"}
    },
    {
      "name": "FabrikamListener",
      "frontendIPConfiguration": {"id": ".../PublicFrontend"},
      "frontendPort": {"id": ".../port_443"},
      "protocol": "Https",
      "hostName": "www.fabrikam.com",
      "requireServerNameIndication": true,
      "sslCertificate": {"id": ".../fabrikamCert"}
    },
    {
      "name": "AdventureListener",
      "frontendIPConfiguration": {"id": ".../PublicFrontend"},
      "frontendPort": {"id": ".../port_443"},
      "protocol": "Https",
      "hostName": "api.adventure-works.com",
      "requireServerNameIndication": true,
      "sslCertificate": {"id": ".../adventureCert"}
    }
  ],
  "requestRoutingRules": [
    {
      "name": "ContosoRule",
      "httpListener": {"id": ".../ContosoListener"},
      "backendAddressPool": {"id": ".../ContosoPool"},
      "backendHttpSettings": {"id": ".../contosoSettings"},
      "priority": 100
    },
    {
      "name": "FabrikamRule",
      "httpListener": {"id": ".../FabrikamListener"},
      "backendAddressPool": {"id": ".../FabrikamPool"},
      "backendHttpSettings": {"id": ".../fabrikamSettings"},
      "priority": 200
    },
    {
      "name": "AdventureRule",
      "httpListener": {"id": ".../AdventureListener"},
      "backendAddressPool": {"id": ".../AdventurePool"},
      "backendHttpSettings": {"id": ".../adventureSettings"},
      "priority": 300
    }
  ]
}
```

**DNS Configuration**:

```
DNS Records:
www.contoso.com          A    40.50.60.100
www.fabrikam.com         A    40.50.60.100
api.adventure-works.com  A    40.50.60.100

All three domains point to same IP!
```

**Request Routing with SNI**:

```
Client connects to: www.contoso.com

TLS Handshake:
Client → AppGW:
  ClientHello {
    SNI: "www.contoso.com"  ← Key field!
  }

AppGW Processing:
1. Reads SNI: "www.contoso.com"
2. Matches to ContosoListener
3. Loads contoso-cert
4. Completes TLS handshake

Request Processing:
5. HTTP request received
6. Host header: www.contoso.com
7. ContosoRule matched
8. Routed to ContosoPool

Different client connects to: www.fabrikam.com

TLS Handshake:
Client → AppGW:
  ClientHello {
    SNI: "www.fabrikam.com"
  }

AppGW Processing:
1. Reads SNI: "www.fabrikam.com"
2. Matches to FabrikamListener
3. Loads fabrikam-cert
4. Routes to FabrikamPool

Cost Savings:
- One Application Gateway ($$$)
- Three websites ($ each)
- Single public IP
- Shared infrastructure
```

### Example 4: End-to-End TLS with Azure App Service

**Scenario**: Application Gateway → Azure App Service with end-to-end encryption

```
Requirements:
- Public HTTPS endpoint
- End-to-end TLS to App Service
- App Service custom domain
- Azure-managed certificates
```

**Architecture**:

```
Internet
    ↓
Application Gateway: 40.50.60.110
Frontend: www.contoso.com (TLS 1.3)
    ↓ (HTTPS)
Azure App Service: myapp.azurewebsites.net
Backend: TLS 1.2 with Azure certificate
```

**Backend Configuration for App Service**:

```json
{
  "backendAddressPools": [{
    "name": "AppServicePool",
    "backendAddresses": [{
      "fqdn": "myapp.azurewebsites.net"
    }]
  }],
  "backendHttpSettingsCollection": [{
    "name": "appServiceSettings",
    "port": 443,
    "protocol": "Https",
    "cookieBasedAffinity": "Disabled",
    "pickHostNameFromBackendAddress": true,
    "requestTimeout": 30,
    "probe": {"id": ".../appServiceProbe"}
  }],
  "probes": [{
    "name": "appServiceProbe",
    "protocol": "Https",
    "host": "myapp.azurewebsites.net",
    "path": "/",
    "interval": 30,
    "timeout": 30,
    "unhealthyThreshold": 3
  }]
}
```

**Why `pickHostNameFromBackendAddress: true`?**

```
Without pickHostNameFromBackendAddress:

Request to Backend:
GET / HTTP/1.1
Host: www.contoso.com  ← Frontend hostname
    ↓
App Service receives:
Host: www.contoso.com
App Service responds: 404 Not Found
(App Service doesn't recognize this hostname)

With pickHostNameFromBackendAddress: true

Request to Backend:
GET / HTTP/1.1
Host: myapp.azurewebsites.net  ← Backend hostname
    ↓
App Service receives:
Host: myapp.azurewebsites.net ✓
App Service responds: 200 OK
(App Service recognizes its own hostname)

This is required for:
✓ Azure App Service
✓ Azure Functions
✓ Azure API Management
✓ Any backend expecting specific Host header
```

**TLS Certificate Trust**:

```
App Service Certificate:
- Issued by: Microsoft Azure TLS Issuing CA
- Subject: myapp.azurewebsites.net
- Trusted by: Azure (automatically)

Application Gateway:
- Trusts Azure certificates by default
- No need to upload root certificate
- TLS validation succeeds automatically

If using custom domain on App Service:
1. Upload custom certificate to App Service
2. Upload root/intermediate cert to AppGW
3. AppGW validates backend certificate chain
```

**Complete Request Flow**:

```
1. Client → AppGW (TLS 1.3):
   https://www.contoso.com/api/data

2. AppGW terminates TLS:
   - Decrypts request
   - Parses HTTP

3. AppGW → App Service (TLS 1.2):
   https://myapp.azurewebsites.net/api/data
   - New TLS connection
   - Host header: myapp.azurewebsites.net

4. App Service processes:
   - Receives HTTPS request
   - Decrypts at App Service
   - Processes request
   - Returns encrypted response

5. AppGW → Client:
   - Receives encrypted response
   - Re-encrypts for client
   - Sends to client

End-to-End Encryption:
✓ Client → AppGW: Encrypted (TLS 1.3)
✓ AppGW → App Service: Encrypted (TLS 1.2)
✓ Data encrypted entire journey
✓ Compliance requirements met
```

-----

## Summary: The Complete Picture

### What You Now Understand About Application Gateway

**Layer 1 (Portal)**:

- Create and configure from simple UI
- Abstracts complex reverse proxy system
- Multiple configuration options

**Layer 2 (Components)**:

- Frontend IPs (public/private)
- Listeners (basic/multi-site)
- Backend pools (VMs, IPs, FQDN, App Service)
- Backend HTTP settings
- Routing rules (basic/path-based)
- URL path maps
- Health probes (TCP/HTTP/HTTPS)
- SSL certificates
- Rewrite rules
- WAF configuration
- Autoscaling

**Layer 3 (How It Works)**:

- Two-connection model (client ←→ AppGW ←→ backend)
- HTTP parsing and inspection
- URL-based routing decisions
- Session affinity (cookies)
- Connection multiplexing
- WAF threat detection
- Health-based routing

**Layer 4 (Technology)**:

- NGINX-based reverse proxy
- ModSecurity WAF engine
- OpenSSL for TLS/SSL
- Distributed architecture
- Instance-based deployment
- Configuration synchronization

**Layer 5 (TCP/IP)**:

- Operates at Layer 7 (Application) primarily
- Full HTTP/HTTPS awareness
- Also Layer 3 (Transport) for TCP management
- Connection termination and re-establishment
- Content inspection and routing

**Layer 6 (Request Flow)**:

- Client → TLS → AppGW → Parse → WAF → Route → Backend
- Backend → AppGW → Transform → Client
- Complete request/response transformation

**Layer 7 (Physical)**:

- VM-based instances
- Zone-redundant deployment
- Internal load balancing to instances
- Autoscaling capabilities
- Configuration storage and distribution

### Key Insights

**Application Gateway IS**:
✅ Layer 7 (Application) load balancer
✅ Full HTTP/HTTPS reverse proxy
✅ Content-aware routing platform
✅ Web Application Firewall (WAF V2)
✅ SSL/TLS termination point
✅ Connection multiplexing system
✅ Request/response transformer
✅ Multi-site hosting platform
✅ Autoscaling service (V2)

**Application Gateway is NOT**:
❌ Simple packet forwarder (like Layer 4 LB)
❌ Transparent to backends
❌ Just a load balancer
❌ Free (has per-hour and data processing costs)

### When to Use Application Gateway vs Other Services

```
Use Application Gateway when:
✓ Need Layer 7 load balancing
✓ HTTP/HTTPS applications
✓ URL-based routing required
✓ Multiple websites on one IP
✓ SSL termination needed
✓ WAF protection required
✓ Cookie-based session affinity
✓ Header manipulation needed
✓ End-to-end TLS required

Use Azure Load Balancer when:
✓ Need Layer 4 load balancing
✓ Non-HTTP protocols (TCP/UDP)
✓ Maximum performance
✓ Cost-sensitive
✓ Simple distribution

Use Azure Front Door when:
✓ Global load balancing
✓ Multi-region failover
✓ CDN required
✓ Edge caching
✓ Global WAF

Use Azure Traffic Manager when:
✓ DNS-based routing
✓ Geographic routing
✓ Multi-region applications
✓ Disaster recovery
```

### Best Practices Summary

**1. SKU Selection**:

- Use V2 SKU (not V1)
- Use WAF V2 for internet-facing apps
- Standard V2 for internal apps
- Enable autoscaling

**2. Subnet Planning**:

- Dedicated subnet for Application Gateway
- /24 subnet recommended (256 IPs)
- Minimum /27 (32 IPs)
- Plan for autoscaling growth

**3. SSL/TLS**:

- Use TLS 1.2 minimum
- Use strong cipher suites
- Keep certificates updated
- Use managed certificates when possible
- Consider end-to-end TLS for sensitive data

**4. WAF Configuration**:

- Start in Detection mode
- Monitor and tune rules
- Add exclusions for false positives
- Switch to Prevention mode
- Use custom rules for specific threats
- Monitor WAF logs regularly

**5. Backend Configuration**:

- Use custom health probes
- Create dedicated health endpoints
- Set appropriate timeouts
- Enable connection draining
- Use FQDN for App Service backends

**6. Routing**:

- Use path-based routing for microservices
- Use multi-site for multiple domains
- Organize rules logically
- Document routing decisions

**7. Monitoring**:

- Enable diagnostic logging
- Watch backend health metrics
- Monitor WAF blocked requests
- Set up alerts for failures
- Review access logs

**8. Performance**:

- Enable HTTP/2
- Use connection multiplexing
- Configure appropriate timeouts
- Enable autoscaling
- Monitor capacity units

**9. Security**:

- Always use HTTPS (not HTTP)
- Enable WAF for public apps
- Use security headers (rewrites)
- Remove server identity headers
- Regular security reviews

**10. Cost Optimization**:

- Use autoscaling (pay for what you use)
- Review capacity unit usage
- Consolidate multiple apps
- Choose appropriate SKU
- Monitor data processing costs

-----

## Next Steps

You’ve now completed the Application Gateway deep dive!

**You’ve learned**:

1. **Azure Load Balancer** (Layer 4) - packet forwarding
1. **Application Gateway** (Layer 7) - HTTP routing

**Suggested next topics**:

1. **Azure Front Door** - Global Layer 7 load balancing
1. **Azure Traffic Manager** - DNS-based routing
1. **Azure Virtual Network** - Networking foundation
1. **Azure Firewall** - Network security appliance
1. **Azure VPN Gateway** - Hybrid connectivity

Each builds on these load balancing concepts!

This document follows the same 8-layer pattern to help you understand Azure Application Gateway from the Azure Portal down to the HTTP request processing and physical implementation. Understanding Layer 7 load balancing is critical for designing modern cloud architectures!
