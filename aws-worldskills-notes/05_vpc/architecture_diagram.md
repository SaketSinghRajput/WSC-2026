# VPC Architecture Patterns and Diagrams

## Single-Tier Architecture

Simple public subnet with web servers. Use for learning and simple applications.

```mermaid
graph TB
    Internet[Internet]
    IGW[Internet Gateway]
    
    subgraph "VPC 10.0.0.0/16"
        subgraph "Public Subnet 10.0.1.0/24"
            WEB1["Web Server 1<br/>10.0.1.10"]
            WEB2["Web Server 2<br/>10.0.1.20"]
            WEBSG["Security Group<br/>HTTP/HTTPS/SSH"]
        end
        RT1["Route Table<br/>0.0.0.0/0 → IGW"]
    end
    
    Internet <-->|HTTP/HTTPS| IGW
    IGW --> RT1
    RT1 --> WEB1 & WEB2
    WEB1 & WEB2 --> WEBSG
```

**When to use**:
- Static websites
- Learning labs
- Temporary testing

**Limitations**:
- No database protection (everything public)
- Single AZ (no redundancy)
- Not production-ready

## Two-Tier Architecture

Public subnet (web) + private subnet (database). Standard for applications.

```mermaid
graph TB
    Internet[Internet]
    IGW[Internet Gateway]
    
    subgraph "VPC 10.0.0.0/16"
        subgraph "Public Subnet 10.0.1.0/24"
            WEB["Web Server<br/>10.0.1.10"]
            WEBSG["Web SG<br/>Port 80/443/22"]
        end
        
        subgraph "Private Subnet 10.0.2.0/24"
            DB["Database<br/>10.0.2.10"]
            DBSG["Database SG<br/>Port 3306"]
        end
        
        PubRT["Public RT<br/>0.0.0.0/0 → IGW"]
        PrivRT["Private RT<br/>0.0.0.0/0 → NAT"]
    end
    
    NAT["NAT Gateway<br/>(Public Subnet)"]
    
    Internet <-->|HTTP/HTTPS| IGW
    IGW --> PubRT --> WEB
    WEB --> WEBSG
    WEBSG -->|Port 3306| DB
    DB --> DBSG
    WEB -->|Outbound| NAT
    NAT --> IGW
    
    DBSG -->|Deny<br/>Internet| PrivRT
```

**When to use**:
- E-commerce sites (web + database)
- Web applications with backend
- Most production deployments

**Security**:
- Database not directly internet-accessible
- Web server accesses database over local route
- Web can reach internet via NAT for updates

## Three-Tier Architecture

Public (load balancer) + private (app servers) + private (database).

```mermaid
graph TB
    Internet[Internet]
    IGW["Internet<br/>Gateway"]
    
    subgraph "VPC 10.0.0.0/16"
        subgraph "Public Subnet 10.0.1.0/24 (AZ-a)"
            ALB["Application<br/>Load Balancer"]
            ALBSG["ALB SG<br/>80/443"]
        end
        
        subgraph "Private Subnet 10.0.2.0/24 (AZ-a)"
            APP1["App Server 1<br/>10.0.2.10"]
            APPSG["App SG<br/>Port 8080"]
        end
        
        subgraph "Private Subnet 10.0.3.0/24 (AZ-b)"
            APP2["App Server 2<br/>10.0.3.10"]
        end
        
        subgraph "Private Subnet 10.0.4.0/24 (AZ-b)"
            DB["Database<br/>10.0.4.10"]
            DBSG["DB SG<br/>3306"]
        end
    end
    
    NAT["NAT Gateway"]
    
    Internet <-->|HTTP/HTTPS| IGW
    IGW --> ALB
    ALB -->|Port 8080| APP1 & APP2
    APP1 & APP2 -->|Port 3306| DB
    APP1 & APP2 -->|Outbound| NAT
    NAT --> IGW
    
    ALBSG --> APPSG --> DBSG
```

**When to use**:
- Enterprise applications
- High-traffic sites
- Multi-AZ for resilience
- Load-balanced services

**Security**:
- ALB only public tier
- App servers hidden behind ALB
- Database isolated
- Each tier has restrictive SG

## High Availability Pattern (Multi-AZ)

Replicate across multiple availability zones.

```mermaid
graph TB
    Internet[Internet]
    IGW["Internet<br/>Gateway"]
    
    subgraph "VPC 10.0.0.0/16"
        subgraph "AZ: us-east-1a"
            PubSN1["Public SN<br/>10.0.1.0/24"]
            PrivSN1["Private SN<br/>10.0.3.0/24"]
            WEB1["Web Srv 1"]
            DB1["Primary DB"]
        end
        
        subgraph "AZ: us-east-1b"
            PubSN2["Public SN<br/>10.0.2.0/24"]
            PrivSN2["Private SN<br/>10.0.4.0/24"]
            WEB2["Web Srv 2"]
            DB2["Standby DB<br/>(Replication)"]
        end
    end
    
    Internet <-->|HTTP| IGW
    IGW --> PubSN1 & PubSN2
    PubSN1 --> WEB1
    PubSN2 --> WEB2
    PrivSN1 --> DB1
    PrivSN2 --> DB2
    DB1 -->|Sync| DB2
```

**When to use**:
- Production deployments
- Resilience to AZ failures
- Regional redundancy requirement

**Benefits**:
- Automatic failover
- Load distribution
- Database replication

## Bastion Host Pattern (Secure SSH Access)

Jump server in public subnet for SSH to private instances.

