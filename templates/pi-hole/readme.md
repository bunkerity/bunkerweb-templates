# Pi-hole on Root Domain with BunkerWeb

It is possible to serve Pi-hole at the root of your domain, but this adds an extra attack surface and is **not advised**. Additionally, you will always be redirected to `/admin/` — this is how Pi-hole is built and cannot be changed, by using bunkerweb you would need to modify pi-hole its self to achieve that.

If you still want to proceed, use the raw config below instead of the template.json.

---

## Raw Config

```env
IS_DRAFT=no
SERVER_NAME=example.com
BAD_BEHAVIOR_STATUS_CODES=400 403 404 405 429 444
BAD_BEHAVIOR_THRESHOLD=20
BAD_BEHAVIOR_BAN_TIME=3600
USE_BUNKERNET=yes
USE_DNSBL=no
GZIP_PROXIED=expired no-cache no-store private auth
REMOVE_HEADERS=Server Expect-CT X-Powered-By X-AspNet-Version X-AspNetMvc-Version Public-Key-Pins X-Frame-Options
AUTO_LETS_ENCRYPT=yes
EMAIL_LETS_ENCRYPT=registration@example.com
LIMIT_CONN_MAX_HTTP2=50
LIMIT_CONN_MAX_HTTP3=50
LIMIT_REQ_RATE=20r/s
USE_METRICS=no
MODSECURITY_SEC_RULE_ENGINE=DetectionOnly
USE_REAL_IP=yes
REAL_IP_FROM=172.65.0.0/16 172.64.0.0/13 104.16.0.0/13
REAL_IP_HEADER=CF-Connecting-IP
USE_REVERSE_PROXY=yes
REVERSE_PROXY_URL_1=/
REVERSE_PROXY_HOST_1=http://pihole-server-ip:80/admin/
REVERSE_PROXY_KEEPALIVE_1=yes
REVERSE_PROXY_HIDE_HEADERS_1=
REVERSE_PROXY_CONNECT_TIMEOUT_1=30s
```
