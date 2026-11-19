# Blue-Green Deployment on AWS (EC2 + Application Load Balancer)

Zero-downtime deployment with instant rollback, built on AWS EC2, an Application Load Balancer (ALB), Target Groups and Security Groups.

## What is Blue-Green Deployment?

Two identical environments run side by side:

- **Blue**: current production version (App v1.0)
- **Green**: new version (App v2.0)

Traffic is switched from Blue to Green by changing a single ALB listener rule. If something goes wrong, switching back to Blue is the rollback. No downtime, no redeploy.

## Architecture

```
                    Internet
                       |
                 [ ALB-SG: HTTP 80 from 0.0.0.0/0 ]
                       |
              Application Load Balancer
                (BlueGreen-ALB, HTTP:80)
                       |
          Listener forwards to ONE target group
               /                     \
        [ Blue-TG ]              [ Green-TG ]
             |                        |
      Blue-Server (EC2)        Green-Server (EC2)
       App v1.0 (BLUE)          App v2.0 (GREEN)
      [ EC2-SG: HTTP 80 only from ALB-SG ]
```

## Tech Stack

| Component | Details |
|---|---|
| Cloud | AWS (AWS Academy Learner Lab), region `us-east-1` |
| Compute | EC2, Amazon Linux 2023, t2/t3.micro |
| Web server | Apache httpd (installed via EC2 User Data) |
| Load balancing | Application Load Balancer, 2 Target Groups |
| Networking | Default VPC, 2 Availability Zones (us-east-1a, us-east-1b) |
| Security | 2 Security Groups (ALB-SG, EC2-SG) |

## Setup Steps

1. **Security Groups**
   - `ALB-SG`: inbound HTTP (80) from `0.0.0.0/0`
   - `EC2-SG`: inbound HTTP (80) with source `ALB-SG` (instances are reachable only through the ALB)
2. **EC2 instances** (Amazon Linux 2023, EC2-SG): launched with User Data that installs `httpd` and serves a page
   - `Blue-Server`: shows `App v1.0 - BLUE Environment`
   - `Green-Server`: shows `App v2.0 - GREEN Environment`
3. **Target Groups** (HTTP:80, health check path `/`)
   - `Blue-TG` with Blue-Server
   - `Green-TG` with Green-Server
4. **Application Load Balancer** `BlueGreen-ALB`: internet-facing, IPv4, 2 AZs, `ALB-SG`, listener `HTTP:80` forwarding to `Blue-TG`
5. **Switch to Green**: Listener, Edit, default action `Green-TG`, Save
6. **Rollback**: Listener, Edit, default action `Blue-TG`, Save

### Sample User Data (Blue)

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd
cat > /var/www/html/index.html <<'EOF'
<html>
  <body style="background-color:blue; color:white; font-family:serif;">
    <h1>App v1.0 - BLUE Environment</h1>
  </body>
</html>
EOF
```

For Green, change the colour to `green` and the heading to `App v2.0 - GREEN Environment`.

## Results

| Stage | Screenshot |
|---|---|
| Blue live (v1.0) | `screenshots/01-blue.png` |
| Listener switched to Green-TG | `screenshots/02-listener-green.png` |
| Green live (v2.0) | `screenshots/03-green.png` |
| Rollback to Blue | `screenshots/04-rollback-blue.png` |
| Target Groups healthy | `screenshots/05-target-groups-healthy.png` |

The same ALB URL served v1.0, then v2.0 after the listener change, then v1.0 again after rollback.

## Key Learnings

- Traffic switching happens at the load balancer, so the old version stays untouched and ready for rollback.
- A target shows `Unused` until its target group is attached to a listener; it turns `Healthy` once health checks pass.
- Locking EC2-SG to ALB-SG means instances are never directly exposed to the internet.
- Browsers cache pages, so use a hard refresh (`Ctrl + Shift + R`) or Incognito to verify a switch.

## Future Improvements

- Weighted routing (canary release, e.g. 90/10)
- Auto Scaling Groups behind each target group
- HTTPS with ACM
- Route 53 custom domain
- CloudWatch alarms for automatic rollback
- Infrastructure as Code (Terraform / CloudFormation)
- CI/CD pipeline for automated deploys

## Cleanup

To avoid charges, delete in this order: ALB, Target Groups, EC2 instances, Security Groups (`EC2-SG` first, then `ALB-SG`).

## Author

**Amisha Sinha**
GitHub: [@amishasinha18](https://github.com/amishasinha18)
