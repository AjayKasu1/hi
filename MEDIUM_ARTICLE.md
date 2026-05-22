# How I Moved a Side Project Off My Laptop and Onto the Internet for $0

A practical guide to taking any Python/FastAPI app from local-only to publicly-accessible, with HTTPS, on Google Cloud's free tier.

---

## The problem

I had a side project running on my Mac. It worked great when I was using my computer. The moment I closed my laptop, it died.

I'd open my laptop in the morning and realize my app had been dead for 18 hours. Sometimes longer. Caffeinate kept the Mac awake, but it didn't help when the Mac restarted itself for an update, or my power adapter wiggled loose, or macOS just decided to be weird.

I'd been telling myself I'd "deploy it properly someday" for months.

One weekend I finally did. It cost me $0. It took me about 4 hours. Most of that was fighting Python dependencies. Now my app runs 24/7 on a server I never have to think about.

This post is the general process. It applies to pretty much any Python web app: a trading bot, a scraper, a small API, an ML inference server, a hobby SaaS. The specifics will vary but the shape is the same.

## What you'll need

- A Google account
- A credit card (Google verifies it but doesn't charge for free tier)
- Your Python project that runs locally
- Around 2 hours

That's it. No prior cloud experience required.

---

## Step 1: Pick the right free tier

The "free" landscape is confusing because most platforms call something "free" that's actually a trial. Here's what real, actually-free options look like in 2026:

| Provider | What you actually get | Catch |
|---|---|---|
| **Google Cloud (e2-micro)** | 1 small VM, 30 GB disk, forever | Region-locked to us-east1, us-west1, us-central1 |
| **Oracle Cloud (ARM Ampere)** | Up to 4 cores, 24 GB RAM, forever | Signup verification is finicky |
| **AWS (t2.micro)** | 1 VM for 12 months only | After 12 months you start paying |
| **Render, Railway free** | Fake free | Sleeps after inactivity, wipes state |

If your app needs to be ALWAYS RUNNING (mine did, anything that polls APIs or holds state does), avoid anything that sleeps. The "free tier" of Render or Railway will save your laptop battery but it'll destroy your app's state every time it wakes up.

I picked Google Cloud because the free tier is genuinely free forever, and the e2-micro is just big enough for a small Python web app.

## Step 2: Spin up the VM

This is where most tutorials skip the critical details. You have to use SPECIFIC settings to actually get the free tier. Get them wrong and you'll see a $7/month bill instead of $0.

Inside Google Cloud Console, go to Compute Engine → Create Instance. Use these:

- **Machine type**: e2-micro (the only one that qualifies as Always Free)
- **Region**: us-east1, us-west1, or us-central1 (other regions cost money)
- **OS**: Debian 12 (lightweight, has Python 3.11 in apt)
- **Boot disk**: 30 GB Standard Persistent Disk (Balanced and SSD are NOT free)
- **Firewall**: Allow HTTP and HTTPS traffic

That's it. The pricing estimate on the right will say ~$7/month. **Ignore it.** Google shows list price, not the Always Free discount. Your actual bill will be $0.

If you're paranoid (you should be), set a $1 budget alert before doing anything else. Search "budgets" in the GCP console, create one with email notifications. If anything ever costs more than $1, you'll know within hours.

## Step 3: Get a real domain (for free)

You need a domain name so Let's Encrypt can issue a real HTTPS certificate. You don't need to buy one.

Go to DuckDNS, sign in with Google, pick a subdomain. You get something like `yourname.duckdns.org`. Point it at your VM's external IP. It's free, it works, it's been around for years.

Real domains are nice but not required. DuckDNS subdomains get real SSL certs from Let's Encrypt and look fine in URLs.

## Step 4: The reverse proxy that does all the hard work

This is where Caddy is magical.

A reverse proxy sits between the internet and your app. It handles HTTPS, can block dangerous URLs, rate-limits attackers, and forwards good traffic to your app on localhost.

Caddy makes this a 5-line config:

```
yourname.duckdns.org {
    reverse_proxy localhost:8000
}
```

That's the whole config. Caddy automatically:

- Requests a Let's Encrypt SSL certificate
- Renews it every 90 days
- Serves HTTPS on port 443
- Forwards requests to your app

If your app has any URLs that should NEVER be accessible from the public internet (admin actions, debug endpoints, anything that mutates state without auth), block them at this layer:

```
yourname.duckdns.org {
    @dangerous path /api/admin/* /api/shutdown
    respond @dangerous "403 Forbidden" 403

    reverse_proxy localhost:8000
}
```

Anyone hitting `/api/admin/*` gets a 403 from Caddy before the request even reaches your app. Defense in depth.

## Step 5: Make it actually run forever

systemd is Linux's built-in process manager. It's what keeps web servers, databases, and everything else running on Linux servers. You write a small unit file and systemd handles:

- Starting your app on boot
- Restarting it if it crashes
- Restarting it if it runs out of memory
- Capturing stdout/stderr to a queryable log

The unit file is straightforward:

```
[Unit]
Description=My App
After=network-online.target

[Service]
User=youruser
WorkingDirectory=/home/youruser/myapp
EnvironmentFile=/home/youruser/myapp/.env
ExecStart=/home/youruser/myapp/.venv/bin/python main.py
Restart=always
RestartSec=10
MemoryMax=950M

[Install]
WantedBy=multi-user.target
```

`Restart=always` is the magic line. If your app crashes, systemd restarts it in 10 seconds. If the server reboots, your app starts automatically.

`MemoryMax=950M` is critical on a 1 GB e2-micro. Without it, a runaway memory leak would crash the entire VM, not just your app.

After writing the file, two commands:

```
sudo systemctl enable myapp    # start on boot
sudo systemctl start myapp     # start now
```

Done. Your app is now running forever.

## Step 6: The "it works on my Mac" trap

This is where I lost an hour. Your code works locally because your Mac has all sorts of Python packages installed from old experiments. Your fresh VM has none of them.

I had to install five packages separately that I didn't realize were dependencies, each error cascading to the next:

1. `alpaca-py` (I had two different Alpaca SDKs installed locally and didn't know it)
2. `alpha_vantage<3.0.0` (a sub-dependency that broke between versions)
3. `duckduckgo-search` (used by one tiny utility I forgot about)
4. `requests` (transitive, I assumed it was always there)
5. `pytz` (same)

Every time I restarted my app, a different error appeared. The fix took 20 minutes but I felt incompetent the whole time.

**Lesson**: Pin every dependency in your requirements.txt with version numbers. Use `pip freeze > requirements.txt` once your local environment is stable. Commit that file. Future-you will thank you.

```
fastapi==0.118.0
uvicorn[standard]==0.32.0
numpy==1.26.4
# ... etc
```

The unpinned `fastapi` you have today is not the same `fastapi` that'll be on PyPI in 6 months.

## Step 7: Test what you shipped

Before declaring victory, verify three things:

**Bot health**:
```
sudo systemctl status myapp
```
You want to see "Active: active (running)" in green.

**Live logs**:
```
sudo journalctl -u myapp -f
```
You should see your app's normal log output streaming. Press Ctrl+C to exit.

**The actual URL**:
Open `https://yourname.duckdns.org` in any browser. First load takes 20 to 40 seconds while Let's Encrypt issues the certificate. After that, instant.

**Security check**: If you blocked any admin endpoints at the proxy, hit them directly:
```
https://yourname.duckdns.org/api/admin/whatever
```
Should return 403 Forbidden.

## Step 8: Back up your code

Don't skip this. Your VM could die, your laptop could die, your code could vanish.

Create a private GitHub repo. Push everything except your `.env` and any state files.

The critical part is the `.gitignore`. Get this right BEFORE your first commit:

```
.env
.env.*
*.log
*.db
__pycache__/
.venv/
*.json   # if any of your state files are JSON
```

Run a verification before pushing:

```
git status | grep -E "\.env|\.db|password"
```

If anything matches, STOP. Fix the `.gitignore`. Your API keys are about to land on GitHub.

Once it's clean:

```
git push -u origin main
```

Future deploys become a one-line change:

```
# On your laptop
git push

# On the VM
git pull && sudo systemctl restart myapp
```

30 seconds per deploy instead of "tarball, upload, extract, restart" which is 3 to 5 minutes.

---

## What this whole thing costs

Adding everything up:

- VM: $0 (Always Free e2-micro)
- Disk: $0 (Always Free 30 GB Standard PD)
- Bandwidth: $0 (first 1 GB egress is free, my app uses maybe 100 MB)
- DuckDNS domain: $0 (free forever)
- Let's Encrypt SSL cert: $0 (free forever, auto-renews)
- systemd, Caddy: $0 (open source)

**Total: $0 / month forever.**

I genuinely do not pay for this. After 8 months it's still running, still free, still not bothering me.

---

## Lessons that travel

These apply to any project, not just the one I deployed.

### 1. Side projects on laptops aren't really shipped

If your project depends on your specific computer being awake and connected, you don't have a product. You have a demo. The minute you put it on a server you stop thinking about it, which means you can actually iterate on the interesting parts.

### 2. Free tier is real if you read the terms

Most engineers I know are convinced free cloud tiers are bait. They're not. Read the actual Always Free documentation. The limits are very specific and very generous for small projects.

### 3. Pin your dependencies

`pip freeze > requirements.txt` is a 30-second hygiene step that saves you hours later. Do it.

### 4. Network proximity matters more than you think

I picked us-east1 because my app talks to a broker headquartered in New Jersey. The latency is 15 ms instead of 70 ms in us-west1. That sounds small. It matters for anything time-sensitive.

### 5. Reverse proxy + systemd is the standard stack

Once you've set this up once, you can deploy any web service. It's the boring infrastructure pattern that powers half the internet. Nginx + supervisord works the same way. Caddy + systemd is the modern, easier version.

### 6. Document your gotchas

While you're hitting weird errors, write them down. Future-you will hit the same errors deploying the next project. A 200-line markdown file titled GOTCHAS.md is more valuable than a 50-page tutorial because it's about the specific traps YOUR setup has.

### 7. The hard part is rarely the headline

I expected the hard part to be Caddy or systemd. Those were 10 minutes. The hard part was Python dependencies that worked on my Mac but didn't exist on the fresh VM. The hard part is always something you didn't anticipate. Budget time for unknown unknowns.

---

## What I would do differently next time

- Pin requirements.txt before, not after
- Set up the GitHub repo before deploying, not after
- Use Tailscale instead of a public URL if the dashboard is only for me
- Take a screenshot of the working dashboard at the end. It's the proof you can show people

---

## Closing thought

If you have a side project sitting on your laptop right now, you don't need to ship it perfectly. You just need to ship it.

The first time will take you longer than it should. Most of that time will be Python package errors and DNS propagation and figuring out which button to click in some console you've never seen before. That's fine. Every subsequent deploy will take 10 minutes.

After this you'll have:

- A real URL you can put on your resume
- A working understanding of how the internet actually serves your code
- A repo with deployment scripts and a runbook you can reuse forever
- One less thing tied to your laptop's battery

Total cost: a Saturday afternoon and $0/month.

Worth it.
