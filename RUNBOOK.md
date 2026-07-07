# RUNBOOK — deploying to AWS EC2 (free tier)

Everything here is the **account-dependent** part: the steps that need a real
AWS account with a card on file. The heavy lifting (installing and wiring up the
app + n8n + nginx) is already automated in [`cloud-init.yaml`](./cloud-init.yaml)
— you just paste it in at launch.

> **Order matters. Do Step 1 (billing alarm) before launching anything.**

---

## Step 1 — Billing alarm FIRST (before any instance)

Free tier is free for 12 months, but a misconfiguration can still cost money.
Set a hard tripwire before you launch a single resource.

1. Sign in → top-right account menu → **Billing and Cost Management**.
2. **Billing preferences** → tick **Receive Free Tier alerts** and **Receive
   CloudWatch billing alerts** → save. Enter your email.
3. Switch region to **US East (N. Virginia) / us-east-1** (billing metrics only
   live there) → **CloudWatch** → **Alarms** → **Create alarm**.
4. **Select metric** → **Billing** → **Total Estimated Charge** → **USD**.
5. Condition: **Greater than `1` USD** → Next.
6. Notification: create an SNS topic, enter your email, confirm the subscription
   email that arrives → finish.

Now if estimated charges ever cross **$1**, you get an email the same day.

CLI equivalent (once the AWS CLI is configured):

```bash
aws cloudwatch put-metric-alarm --alarm-name billing-over-1usd \
  --namespace "AWS/Billing" --metric-name EstimatedCharges \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum --period 21600 --evaluation-periods 1 \
  --threshold 1 --comparison-operator GreaterThanThreshold \
  --alarm-actions <YOUR_SNS_TOPIC_ARN> --region us-east-1
```

---

## Step 2 — Launch the EC2 instance

1. **EC2** → **Launch instance**.
2. **Name:** `docuchat-n8n`.
3. **AMI:** Ubuntu Server 24.04 LTS (free-tier eligible).
4. **Instance type:** `t3.micro` (or `t2.micro` where t3 isn't free-tier in your
   region). Both are free-tier eligible.
5. **Key pair:** create one (`docuchat-key`), download the `.pem` — this is your
   SSH key, keep it safe.
6. **Network / security group** — create a new group with exactly three inbound
   rules:
   | Type  | Port | Source            | Why |
   |-------|------|-------------------|-----|
   | SSH   | 22   | **My IP**         | admin only — never 0.0.0.0/0 |
   | HTTP  | 80   | Anywhere (0.0.0.0/0) | serve the app + n8n |
   | HTTPS | 443  | Anywhere (0.0.0.0/0) | after you add a domain |
7. **Storage:** 8–20 GB gp3 (within the 30 GB free-tier allowance).
8. **Advanced details → User data:** paste the entire contents of
   `cloud-init.yaml`.
9. **Launch.**

The instance provisions itself on first boot (~3–5 min). No manual install.

---

## Step 3 — Verify

Grab the instance's **Public IPv4 address** from the console, then:

```bash
# app should answer on the root
curl http://<PUBLIC_IP>/health

# n8n UI (first visit asks you to create an owner account)
open http://<PUBLIC_IP>/n8n/
```

SSH in to check services if anything's off:

```bash
ssh -i docuchat-key.pem ubuntu@<PUBLIC_IP>
sudo systemctl status docuchat n8n nginx --no-pager
sudo tail -n 50 /var/log/cloud-init-output.log   # provisioning log
```

Set app secrets (OpenAI/Supabase/etc.) in `/etc/docuchat.env`, n8n secrets in
`/etc/n8n.env`, then `sudo systemctl restart docuchat n8n`.

---

## Step 4 — HTTPS (needs a domain)

Let's Encrypt won't issue for `*.compute.amazonaws.com`, so point a domain you
own (an A record → the instance's public IP) and then:

```bash
sudo snap install --classic certbot
sudo certbot --nginx -d yourdomain.com     # edits app.conf, reloads nginx, auto-renews
```

For n8n behind the proxy over HTTPS, set in `/etc/n8n.env`:

```
N8N_PROTOCOL=https
WEBHOOK_URL=https://yourdomain.com/n8n/
```

then `sudo systemctl restart n8n`.

---

## Step 5 — Cost guardrails

- **Stop** (not terminate) the instance when you don't need it — stopped
  instances don't bill compute (you still pay a few cents/mo for the EBS volume).
- Free-tier compute is **750 hours/month** of t2/t3.micro — one always-on
  instance fits inside that for 12 months.
- Watch the **Free Tier** page under Billing for usage against limits.
- The $1 alarm from Step 1 is your backstop.

---

*This kit is provider-agnostic in spirit — the same cloud-init runs on any
Ubuntu VPS (Hetzner, DigitalOcean, Lightsail). EC2 is just the free-tier target.*
