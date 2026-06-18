# When the Server Keeps Dying: A Cloudflare Bot Story

Our web servers started falling over intermittently a few weeks back. Not crashing in a clean, obvious way — just grinding to a halt under load, timing out, and recovering slowly. The kind of failure that makes you question everything you thought you knew about the stack.

What followed was several days of troubleshooting that took us through WordPress internals, plugin audits, access log archaeology, and eventually to a conclusion we genuinely did not expect.

## The Symptoms

- PHP-FPM workers exhausting their pool repeatedly throughout the day
- MySQL CPU spiking on queries that had always been fast
- Pages timing out sporadically, then recovering, then timing out again
- No obvious pattern tied to actual human traffic spikes

The servers were not undersized for the traffic we expected. Something was generating far more load than the analytics suggested.

## First Suspect: WordPress

WordPress was the obvious starting point. The sites were running a reasonable number of plugins, but we went through them anyway.

### Plugin Audit

We deactivated plugins in batches and profiled each configuration with [Query Monitor](https://wordpress.org/plugins/query-monitor/). A few things came out of it:

- Two plugins were making external HTTP calls on every page load, adding 300–800 ms of blocking time
- A contact form plugin was running its spam-check database queries without an index, causing full table scans
- A slider plugin was loading its full JS/CSS bundle on every page regardless of whether the page had a slider

We fixed all of those. Page load times improved noticeably. But the servers kept falling over.

### Caching

We tightened up object caching, enabled full-page caching via a plugin, and confirmed opcache was warm. Helped at the margins. Still not the root cause.

## Second Suspect: Bots

Pulling up the raw access logs told a different story to the analytics dashboard. The analytics only count sessions with JavaScript execution — bots don't run JS. The access logs count everything.

```bash
# Requests per minute over the last hour
awk '{print $4}' access.log \
  | cut -d: -f1-3 \
  | sort | uniq -c | sort -rn \
  | head -20
```

The volume was enormous. Hundreds of requests per second hitting dynamic WordPress routes — `/?p=`, `/?s=`, `wp-login.php`, `xmlrpc.php` — all of which bypass the page cache and hammer PHP and MySQL directly.

We added rate limiting rules, dropped known bad user-agents, and blocked the most aggressive IP ranges. Each measure helped briefly. Then the patterns shifted and load crept back up.

## The Actual Problem: Cloudflare Was Forwarding Everything

Here is what took too long to figure out: we were running the sites behind Cloudflare's proxy. Cloudflare was absorbing the requests at the edge, inspecting them… and then **forwarding the vast majority on to the origin server anyway**.

By default, Cloudflare does not block bots — it categorises them and applies your configured rules. Our rules were not aggressive enough. The bots that Cloudflare flagged as "verified" (Googlebot, Bingbot, and various legitimate crawlers) were passed straight through. On top of that, the "unknown" bot category — which covers most of the junk — was set to allow by default.

The compounding problem: because all requests arrive at the server from **Cloudflare's own IP ranges**, you cannot block at the network level without blocking all legitimate traffic too. Any IP ban you write blocks Cloudflare, not the bot behind it. The real source IPs are only available in the `CF-Connecting-IP` header, and mod_security / fail2ban do not read headers — they see the connection IP.

```
Real bot IP → Cloudflare edge → (forwards) → Your server (sees Cloudflare IP)
```

Every firewall rule we wrote was useless.

## What We Tried with Cloudflare

We went through Cloudflare's tooling:

- **Bot Fight Mode** — turned on, made a small difference for the most obvious junk
- **Super Bot Fight Mode** (Pro plan) — more aggressive, but verified bots still passed through
- **WAF custom rules** — we wrote rules to challenge or block high-rate paths like `xmlrpc.php` and `wp-login.php`; helped with specific endpoints but not the general crawl load
- **Rate limiting** — effective for burst attacks, less so for distributed low-rate crawling spread across many source IPs

The issue was never going to be fully solved at the Cloudflare layer without a paid plan with aggressive bot management and a lot of ongoing rule maintenance. The bots were too varied, the rate per individual IP too low to trigger simple rate limits, and the verified crawlers were legitimate and untouchable.

## The Decision: Turn Off the Cloudflare Proxy

After weighing it up, we turned off Cloudflare's orange-cloud proxy for the affected domains and switched to **DNS-only mode** (the grey cloud in Cloudflare's dashboard).

This means:

- Traffic goes directly to the origin server — Cloudflare is no longer in the path
- We lose Cloudflare's CDN caching, DDoS mitigation, and SSL termination at the edge
- **We get real source IPs back at the server**

With real IPs visible, our existing tooling started working properly. fail2ban can ban abusive IPs. mod_security can track per-IP request rates. We can write actual firewall rules. The signal-to-noise ratio in the logs improved immediately because we could see who was really talking to us.

Within a few hours of switching, server load dropped to what we would have expected from the legitimate traffic volume. PHP-FPM stopped exhausting its worker pool. MySQL settled down.

## What We Kept from Cloudflare

Cloudflare DNS management stayed in place — it is still the authoritative nameserver, just not proxying. That means:

- We keep the Cloudflare dashboard for DNS record management
- Cloudflare's DDoS protection at the network layer (L3/L4) still applies even in DNS-only mode
- SSL is now handled directly at the server with Let's Encrypt

## Lessons Learned

**Check your access logs, not just your analytics.** Analytics tools only see human sessions. Bots show up in the raw logs and the gap between the two can be shocking.

**Cloudflare's proxy is not a bot firewall by default.** It provides protection, but it also acts as a forwarding layer for enormous amounts of bot traffic unless you actively configure and maintain bot rules.

**You cannot firewall Cloudflare's IPs.** If you are behind the proxy and your bot problem is volume-based, your server-level tools are blind to source IPs. Factor this in before assuming Cloudflare is handling the problem.

**Sometimes the right answer is less abstraction.** A CDN proxy adds value in specific scenarios — high static asset traffic, genuine DDoS mitigation needs, global latency reduction. For origin-heavy WordPress sites absorbing bot load, it was creating more problems than it solved for us.
