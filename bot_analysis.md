# Bot Traffic Analysis on LEMP WordPress EC2 Server

## Introduction

This report covers the analysis of bot traffic on a WordPress website hosted on an EC2 instance using the LEMP (Linux, Nginx, MySQL, PHP) stack. The goal is to:

1. Detect whether bots are accessing the website.
2. Format and extract specific log data.
3. Identify useful and aggressive bots, particularly for the US, Europe, and China regions.
4. Recommend bot filtering strategies.

---

## 1. Prerequisites and Environment Setup

Ensure you are connected to your EC2 instance via SSH.

```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

### Log File Location

Nginx access logs are usually located at:

```bash
/var/log/nginx/access.log
```

If your virtual host is configured with a custom access log, verify it in the virtual host configuration:

```bash
sudo nano /etc/nginx/sites-available/your-site
```

Look for a line like:

```nginx
access_log /var/log/nginx/your-site-access.log;
```

---

## 2. Checking for Bot Access

### 2.1 Checking for Bot Calls

Search your logs for common bots by User-Agent strings. Run:

```bash
grep -i "bot\|crawl\|spider" /var/log/nginx/access.log | less
```

This command looks for lines containing the words "bot", "crawl", or "spider" (common in bot User-Agents).

### 2.2 Sample Output Breakdown

Example log line:

```
66.249.66.1 - - [18/Jul/2025:10:20:50 +0000] "GET / HTTP/1.1" 200 1024 "-" "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
```

**Explanation:**

- IP Address: `66.249.66.1`
- Date and Time: `18/Jul/2025:10:20:50 +0000`
- Status Code: `200`
- Size: `1024` bytes
- User-Agent: `Googlebot`

---

## 3. Extracting & Formatting Required Data

### 3.1 Time and Date

```bash
awk '{print $4}' /var/log/nginx/access.log | tr -d '[' | sort | uniq -c | head
```

This command extracts timestamps, strips brackets, and summarizes unique time entries.

### 3.2 Number of Bits (Bytes Served)

```bash
awk '{sum += $10} END {print "Total bytes served:", sum}' /var/log/nginx/access.log
```

This adds the 10th field (bytes sent) for all requests.

### 3.3 Hits on Unique IPs

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

This will list the IPs with the highest number of hits.

### 3.4 Bot Crawl Detection

```bash
grep -i "bot\|crawl\|spider" /var/log/nginx/access.log | awk -F\" '{print $6}' | sort | uniq -c | sort -nr | head
```

This isolates and counts unique bot User-Agent strings.

---

## 4. Bot Behavior by Region

### 4.1 Identifying Aggressive and Useful Bots

You can analyze bots by reverse DNS lookups or by IP ranges (geo-locating them). Use tools like `geoiplookup` or online services to identify the region.

#### Install geoiplookup (Ubuntu):

```bash
sudo apt update && sudo apt install geoip-bin -y
```

#### Usage:

```bash
grep -i "bot" /var/log/nginx/access.log | awk '{print $1}' | sort | uniq | while read ip; do echo "$ip - $(geoiplookup $ip)"; done
```

This shows which bots come from which countries.

### 4.2 Region-Based Bot Assessment

#### **US & Europe: Useful & Aggressive Bots**

| Bot        | Usefulness | Aggressiveness | Notes                             |
| ---------- | ---------- | -------------- | --------------------------------- |
| Googlebot  | High       | Moderate       | Essential for SEO, from US        |
| Bingbot    | Medium     | Moderate       | Also US-based, useful for Bing    |
| AhrefsBot  | Low        | High           | Aggressive; consumes bandwidth    |
| SemrushBot | Medium     | High           | Aggressive but provides analytics |
| YandexBot  | Low        | High           | Mostly for Russian audience       |

#### **China: Useful Bots**

| Bot          | Usefulness | Aggressiveness | Notes                                 |
| ------------ | ---------- | -------------- | ------------------------------------- |
| Baiduspider  | High       | Moderate       | Important for Chinese search presence |
| Sogou Spider | Medium     | Moderate       | Used by Sogou search engine           |
| 360Spider    | Medium     | High           | From Qihoo 360, often aggressive      |

Use the above table to create firewall rules or robots.txt entries accordingly.

---

## 5. Recommendations

### 5.1 Create a Custom robots.txt

```txt
User-agent: AhrefsBot
Disallow: /

User-agent: SemrushBot
Crawl-delay: 10

User-agent: Baiduspider
Allow: /

User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
```

### 5.2 Use Firewall Rules (e.g., UFW or iptables)

```bash
sudo ufw deny from 192.168.1.100 to any
```

Or block by country (requires additional tools like `xtables-addons` and MaxMind DB).

### 5.3 Rate Limit Bots in Nginx

```nginx
limit_req_zone $binary_remote_addr zone=botlimit:10m rate=1r/s;

server {
  location / {
    limit_req zone=botlimit burst=5;
  }
}
```

---

## 6. Conclusion

This document helps identify and analyze bot traffic on your WordPress site hosted with LEMP. By filtering logs and combining geolocation with bot User-Agent analysis, you can:

- Recognize useful vs aggressive bots.
- Fine-tune your server for SEO bots (like Googlebot, Baiduspider).
- Restrict bandwidth-heavy bots.
- Regionally optimize your site’s visibility and performance.

---

## References

- [https://www.keycdn.com/blog/web-crawlers](https://www.keycdn.com/blog/web-crawlers)
- [https://moz.com/learn/seo/search-engine-crawlers](https://moz.com/learn/seo/search-engine-crawlers)
- [https://www.projecthoneypot.org](https://www.projecthoneypot.org)
- [https://ipinfo.io](https://ipinfo.io)
- [https://www.cloudflare.com/learning/bots/what-is-bot-management/](https://www.cloudflare.com/learning/bots/what-is-bot-management/)

---

