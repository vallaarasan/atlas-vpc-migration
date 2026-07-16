# atlas-vpc-migration
# Zero-Downtime EC2 Migration into a Custom VPC

A hands-on AWS networking project: designing a VPC from scratch and migrating
a live, production music-streaming server into it with zero downtime and
verified data integrity.

## Why this project

I had a real EC2 instance running in AWS's default VPC (Navidrome, a
self-hosted music server, behind Nginx with Let's Encrypt SSL). Instead of
practicing VPC concepts on a throwaway instance, I rebuilt the network
properly and migrated my actual live service into it — so every step had
real consequences if I got it wrong.

## Architecture

```
                          Internet
                              │
                     ┌────────▼────────┐
                     │ Internet Gateway │
                     │   (atlas-igw)    │
                     └────────┬────────┘
                              │
   VPC: atlas-vpc (10.0.0.0/16, ap-south-1)
   ┌──────────────────────────┼──────────────────────────┐
   │                          │                           │
   │   Route Table            │                           │
   │   0.0.0.0/0 → atlas-igw  │                           │
   │                          │                           │
   │   ┌──────────────────────▼──────────────────────┐    │
   │   │  Public Subnet — atlas-public-subnet          │    │
   │   │  10.0.1.0/24 (ap-south-1a)                    │    │
   │   │                                                │    │
   │   │   ┌────────────────────────────────────────┐  │    │
   │   │   │  EC2: atlas-navidrome                    │  │    │
   │   │   │  Security Group: atlas-navidrome-sg       │  │    │
   │   │   │   - 22  (SSH)   ← My IP                   │  │    │
   │   │   │   - 80  (HTTP)  ← 0.0.0.0/0                │  │    │
   │   │   │   - 443 (HTTPS) ← 0.0.0.0/0                │  │    │
   │   │   │                                            │  │    │
   │   │   │  Nginx (reverse proxy, Let's Encrypt SSL) │  │    │
   │   │   │  Docker → Navidrome                        │  │    │
   │   │   └────────────────────────────────────────┘  │    │
   │   └────────────────────────────────────────────────┘    │
   └──────────────────────────────────────────────────────────┘
```

## What was built, manually (no CloudFormation/Terraform wizard)

- **VPC** — `10.0.0.0/16`, named `atlas-vpc`, isolated from AWS's shared default VPC
- **Public subnet** — `10.0.1.0/24`, carved out of the VPC's address space
- **Internet Gateway** — attached to the VPC, giving the subnet a path to the internet
- **Route table** — `0.0.0.0/0 → atlas-igw`, associated with the public subnet
- **Security group** — scoped inbound rules for SSH (22), HTTP (80), HTTPS (443) only

## Migration approach

Direct "move" of an EC2 instance between VPCs isn't possible in AWS —
instances are bound to the VPC they're launched in. The migration used an
image-based cutover instead:

1. **Snapshot** — created an AMI of the live production instance (`navidrome-migration-backup`), capturing Nginx config, SSL certs, Docker containers, and application data
2. **Launch** — launched a new instance from that AMI directly into `atlas-vpc` / `atlas-public-subnet`, reusing the same key pair and matching security group rules
3. **Verify** — before touching DNS:
   - Confirmed Docker containers and Nginx were running (`docker ps`, `systemctl status nginx`)
   - Tested the app through Nginx using a spoofed `Host` header (`curl -H "Host: <domain>" http://localhost/...`) to confirm virtual-host routing survived the migration
   - Compared file counts and directory sizes (`du -sh`) between old and new instance to confirm no data was created after the snapshot was taken and would be lost
4. **Elastic IP** — allocated and associated an Elastic IP with the new instance so its address wouldn't change on future stop/start cycles
5. **DNS cutover** — repointed the domain's DNS record to the new instance's Elastic IP
6. **Cooldown** — kept the old instance stopped (not terminated) for a buffer period, confirming the new instance was stable under real traffic before decommissioning
7. **Cleanup** — terminated the old instance and deregistered the AMI + underlying snapshot once confident

Total downtime during cutover: effectively zero — the old instance kept
serving traffic until DNS had fully propagated to the new one.

## Lessons learned

- **Region scoping** — VPCs, subnets, and route tables are all region-specific; nothing carries over when you switch AWS regions, including "default" resources, which exist separately per region
- **Route table ≠ automatically applied** — creating a route isn't enough; it has to be explicitly associated with the subnet, or traffic still won't flow
- **Security group source IPs go stale** — dynamic/mobile IPs rotate, so a "My IP" rule can silently start blocking you; worth checking this first whenever a connection times out (vs. being refused outright, which usually points elsewhere)
- **AMI snapshots don't include filesystem changes made after the image was taken** — verify data parity before deleting the source

## Stack

`AWS VPC` `EC2` `AMI` `Route Tables` `Internet Gateway` `Security Groups` `Elastic IP`
