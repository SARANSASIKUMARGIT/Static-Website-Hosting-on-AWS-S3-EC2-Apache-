# Static Website Hosting on AWS — S3 + EC2 (Apache)

Host a static website's files in an **Amazon S3** bucket and serve them to end users from an **EC2 Linux instance** running Apache HTTP Server. The EC2 instance pulls the site files from S3 using an **IAM role**, and then handles all HTTP requests and responses.

> **Status: Decommissioned.** This was a personal learning project. The EC2 instance has been stopped and the AWS resources torn down to avoid billing charges, so the public IP `18.191.20.163` referenced throughout this README is no longer live. All steps, commands, and configuration below are reproducible in any AWS account.

---

## Architecture

```
                 ┌──────────────────────────────┐
  Browser ──────▶│  EC2 instance (t3.micro)      │
  HTTP :80       │  Amazon Linux 2023            │
                 │  Apache httpd                 │
                 │  DocumentRoot /var/www/html   │
                 └──────────────┬───────────────┘
                                │  aws s3 cp --recursive
                                │  (authorized by IAM role)
                                ▼
                 ┌──────────────────────────────┐
                 │  S3 bucket: chain-dev-web     │
                 │  66 static files (~3.7 MB)    │
                 │  index.html, assets/, vendor/ │
                 └──────────────────────────────┘

  Admin ──SSH :22 (MobaXterm + web-chain.pem)──▶ EC2 (ec2-user)
```

**Request flow:** a visitor hits `http://<EC2-public-IP>` → Apache on the EC2 instance resolves the request against `/var/www/html` → returns `index.html` and its assets. S3 acts as the origin/storage layer; EC2 acts as the web server.

---

## Tech Stack

| Component | Choice |
|---|---|
| Storage | Amazon S3 (General purpose bucket) |
| Compute | Amazon EC2, `t3.micro` (Free Tier eligible) |
| OS | Amazon Linux 2023 |
| Web server | Apache HTTP Server (`httpd`) |
| Access control | IAM Role (`AmazonS3FullAccess`), EC2 trusted entity |
| Remote access | SSH via MobaXterm, RSA `.pem` key pair |
| Region | US East (Ohio) — `us-east-2` |
| Website template | Chain App Dev (TemplateMo), Bootstrap v5.1.3 |

---

## Resources Created

| Resource | Value used in this project |
|---|---|
| S3 bucket | `chain-dev-web` |
| EC2 instance ID | `i-0f850ada064350d1c` (`instance-1`) |
| Public IPv4 | `18.191.20.163` *(released)* |
| Private IPv4 | `172.31.47.155` |
| Key pair | `web-chain.pem` (RSA, `.pem` / OpenSSH format) |
| Security group | `launch-wizard-1` (SSH 22, HTTP 80) |
| IAM role | `webchain` |
| Storage | 8 GiB gp3 root volume |
| Default SSH user | `ec2-user` |

---

## Deployment Steps

### 1. Prepare the website files

Download a free static HTML template, unzip it locally, and confirm `index.html` sits at the root of the folder alongside `assets/` and `vendor/`.

### 2. Create the S3 bucket

1. AWS Console → **Amazon S3** → **Create bucket**.
2. Bucket type: **General purpose**.
3. Bucket name: `chain-dev-web` (must be globally unique, 3–63 chars, lowercase).
4. Region: **US East (Ohio) us-east-2**.
5. Leave the remaining defaults → **Create bucket**.

### 3. Upload the site to S3

1. Open the bucket → **Upload**.
2. Drag and drop the unzipped website files **and** folders (`index.html`, `assets/`, `vendor/`).
3. Confirm the count (66 files, 3.7 MB in this project) → **Upload**.
4. Wait for **Upload succeeded** — all objects should report `Succeeded`.

### 4. Launch the EC2 instance

1. AWS Console → **EC2** → **Launch instance**.
2. **Name:** `instance-1`.
3. **AMI:** Amazon Linux 2023.
4. **Instance type:** `t3.micro`.
5. **Key pair:** *Create new key pair* → name `web-chain`, type **RSA**, format **.pem** → the file downloads locally. Keep it safe; it cannot be re-downloaded.
6. **Network settings → Firewall (security group):** create a new security group and tick:
   - ✅ **Allow SSH traffic from** Anywhere (`0.0.0.0/0`)
   - ✅ **Allow HTTP traffic from the internet**
   
   > `0.0.0.0/0` is used here for convenience. In any real deployment, restrict SSH to your own IP.
7. **Configure storage:** 8 GiB, gp3.

### 5. Add the initialization script (User data)

