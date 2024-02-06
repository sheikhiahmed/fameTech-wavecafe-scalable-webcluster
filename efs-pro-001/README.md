# EFS-PRO-001 – AWS Elastic File System (EFS) Shared Storage Integration

_FameTech DevOps Storage Series_

---

## Role & Sprint Context

**Role:** DevOps Engineer – FameTech NYC  
**Sprint Goal:** Integrate a shared storage system for the WaveCafe production web cluster to centralize user-uploaded images and prepare the system for Auto Scaling Group deployment.  
**Business Need:** Multiple web servers need to access the same `/var/www/html/img` directory for user-uploaded content. The storage must be scalable, highly available, and accessible across AZs.

---

## Lab Metadata

| Item                     | Details                                              |
| ------------------------ | ---------------------------------------------------- |
| **Lab ID**               | EFS-PRO-001                                          |
| **Environment**          | AWS VPC, EC2 (Amazon Linux 2), EFS                   |
| **Primary AWS Services** | Amazon EFS, EC2, Security Groups, Access Points, AMI |
| **Protocol**             | NFS (Port 2049)                                      |
| **Difficulty**           | Intermediate                                         |
| **Estimated Time**       | 60–90 min                                            |

---

## Pre-Deployment Checklist

- [ ] EC2 instance running Amazon Linux 2 (WaveCafe web server)
- [ ] Security group for EC2 allowing inbound HTTP (80) and SSH (22)
- [ ] IAM role with permissions for EFS
- [ ] AWS CLI configured (optional, for CLI-based setup)
- [ ] EFS mount helper package available (`amazon-efs-utils`)

---

## Implementation Steps

### 1. Create Security Group for EFS

```bash
Name: efs-wave-image
Inbound Rule: NFS (2049) → Source: Web server’s security group
```

**Why:** Restricts EFS access only to EC2 instances in the web server SG.

---

### 2. Create the EFS Filesystem

- Navigate to **EFS → Create file system**.
- Name: `wave-web-image`.
- Performance mode: General Purpose.
- Throughput mode: Bursting.
- Encryption: Enabled.
- Networking: Select VPC, assign `efs-wave-image` SG to all AZs.
- Remove default SG.

---

### 3. Create EFS Access Point

- Go to **Access Points → Create Access Point**.
- Select `wave-web-image` filesystem.
- Set root directory path (optional).
- Note **Access Point ID** for later.

---

### 4. Prepare EC2 Mount Directory

```bash
sudo -i
mkdir -p /tmp/img-backup
mv /var/www/html/img/* /tmp/img-backup/
```

---

### 5. Install EFS Utilities

```bash
sudo yum install -y amazon-efs-utils
```

_For non-Amazon Linux: build from source or use OS package manager._

---

### 6. Configure Auto-Mount in `/etc/fstab`

```bash
# Replace <filesystem-id> and <access-point-id> with your values
<filesystem-id>:/ /var/www/html/images efs _netdev,noresvport,tls,accesspoint=<access-point-id> 0 0
```

Example:

```bash
fs-06528aa1390dd321a:/ /var/www/html/images efs _netdev,noresvport,tls,accesspoint=fsap-0352ada536721d327 0 0
```

---

### 7. Mount and Verify

```bash
sudo mount -fav
df -h
```

Confirm `/var/www/html/img` is now EFS-backed.

---

### 8. Restore Backup Files

```bash
mv /tmp/img-backup/* /var/www/html/images/
```

---

### 9. Create AMI for Auto Scaling

- EC2 → Select instance → **Create Image**
- Name: `wave-web-img-efs`
- Delete old AMI if no longer needed

---

## Validation Checklist

- [ ] `df -h` shows EFS mount at `/var/www/html/img`
- [ ] Files uploaded via web app appear instantly on all EC2s in cluster
- [ ] NFS port 2049 is restricted to required SGs
- [ ] AMI successfully created and ready for ASG

---

## Common Issues & Fixes

| Issue                      | Likely Cause             | Fix                                 |
| -------------------------- | ------------------------ | ----------------------------------- |
| Mount command fails        | SG rules missing for NFS | Open port 2049 between SGs          |
| Files missing after reboot | No `/etc/fstab` entry    | Add correct mount line              |
| Slow response              | Max I/O mode latency     | Use General Purpose for low-latency |

---

## Interview Q\&A

**Q:** How is EFS different from EBS?
**A:** EBS is single-instance block storage; EFS is multi-instance NFS shared storage with automatic scaling.

**Q:** Why use Access Points for EFS?
**A:** They provide controlled entry paths, user/group permissions, and simplify mounting for multiple workloads.

---

## Screenshots to Capture

1. `efs-wave-image` security group inbound rule
2. EFS file system creation page
3. Access point details page
4. `/etc/fstab` entry
5. `df -h` output showing EFS mount
6. AMI creation screen

---

## Production Notes

- Always encrypt EFS in transit (`tls`) and at rest
- In multi-tenant environments, separate filesystems per app
- For Kubernetes/EKS, EFS works well with the **EFS CSI driver**
