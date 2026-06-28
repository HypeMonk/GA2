# Universal Guide: TDS Q2 (OAuth JWKS Verify Server)

### Step 1: Get Your Unique Values
Before you begin, look at the "Your assigned values" panel on your assignment portal. 
You must update the following variables in the script below:

| Variable in Script | Where to find it |
| :--- | :--- |
| `ISSUER` | Copy from "Issuer (iss)" in the panel |
| `AUDIENCE` | Copy from "Audience (aud)" in the panel |
| `PUBLIC_KEY_PEM` | Copy the entire "IdP public key (RS256)" block (including BEGIN/END headers) |

---

### Step 2: Deploy to Hugging Face Spaces
1. Go to [Hugging Face Spaces](https://huggingface.co/spaces) and click **Create new Space**.
2. Name it `tds-q2-server`.
3. Choose **Docker** as the Space SDK, select the **Blank** template, and click **Create Space**.
4. In your new Space, click on the **Files** tab.
5. Click **Add file > Create new file** and create `Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
# cryptography is required to decode RS256!
RUN pip install --no-cache-dir fastapi uvicorn "pyjwt[cryptography]" cryptography
COPY main.py .
EXPOSE 7860
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "7860"]
```
6. Click **Add file > Create new file** again, and create `main.py` with this exact code:

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import jwt

app = FastAPI()

# !!! UPDATE THESE THREE VARIABLES !!!
AUDIENCE = "tds-xxxxxxxx.apps.exam.local" 
ISSUER = "https://idp.exam.local"
PUBLIC_KEY_PEM = """-----BEGIN PUBLIC KEY-----
... paste your key from the panel here ...
-----END PUBLIC KEY-----"""

@app.post("/verify")
async def verify(request: Request):
    try:
        body = await request.json()
        token = body.get("token", "")
        
        # PyJWT handles signature, audience, and issuer validation automatically
        claims = jwt.decode(
            token,
            PUBLIC_KEY_PEM,
            algorithms=["RS256"],
            audience=AUDIENCE,
            issuer=ISSUER
        )
        
        return JSONResponse(content={
            "valid": True,
            "email": claims.get("email", ""),
            "sub": claims.get("sub", ""),
            "aud": claims.get("aud", "")
        })
        
    except Exception as e:
        print("Rejected Token:", e, flush=True)
        return JSONResponse(status_code=401, content={"valid": False})
```

---

### Step 3: Get Your URL and Submit
1. Wait for the space to rebuild and show the green **Running** badge.
2. You will see 3 dots at right side then click **"Embed this Space"**. 
3. You will get a space URL. It looks like `https://yourusername-tds-q2-server.hf.space`.
4. Add `/verify` to the end of that URL (so it becomes `https://yourusername-tds-q2-server.hf.space/verify`).
5. Submit this exact URL to the exam platform to get your green checkmark!