```mermaid
graph LR
    Admin["Admin<br/>(external)"]
    IGW["Internet<br/>Gateway"]
    
    subgraph "VPC 10.0.0.0/16"
        subgraph "Public Subnet"
            Bastion["Bastion Host<br/>10.0.1.10"]
            BastionSG["Bastion SG<br/>SSH:22"]
        end
        
        subgraph "Private Subnet"
            Private["Private EC2<br/>10.0.2.10"]
            PrivateSG["Private SG<br/>SSH from Bastion"]
        end
    end
    
    Admin -->|SSH Port 22| IGW
    IGW --> Bastion
    Bastion -->|SSH via<br/>local route| Private
    Bastion --> BastionSG
    Private --> PrivateSG
```

**When to use**:
- Accessing private instances without internet access
- Single entry point for security audits
- Privilege separation

**Security**:
- Bastion SG: SSH from admin IP only
- Private SG: SSH from Bastion SG only
- Private instances unreachable from internet

## Component Relationships (Dependency Diagram)

```mermaid
graph TD
    VPC["VPC<br/>(10.0.0.0/16)"]
    
    VPC --> Subnets["Subnets<br/>(10.0.1.0/24, etc.)"]
    Subnets --> RT["Route Tables<br/>(Public, Private)"]
    RT --> IGW["Internet<br/>Gateway"]
    RT --> NAT["NAT Gateway<br/>(in Public SN)"]
    
    Subnets --> NACL["Network ACLs<br/>(Subnet-level)"]
    Subnets --> Instances["EC2 Instances"]
    
    Instances --> SG["Security Groups<br/>(Instance-level)"]
    Instances --> EIP["Elastic IPs<br/>(Static)"]
    
    EIP --> NAT
```

## Traffic Flow: Inbound Internet

Internet traffic to public instance:

```
1. Internet packet (destination: public IP)
2. IGW receives packet
3. Route table: check destination
4. If 0.0.0.0/0 → IGW: route to public subnet
5. NACL: if allow, pass to instance
6. Security Group: if allow, pass to instance
7. Instance receives packet on public NIC
8. Application processes
9. Return path: reverse (IGW → Internet)
```

## Traffic Flow: Outbound Internet (Private)

Private instance reaching internet via NAT:

```
1. EC2 (10.0.2.10) initiates connection (curl example.com)
2. Route table check: destination not in 10.0.0.0/16 local
3. 0.0.0.0/0 → NAT Gateway route matched
4. Traffic sent to NAT Gateway
5. NAT translates source IP to Elastic IP
6. IGW route: 0.0.0.0/0 → IGW
7. IGW sends to internet (source: NAT Elastic IP)
8. Return traffic: Internet → IGW (destination: NAT EIP)
9. NAT: reverse translation (destination: 10.0.2.10)
10. Return packet reaches EC2
```

## Inter-Subnet Communication

Local route (automatic in all subnets):

```
Source: 10.0.1.10 (public subnet)
Destination: 10.0.2.10 (private subnet)

Route table check:
- Destination 10.0.2.10 matches 10.0.0.0/16 (local)
- Target: local (direct subnet-to-subnet via local NIC)
- NACL: allow (if rules permit)
- SG: allow (if rules permit)
- Packet delivered
```

## WorldSkills Competition Architectures

### Scenario 1: Static Website

```mermaid
graph LR
    Users[Users] -->|HTTP| IGW[IGW]
    IGW --> PubSN["Public SN<br/>10.0.1.0/24"]
    PubSN --> S3["S3 Bucket<br/>(website)"]
    
    AWS["AWS Region"]
```

**Setup**: S3 website endpoint, CloudFront optional.

### Scenario 2: Web Application (2-Tier)

```mermaid
graph TB
    Users[Users] -->|HTTPS| ALB["ALB<br/>(Public)"]
    ALB -->|App Traffic| App["EC2 App Server<br/>(Private)"]
    App -->|Query| DB["RDS MySQL<br/>(Private)"]
    App -->|Outbound| NAT["NAT Gateway"]
    NAT --> IGW["IGW"]
```

**Setup**: 1 public ALB, 1 private app EC2, 1 private RDS.

### Scenario 3: Secure Multi-AZ

```mermaid
graph TB
    Users[Users] -->|HTTPS| ALB["ALB<br/>(Multi-AZ)"]
    ALB --> AZa["AZ-a: App Server"]
    ALB --> AZb["AZ-b: App Server"]
    AZa & AZb -->|Query| DB["RDS Primary"]
    DB -->|Sync| DBStandby["RDS Standby<br/>(AZ-b)"]
    AZa & AZb -->|Outbound| NAT1["NAT-a"]
    NAT1 & NAT2["NAT-b"] --> IGW
```

**Setup**: Multi-AZ ALB, replicated app servers, RDS failover.

## Design Principles

1. **Least Privilege**: Allow only necessary traffic.
2. **Defense in Depth**: Layer security (NACL → SG → app firewall).
3. **High Availability**: Multi-AZ deployment.
4. **Separation of Concerns**: Tier isolation (web, app, database).
5. **Cost Optimization**: Minimize NAT, use free tier resources.
6. **Monitoring**: CloudWatch alarms on network metrics.

## Diagram Legend

| Symbol | Meaning |
|--------|---------|
| Rectangle | AWS service or component |
| Arrow → | Traffic or data flow |
| Dashed arrow ⇢ | Optional/secondary flow |
| **Bold text** | Critical components |
| Subgraph box | VPC, subnet, or AZ boundary |

## Next Steps

- [server_lab.md](server_lab.md): Build complete secure VPC from scratch.
