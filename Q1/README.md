# Universal Guide: FastAPI Deployment Questions

## Step 1: Get Your Unique Values
Before you begin, look at the "Your assigned values" panel on your assignment portal. 
You must update the following variables in the script below:

| Variable in Script | Where to find it |
| :--- | :--- |
| `EMAIL` | Your logged-in email address |
| `ALLOWED_ORIGIN` | Copy from "Allowed CORS origin" in the panel |

---

## Step 2: Create main.py

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import uuid, time

app = FastAPI()
EMAIL = "YOUR_EMAIL_HERE"           # change to your mail id
ALLOWED_ORIGIN = "YOUR_ORIGIN_HERE" # e.g. https://dash-r2cr95.example.com

@app.middleware("http")
async def add_headers(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Request-ID"] = str(uuid.uuid4())
    response.headers["X-Process-Time"] = f"{time.time() - start:.6f}"
    return response

@app.options("/stats")
async def preflight(request: Request):
    origin = request.headers.get("origin", "")
    if origin == ALLOWED_ORIGIN:
        return JSONResponse(content={}, headers={
            "Access-Control-Allow-Origin": ALLOWED_ORIGIN,
            "Access-Control-Allow-Methods": "GET, OPTIONS",
            "Access-Control-Allow-Headers": "*"
        })
    return JSONResponse(content={})

@app.get("/stats")
async def stats(request: Request, values: str = ""):
    origin = request.headers.get("origin", "")
    nums = [int(x) for x in values.split(",") if x.strip()]
    if not nums:
        return JSONResponse(content={"error": "no values"}, status_code=400)
    result = {
        "email": EMAIL,
        "count": len(nums),
        "sum": sum(nums),
        "min": min(nums),
        "max": max(nums),
        "mean": round(sum(nums) / len(nums), 6)
    }
    headers = {}
    if origin == ALLOWED_ORIGIN:
        headers["Access-Control-Allow-Origin"] = ALLOWED_ORIGIN
    return JSONResponse(content=result, headers=headers)
```

---

## Step 3: Run Locally

```bash
pip install fastapi uvicorn
uvicorn main:app --host 0.0.0.0 --port 8000
```

Keep this terminal open until you submit.

---

## Step 4: Expose with ngrok

Open a **second terminal**:

```bash
ngrok http 8000
```

ngrok will show a URL like:
```
Forwarding  https://abc123.ngrok-free.app -> http://localhost:8000
```

Copy that `https://...ngrok-free.app` URL — that's what you submit.

---

## Multiple Deployment Questions — Port Problem

If you have multiple deployment questions, you can't run them all on port 8000.
Run each on a different port and create a separate ngrok tunnel for each:

**Terminal 1:** `uvicorn main_q1:app --port 8000`
**Terminal 2:** `uvicorn main_q2:app --port 8001`
**Terminal 3:** `uvicorn main_q3:app --port 8002`

**Terminal 4:** `ngrok http 8000` → submit this URL for Q1
**Terminal 5:** `ngrok http 8001` → submit this URL for Q2
**Terminal 6:** `ngrok http 8002` → submit this URL for Q3

> ⚠️ **ngrok free tier only allows 1 tunnel at a time.**
> Solutions:
> - Use **cloudflared** instead (unlimited tunnels, no account needed):
>   ```bash
>   cloudflared tunnel --url http://localhost:8000
>   cloudflared tunnel --url http://localhost:8001
>   cloudflared tunnel --url http://localhost:8002
>   ```
> - Or use **localtunnel** (no account, multiple tunnels):
>   ```bash
>   npx localtunnel --port 8000
>   npx localtunnel --port 8001
>   npx localtunnel --port 8002
>   ```

---

## Does the Server Need to Stay Running Until Submission?

**Yes** — the grader makes a live HTTP request to your URL at the moment you click Check or Submit. If uvicorn is stopped, the grader gets a connection error and marks it wrong.

**Keep all terminals open until you:**
1. Click Check and see "Correct"
2. Click the final Submit button
3. See the submission confirmation

After submission is confirmed, you can close everything.

---

## Quick Verification

Test your local server before submitting:

```bash
# Test stats endpoint
curl "http://localhost:8000/stats?values=1,2,3,4,5"

# Test CORS preflight (replace with your allowed origin)
curl -X OPTIONS "http://localhost:8000/stats" \
  -H "Origin: https://dash-r2cr95.example.com" \
  -H "Access-Control-Request-Method: GET" -v

# Test evil origin (should get no ACAO header)
curl -X OPTIONS "http://localhost:8000/stats" \
  -H "Origin: https://evil.com" \
  -H "Access-Control-Request-Method: GET" -v
```

Expected stats response:
```json
{
  "email": "23fxxxxx@ds.study.iitm.ac.in",
  "count": 5,
  "sum": 15,
  "min": 1,
  "max": 5,
  "mean": 3.0
}
```
