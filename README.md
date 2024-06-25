# WaveCafe – EFS-Backed, Auto-Scaling Web Cluster on AWS

End-to-end reference implementation of a **highly available web tier** on AWS.

## Overview

This project demonstrates a production-grade setup where:

- **Amazon EFS** provides shared, POSIX-compliant storage for user uploads.
- **ALB + ASG** enable horizontal scaling across multiple Availability Zones.
- **Two provisioning strategies** are included:
  - **AMI-baked image** (fast boots, consistent configuration).
  - **User Data bootstrap** (flexible edits, slower boots).

Built as part of the **FameTech NYC DevOps Scaling Series**. Verifiable on request.

---

## Architecture

- **Internet-facing ALB** → Target Group (HTTP 80).
- **ASG** spans 2+ subnets/AZs, health-checked via `/index.html`.
- Each EC2 instance mounts **EFS Access Point** at `/var/www/html/img` (NFS 2049).
- **Security groups**:
  - `web-sg`: inbound 80/22
  - `efs-sg`: inbound 2049 from `web-sg` only

```

wavecafe-aws-efs-asg-webcluster/
├── 00\_architecture/
│   ├── architecture-diagram.png
│   └── wavecafe-efs-asg.drawio
├── 10\_efs-pro-001/
│   └── README.md
├── 20\_asg-pro-002\_ami/
│   ├── README.md
│   └── screenshots/
├── 21\_asg-pro-002\_userdata/
│   ├── README.md
│   ├── userdata-amznlinux2.sh
│   ├── userdata-ubuntu2204.sh
│   └── screenshots/
├── 30\_validation/
│   ├── alb-healthcheck.md
│   └── test-commands.md
├── 40\_ops/
│   ├── cloudwatch-alarms.md
│   ├── rollback-strategy.md
│   └── runbook.md
└── LICENSE

```

---

## Quick Start

1. **Provision EFS + Access Point**  
   → See `10_efs-pro-001/README.md`.

2. **Choose provisioning strategy**:

   - **AMI-baked** → `20_asg-pro-002_ami/README.md`
   - **User Data** → `21_asg-pro-002_userdata/README.md`

3. **Create ALB + Target Group** and attach ASG.

4. **Verify health checks** and confirm scaling.

5. **Validate shared storage**:  
   Upload `hello.txt` and test across instances.

---

## Validation

```bash
# Verify EFS mount
df -h | grep /var/www/html/img

# Verify ALB connectivity
curl -I http://<ALB-DNS>
curl http://<ALB-DNS>/img/hello.txt
```

---

## Why EFS (vs S3)?

- **EFS** → POSIX semantics, directory behavior, shared file access for CMS/web apps.
- **S3** → Object storage, eventual consistency, different access model.

---

## AMI vs User Data – When to Pick Which

- **AMI-baked**:

  - ✅ Fast boot, deterministic, fewer dependencies.
  - ❌ Requires image rebuild for changes.

- **User Data**:

  - ✅ Easy edits, rapid iteration.
  - ❌ Slower boot, external repo dependency.

---

## Ops Notes

- Monitor: CPU, EFS throughput, ALB 5xx errors.
- Ship `/var/log/userdata-efs.log` → CloudWatch Logs.
- Rollback = create new Launch Template version + ASG Instance Refresh.

---

## License

This project is licensed under the **MIT License** – see the [LICENSE](LICENSE) file for details.
