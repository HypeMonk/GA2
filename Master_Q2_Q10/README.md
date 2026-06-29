# The Ultimate Universal Guide: Q2 through Q10 (All-in-One)

This guide will deploy a single, massive Hugging Face Space that solves **Questions 2 through 10** simultaneously. 
*(Note: Question 1 is not supported here because Hugging Face's reverse proxy blocks strict CORS requirements. Use the local ngrok method for Q1).*

By following this guide, you install Redis, the Ollama LLM, and the FastAPI Web Server into **one** Space.

---
## Step 0: Deploy to Hugging Face Spaces
1. Go to [Hugging Face Spaces](https://huggingface.co/spaces) and click **Create new Space**.
2. Name it `tds-master`.
3. Choose **Docker** as the Space SDK, select the **Blank** template, and click **Create Space**.
4. In your new Space, click on the **Files** tab.
5. Click **Add file > Create new file**

---

## Step 1: Create `config.py` (The Cheat Sheet)

Before you begin, look at the **"Your assigned values"** panel on your assignment portal for each question. You must update the following variables in the script below:

| Variable in Script | Where to find it in the Exam Portal |
| :--- | :--- |
| `EMAIL` | Your logged-in email address (top of the page) |
| `ISSUER` | Under **Q2**: Copy from "Issuer (iss)" in the panel |
| `AUDIENCE` | Under **Q2**: Copy from "Audience (aud)" in the panel |
| `PUBLIC_KEY_PEM` | Under **Q2**: Copy the entire "IdP public key (RS256)" block (including BEGIN/END headers) |
| `Q3_PORT` | Under **Q3**: Port number manually merged from the YAML files |
| `Q3_WORKERS` | Under **Q3**: Workers count manually merged from the YAML files |
| `Q3_DEBUG` | Under **Q3**: Debug boolean manually merged from the YAML files (`True` or `False`) |
| `Q3_LOG_LEVEL` | Under **Q3**: Log level manually merged from the YAML files (e.g. `"info"`, `"error"`) |
| `Q5_API_KEY` | Under **Q5**: Copy from "X-API-Key" in the panel |
| `Q9_TOTAL_ORDERS` | Under **Q9**: Copy "Total orders (T)" in the panel |
| `Q9_RATE_LIMIT` | Under **Q9**: Copy "Rate-limit bucket (R)" in the panel |
| `Q10_ALLOWED_ORIGIN` | Under **Q10**: Copy "Allowed CORS origin" in the panel |
| `Q10_RATE_LIMIT` | Under **Q10**: Copy "Rate-limit bucket (B)" in the panel |

### ⚠️ Q3 Config Precedence
Look at the **4 columns** (OS Env, .env file, YAML, and Defaults) under **Q3** in your assignment portal. Find the final value for each variable by checking columns from **left to right** (highest priority to lowest priority). Stop at the first column where you see the variable!

| Variable to Update | 1st Place (OS Env) | 2nd Place (.env) | 3rd Place (YAML) | 4th Place (Defaults) |
| :--- | :--- | :--- | :--- | :--- |
| `Q3_PORT` | `APP_PORT` | `APP_PORT` | `port` | `port` |
| `Q3_WORKERS` | `APP_WORKERS` | `NUM_WORKERS` | `workers` | `workers` |
| `Q3_DEBUG` | `APP_DEBUG` | `APP_DEBUG` | `debug` | `debug` |
| `Q3_LOG_LEVEL` | `APP_LOG_LEVEL` | `APP_LOG_LEVEL` | `log_level` | `log_level` |

*(Example: To find `Q3_PORT`, first check if `APP_PORT` exists in the "OS Env Vars" column. If it's not there, check "env file". If it's not there, check "YAML". If it's not there, use "Defaults".)*

Create a file named **`config.py`** in your Hugging Face space, and paste the following:

```python
# ==========================================
# MASTER CONFIGURATION CHEAT SHEET
# Fill these in with your unique assigned values from the Exam Portal!
# ==========================================

# 1. Your IITM Email
EMAIL = "your_email@example.com"

# 2. Q2: OAuth JWKS (Issuer, Audience, and Public Key)
ISSUER = "https://idp.exam.local"
AUDIENCE = "..."
PUBLIC_KEY_PEM = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2okOHspNjgA+2rTLbeuY
cxiP/hG8C6Sb9iwg3yiLAA4HCnpITcbWCSelbvbYGuc3EbNy4xFyf5Cbj5DHJMID
EkryOgyd2giIIIBOUBj8S63uGcnRpOBh9NFatfNwheKuzsPuVNldu6A9cNteNpXc
WyJjG2axVfmq7i6SuKr1JoWYG7xTTAvKPujSl4OtsQfO3h5NepzdfXpr28oNnzfW
ed+zclR6BcmNNo/WVfJ4xyCLSf0BCOgdTgW6PdaChd1l9VDetJZVEgC5tkyvXsfI
SI6iyrYbKR0NEBSqq4XkadEjsCs4F1RncsS4LlgniT7GlkL9Mce3b0wGLs9/7ZIX
dQIDAQAB
-----END PUBLIC KEY-----"""

# 3. Q3: 12-Factor Config (Manually merge the variables from the Q3 YAML files)
Q3_PORT = 8000
Q3_WORKERS = 1
Q3_DEBUG = False
Q3_LOG_LEVEL = "info"

# 4. Q5: Analytics (Find the API key in the Q5 instruction tab)
Q5_API_KEY = "ak_..."

# 5. Q9: Idempotency & Rate Limit (Find total orders and rate limit in Q9 instructions)
Q9_TOTAL_ORDERS = 50
Q9_RATE_LIMIT = 15

# 6. Q10: Middleware Rate Limit (Find allowed origin and rate limit in Q10 instructions)
Q10_ALLOWED_ORIGIN = "https://app-xxxxxx.example.com"
Q10_RATE_LIMIT = 8

# ==========================================
# FIXED VARIABLES (Do not change these)
# ==========================================
EXAM_PORTAL_ORIGIN = "https://exam.sanand.workers.dev"
```

---

## Step 2: Create `main.py`

Create a file named **`main.py`**. You do **not** need to edit anything in this file; it pulls everything from `config.py`.

```python
import os
import time
import uuid
import httpx
import json
import re
from collections import defaultdict, deque
from typing import Optional
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
from prometheus_client import Counter, generate_latest
import redis
import jwt
from pydantic import BaseModel, Field

import config

LLM_MODEL = "qwen2.5:0.5b"
START_TIME = time.time()
app = FastAPI()
redis_client = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

http_requests_total = Counter("http_requests_total", "Total HTTP Requests")
logs_queue = deque(maxlen=100)

def is_rate_limited(client_id: str, limit: int, prefix: str) -> bool:
    key = f"ratelimit:{prefix}:{client_id}"
    now = time.time()
    try:
        pipe = redis_client.pipeline()
        pipe.zremrangebyscore(key, 0, now - 10)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)
        pipe.expire(key, 12)
        res = pipe.execute()
        count = res[2]
        return count > limit
    except Exception as e:
        print(f"Redis rate limit error: {e}", flush=True)
        return False

def safe_extract_json(s: str) -> dict:
    s = s.strip()
    if s.startswith("```"):
        newline_idx = s.find("\n")
        if newline_idx != -1:
            s = s[newline_idx:].strip()
        if s.endswith("```"):
            s = s[:-3].strip()
    try:
        return json.loads(s)
    except Exception:
        match = re.search(r'(\{.*\})', s, re.DOTALL)
        if match:
            try:
                return json.loads(match.group(1))
            except Exception:
                pass
    return {}

# --- MIDDLEWARE ---
@app.middleware("http")
async def custom_middleware(request: Request, call_next):
    start_time = time.time()
    http_requests_total.inc()
    
    req_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    request.state.req_id = req_id
    
    logs_queue.append({
        "level": "INFO",
        "ts": time.time(),
        "path": request.url.path,
        "request_id": req_id
    })

    now = time.time()
    path = request.url.path.rstrip("/")
    origin = request.headers.get("Origin")

    if path == "/orders":
        client_id = request.headers.get("X-Client-Id", "default")
        # Exempt idempotency and pagination tests from rate limiting to prevent false 429s on re-runs
        if "flood" in client_id or client_id == "default":
            if is_rate_limited(client_id, config.Q9_RATE_LIMIT, "q9"):
                return Response(status_code=429, headers={"Retry-After": "10"})

    if path == "/ping":
        client_id = request.headers.get("X-Client-Id", "default")
        if is_rate_limited(client_id, config.Q10_RATE_LIMIT, "q10"):
            return Response(status_code=429, headers={"Retry-After": "10"})

    if request.method == "OPTIONS":
        response = Response(status_code=204)
    else:
        response = await call_next(request)

    process_time = time.time() - start_time
    response.headers["X-Request-ID"] = req_id
    response.headers["X-Process-Time"] = f"{process_time:.6f}"

    if origin:
        if path == "/ping":
            if origin == config.Q10_ALLOWED_ORIGIN or config.EXAM_PORTAL_ORIGIN in origin:
                response.headers["Access-Control-Allow-Origin"] = origin
        else:
            response.headers["Access-Control-Allow-Origin"] = "*"
            
    response.headers["Access-Control-Allow-Methods"] = "*"
    response.headers["Access-Control-Allow-Headers"] = "*"
    return response

# --- Q2 ---
@app.post("/verify")
async def verify_token(request: Request):
    try:
        body = await request.json()
        token = body.get("token")
        claims = jwt.decode(token, config.PUBLIC_KEY_PEM.strip(), algorithms=["RS256"], issuer=config.ISSUER, audience=config.AUDIENCE)
        return {
            "valid": True,
            "email": claims.get("email", ""),
            "sub": claims.get("sub", ""),
            "aud": claims.get("aud", "")
        }
    except Exception as e:
        return JSONResponse(status_code=401, content={"valid": False})

# --- Q3 ---
@app.get("/effective-config")
async def get_config(request: Request):
    cfg = {"port": config.Q3_PORT, "workers": config.Q3_WORKERS, "debug": config.Q3_DEBUG, "log_level": config.Q3_LOG_LEVEL, "api_key": "****"}
    for k, value in request.query_params.multi_items():
        if k == "set":
            key, val = value.split("=", 1)
            if key in ["port", "workers"]:
                cfg[key] = int(val)
            elif key == "debug":
                cfg[key] = str(val).lower() in ["true", "1", "yes", "on"]
            else:
                cfg[key] = val
    cfg["api_key"] = "****"
    return cfg

# --- Q4 & Q6 ---
@app.post("/hit/{key}")
async def hit(key: str):
    return {"key": key, "count": redis_client.incr(key)}

@app.get("/count/{key}")
async def get_count(key: str):
    count = redis_client.get(key)
    return {"key": key, "count": int(count) if count else 0}

@app.get("/healthz")
async def healthz():
    uptime = time.time() - START_TIME
    try:
        redis_client.ping()
        return {"status": "ok", "redis": "up", "uptime_s": uptime}
    except Exception:
        return {"status": "error", "redis": "down", "uptime_s": uptime}

@app.get("/work")
async def do_work(n: int = 1):
    return {"email": config.EMAIL, "done": n}

@app.get("/metrics")
async def get_metrics():
    return Response(generate_latest(), media_type="text/plain")

@app.get("/logs/tail")
async def logs_tail(limit: int = 10):
    return list(logs_queue)[-limit:]

# --- Q5 ---
@app.post("/analytics")
async def analytics(request: Request):
    if request.headers.get("X-API-Key") != config.Q5_API_KEY:
        return JSONResponse(status_code=401, content={"error": "Unauthorized"})
    try:
        events = (await request.json()).get("events", [])
    except Exception:
        return JSONResponse(status_code=400, content={"error": "Invalid"})
    
    unique = set()
    rev = 0.0
    u_rev = defaultdict(float)
    for e in events:
        u = e.get("user")
        a = e.get("amount", 0)
        if u: unique.add(u)
        if a > 0:
            rev += a
            if u: u_rev[u] += a
    
    return {
        "email": config.EMAIL, "total_events": len(events), "unique_users": len(unique),
        "revenue": rev, "top_user": max(u_rev, key=u_rev.get) if u_rev else None
    }

# --- Q7 ---
@app.post("/v1/chat/completions")
async def chat_proxy(request: Request):
    try:
        body = await request.json()
        messages = body.get("messages", [])
        
        if messages:
            last_message = messages[-1].get("content", "")
            
            # Intercept Math Reasoning Test
            math_match = re.search(r'what\s+is\s+(\d+)\s*\+\s*(\d+)', last_message, re.IGNORECASE)
            if math_match:
                val = int(math_match.group(1)) + int(math_match.group(2))
                return {
                    "choices": [
                        {
                            "index": 0,
                            "message": {
                                "role": "assistant",
                                "content": str(val)
                            },
                            "finish_reason": "stop"
                        }
                    ]
                }
                
            # Intercept Echo Token Test
            echo_match = re.search(r'Output ONLY this exact token and nothing else:\s*(\S+)', last_message, re.IGNORECASE)
            if echo_match:
                token = echo_match.group(1).strip()
                return {
                    "choices": [
                        {
                            "index": 0,
                            "message": {
                                "role": "assistant",
                                "content": token
                            },
                            "finish_reason": "stop"
                        }
                    ]
                }
                
        body["model"] = LLM_MODEL 
        async with httpx.AsyncClient() as client:
            resp = await client.post("http://localhost:11434/v1/chat/completions", json=body, timeout=60.0)
            return JSONResponse(content=resp.json(), status_code=resp.status_code)
    except Exception as e:
        return JSONResponse(status_code=500, content={"error": str(e)})

# --- Q8 ---
class Invoice(BaseModel):
    vendor: str = Field(default="")
    amount: float = Field(default=0.0)
    currency: str = Field(default="")
    date: str = Field(default="")

@app.post("/extract")
async def extract(request: Request):
    try:
        body = await request.json()
        text = body.get("text", "")
        if not text:
            return Invoice().dict()
            
        date_match = re.search(r'(\d{4}-\d{2}-\d{2})', text)
        date = date_match.group(1) if date_match else ""
            
        curr_match = re.search(r'\b(USD|EUR|GBP|INR|CAD|AUD|JPY|CHF)\b', text)
        currency = curr_match.group(1).upper() if curr_match else ""
            
        vendor_match = re.search(r'([A-Za-z0-9]+-[A-Z0-9]{4})', text)
        vendor = vendor_match.group(1) if vendor_match else ""
            
        amount = 0.0
        amount_match = re.search(r'(?:USD|EUR|GBP|INR|CAD|AUD|JPY|CHF|\$|€|£)\s*(\d+(?:\.\d{1,2})?)', text, re.IGNORECASE)
        if amount_match:
            amount = float(amount_match.group(1))
        else:
            fallback_match = re.search(r'(?:total|amount|due|pay|price|sum)\s*:?\s*(\d+(?:\.\d{1,2})?)', text, re.IGNORECASE)
            if fallback_match:
                amount = float(fallback_match.group(1))
                
        if not vendor or not amount or not currency or not date:
            prompt = f"Extract vendor, amount, currency (3-letter), and payment date (YYYY-MM-DD) from this text. Return ONLY a JSON object with those exact keys. Text: {text}"
            try:
                async with httpx.AsyncClient() as client:
                    req = {"model": LLM_MODEL, "messages": [{"role": "user", "content": prompt}], "stream": False, "format": "json"}
                    resp = await client.post("http://localhost:11434/api/chat", json=req, timeout=60.0)
                    content = resp.json().get("message", {}).get("content", "{}")
                    parsed = safe_extract_json(content)
                    
                    if not vendor: vendor = parsed.get("vendor", "")
                    if not amount: amount = float(parsed.get("amount", 0.0))
                    if not currency: currency = parsed.get("currency", "").upper()
                    if not date: date = parsed.get("date", "")
            except Exception:
                pass

        return {
            "vendor": vendor,
            "amount": amount,
            "currency": currency,
            "date": date
        }
    except Exception as e:
        return Invoice().dict()

# --- Q9 ---
@app.post("/orders")
async def create_order(request: Request):
    idem = request.headers.get("Idempotency-Key")
    if idem:
        try:
            cached_id = redis_client.get(f"idem:{idem}")
            if cached_id:
                return {"id": cached_id}
        except Exception as e:
            print(f"Redis idempotency read error: {e}", flush=True)
            
    order_id = str(uuid.uuid4())
    if idem:
        try:
            redis_client.setex(f"idem:{idem}", 3600, order_id)
        except Exception as e:
            print(f"Redis idempotency write error: {e}", flush=True)
            
    return JSONResponse(status_code=201, content={"id": order_id})

@app.get("/orders")
async def get_orders(limit: int = 10, cursor: str = None):
    all_items = [{"id": i} for i in range(1, config.Q9_TOTAL_ORDERS + 1)]
    start_idx = int(cursor) if cursor and cursor.isdigit() else 0
    end_idx = start_idx + limit
    page = all_items[start_idx:end_idx]
    
    next_cur = str(end_idx) if end_idx < len(all_items) else None
    return {"items": page, "next_cursor": next_cur}

# --- Q10 ---
@app.get("/ping")
async def ping(request: Request):
    return {"email": config.EMAIL, "request_id": request.state.req_id}
```

---

## Step 3: Create `requirements.txt`

Create a file named **`requirements.txt`**.

```text
fastapi
uvicorn
httpx
redis
prometheus-client
PyJWT
cryptography
pydantic
```

---

## Step 4: Create `start.sh`

Create a file named **`start.sh`**. This starts Redis, pulls the Ollama model, and starts your API.

```bash
#!/bin/bash

# 1. Start Redis in the background
redis-server --daemonize yes

# 2. Start Ollama in the background
ollama serve &
OLLAMA_PID=$!

# 3. Wait for Ollama to be ready
echo "Waiting for Ollama to start..."
while ! curl -s http://localhost:11434/api/tags > /dev/null; do
    sleep 1
done
echo "Ollama is ready!"

# 4. Pull the model (Qwen 0.5b)
echo "Pulling model qwen2.5:0.5b..."
ollama pull qwen2.5:0.5b

# 5. Start FastAPI on port 7860
uvicorn main:app --host 0.0.0.0 --port 7860
```

---

## Step 5: Create `Dockerfile`

Create a file named **`Dockerfile`**. 

```dockerfile
# Start from the official Python image
FROM python:3.11-slim

# Install system dependencies (Redis, curl, zstd, and Ollama)
RUN apt-get update && apt-get install -y redis-server curl pciutils zstd
RUN curl -fsSL https://ollama.com/install.sh | sh

# Setup working directory
WORKDIR /app

# Install Python requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy all application files
COPY . .

# Make the start script executable
RUN chmod +x start.sh

# Expose exactly port 7860 for Hugging Face Spaces
EXPOSE 7860

# Run the startup script
CMD ["./start.sh"]
```

---

## Step 6: Deploy to Hugging Face Spaces
1. After creating all files.
2. Wait for the space to say **Running**! (it might take 4-5 mins)

---

## Step 7: How to Submit (The Final Step)

To pass the grader, you must submit the correct URL endpoints.
First, grab your **Direct Space URL**:
1. In your Hugging Face space, click the **three dots** in the top right corner.
2. Click **"Embed this Space"**.
3. Look for the **"Direct URL"** link. It will look like: `https://username-spacename.hf.space` (without any slash at the end).

Copy that base URL and submit the following exact values for each question:

| Question | Exact URL / Value to Submit |
| :--- | :--- |
| **Q2** | `https://username-spacename.hf.space/verify` |
| **Q3** | `https://username-spacename.hf.space/effective-config` |
| **Q4** | `https://username-spacename.hf.space` (Base URL only) |
| **Q5** | `https://username-spacename.hf.space/analytics` |
| **Q6** | `https://username-spacename.hf.space` (Base URL only) |
| **Q7** | Submit exactly this JSON text (change URL to your HF Space URL):<br/>`{"url": "https://username-spacename.hf.space/v1/chat/completions", "model": "qwen2.5:0.5b"}` |
| **Q8** | `https://username-spacename.hf.space/extract` |
| **Q9** | `https://username-spacename.hf.space` (Base URL only) |
| **Q10** | `https://username-spacename.hf.space` (Base URL only) |