Under **Advanced details → User data**, paste the following. It runs once, on the instance's first boot, and installs + enables Apache:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
```

Then click **Launch instance**.

### 6. Verify Apache is running

Copy the **Public IPv4 address** from the instance summary and open it in a browser:

```
http://18.191.20.163
```

Apache's default page — **"It works!"** — confirms the web server is installed and reachable over HTTP.

### 7. Create the IAM role for S3 access

The EC2 instance needs permission to read from S3. Rather than storing access keys on the server, attach a role:

1. AWS Console → **IAM** → **Roles** → **Create role**.
2. **Trusted entity:** AWS service → **Use case: EC2**.
3. **Permissions:** search `s3` → select **`AmazonS3FullAccess`**.
   > Least privilege alternative: use `AmazonS3ReadOnlyAccess`, or a custom policy scoped to `arn:aws:s3:::chain-dev-web/*`. Full access is broader than this project needs.
4. **Role name:** `webchain` → **Create role**.

### 8. Attach the role to the instance

**EC2 → Instances →** select the instance **→ Actions → Security → Modify IAM role →** choose `webchain` → **Update IAM role**.

### 9. Connect to the instance over SSH

1. Install **MobaXterm** (or use OpenSSH / PuTTY).
2. **Session → SSH**:
   - **Remote host:** `18.191.20.163`
   - **Specify username:** `ec2-user`
   - **Advanced SSH settings → Use private key:** browse to `web-chain.pem`
3. Click **OK** to connect.

Equivalent command-line form:

```bash
chmod 400 web-chain.pem
ssh -i "web-chain.pem" ec2-user@ec2-18-191-20-163.us-east-2.compute.amazonaws.com
```

### 10. Copy the website from S3 to the web root

Once connected, confirm the role is working by listing buckets:

```bash
aws s3 ls
```

Expected output:

```
2025-12-26 13:01:07 chain-dev-web
```

Switch to Apache's document root and pull the files down:

```bash
cd /var/www/html
sudo aws s3 cp --recursive s3://chain-dev-web .
```

> **Gotcha:** `aws s3 cp -recursive ...` (single dash) fails with `Unknown options: -recursive`. The flag requires **two** dashes: `--recursive`. Don't omit the trailing `.` either — it is the destination (the current directory).

### 11. Verify the deployed site

Open the public IP again:

```
http://18.191.20.163
```

Apache now serves `index.html` from `/var/www/html`, and the full Chain App Dev landing page renders — replacing the default "It works!" page.

---

## Useful Commands

```bash
# List all S3 buckets visible to the attached IAM role
aws s3 ls

# List objects inside the bucket
aws s3 ls s3://chain-dev-web --recursive

# Sync instead of copy (only transfers changed files — better for redeploys)
cd /var/www/html
sudo aws s3 sync s3://chain-dev-web .

# Apache service management
sudo systemctl status httpd
sudo systemctl restart httpd
sudo systemctl enable httpd

# Confirm what Apache is serving
ls -l /var/www/html
```

---

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `Unknown options: -recursive` | Use `--recursive` (two dashes). |
| `Unable to locate credentials` / `AccessDenied` on `aws s3 ls` | IAM role not attached, or missing S3 permissions. Re-check **Actions → Security → Modify IAM role**. |
| Browser times out on port 80 | HTTP inbound rule missing from the security group. |
| "It works!" still shows after copying | Files landed in the wrong directory, or `index.html` is nested inside a subfolder. It must be directly in `/var/www/html`. |
| `Permission denied (publickey)` | Wrong username (`ec2-user` for Amazon Linux), wrong key, or `.pem` permissions too open — run `chmod 400`. |
| SSH connection refused | Instance is stopped, or the public IP changed after a stop/start cycle. |

---

## Notes & Limitations

- **The public IP is ephemeral.** Stopping and starting the instance assigns a new public IPv4 unless an **Elastic IP** is allocated and associated.
- **No HTTPS.** The site is served over plain HTTP on port 80. Production use would require ACM + an Application Load Balancer, or Certbot/Let's Encrypt on the instance.
- **No custom domain.** Traffic is addressed by raw IP. Route 53 (or any DNS provider) would map a domain to the instance.
- **Manual deploys.** Updating the site means re-uploading to S3 and re-running the copy command on the server. A `cron` job running `aws s3 sync`, or S3 Event Notifications, would automate this.
- **Simpler alternative:** for a purely static site, S3 Static Website Hosting + CloudFront removes the need for EC2 entirely. EC2 is used here deliberately, to practise server provisioning, SSH access, IAM roles, and Linux administration.

---

## Cost & Teardown

Everything above fits inside the AWS Free Tier (`t3.micro`, 8 GiB gp3, low S3 storage), but charges accrue once Free Tier credits lapse. This project was torn down for exactly that reason.

To decommission:

1. **EC2 → Instances →** select → **Instance state → Terminate instance**.
2. **EC2 → Elastic IPs →** release any allocated address.
3. **EC2 → Volumes →** confirm the root EBS volume was deleted on termination.
4. **S3 →** empty the bucket, then delete it.
5. **IAM → Roles →** delete the `webchain` role.
6. **EC2 → Key pairs →** delete `web-chain`, and remove the local `.pem`.

---

## What I Learned

- Difference between object storage (S3) and compute-backed hosting (EC2 + Apache).
- Provisioning a Linux server, opening ports through security groups, and bootstrapping software with EC2 **user data**.
- Why **IAM roles** are preferred over hard-coded access keys on an instance.
- SSH key-based authentication and remote Linux administration over the public internet.
- Apache's `DocumentRoot` convention (`/var/www/html`) and how a web server maps a root URL to `index.html`.

---

## Credits

Website template: **Chain App Dev** by [TemplateMo](https://templatemo.com/) — a free Bootstrap 5.1.3 landing page template.
