# Universal Guide: TDS Q3 (Resolve 12-Factor Config Precedence)

### Step 1: Get Your Unique Values
Instead of writing complex code to merge the YAML and `.env` files, we can just manually merge them ourselves by reading the "Your assigned config layers" panel on your assignment portal.

Look at the 4 columns in your panel and find the final value for each variable by checking them in this exact order (from highest priority to lowest priority). Stop at the first column where you see the variable!

| Variable to Update | 1st Place to Check | 2nd Place to Check | 3rd Place to Check | 4th Place to Check |
| :--- | :--- | :--- | :--- | :--- |
| `port` | OS Env Vars (`APP_PORT`) | .env file (`APP_PORT`) | YAML (`port`) | Defaults (`port`) |
| `workers` | OS Env Vars (`APP_WORKERS`) | .env file (`NUM_WORKERS`) | YAML (`workers`) | Defaults (`workers`) |
| `debug` | OS Env Vars (`APP_DEBUG`) | .env file (`APP_DEBUG`) | YAML (`debug`) | Defaults (`debug`) |
| `log_level` | OS Env Vars (`APP_LOG_LEVEL`) | .env file (`APP_LOG_LEVEL`) | YAML (`log_level`) | Defaults (`log_level`) |
| `api_key` | **Leave as `"****"`** | - | - | - |

*(For example: To find your `port`, check OS ENV vars for `APP_PORT`. If it's not there, check .env file for `APP_PORT`. If it's not there, check yaml columns for `port`.)*

---

### Step 2: Deploy to Hugging Face Spaces
1. Go to [Hugging Face Spaces](https://huggingface.co/spaces) and click **Create new Space**.
2. Name it `tds-q3-server`.
3. Choose **Docker** as the Space SDK, select the **Blank** template, and click **Create Space**.
4. In your new Space, click on the **Files** tab.
5. Click **Add file > Create new file** and create `Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir fastapi uvicorn
COPY main.py .
EXPOSE 7860
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "7860"]
```
6. Click **Add file > Create new file** again, and create `main.py` with this exact code:
(Change your values in this code according to yours)

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/effective-config")
async def get_config(request: Request):
    
    # !!! UPDATE THIS DICTIONARY WITH YOUR MANUALLY MERGED VALUES !!!
    config = {
        "port": 8000,
        "workers": 1,
        "debug": False,
        "log_level": "info",
        "api_key": "****"
    }
    
    # Apply the CLI overrides from the ?set=key=value URL parameters
    for key, value in request.query_params.multi_items():
        if key == "set":
            k, v = value.split("=", 1)
            
            if k in ["port", "workers"]:
                config[k] = int(v)
            elif k == "debug":
                config[k] = str(v).lower() in ["true", "1", "yes", "on"]
            else:
                config[k] = v
                
    # Always hide the api key!
    config["api_key"] = "****"
    
    return config
```

---

### Step 3: Get Your URL and Submit
1. Wait for the space to rebuild and show the green **Running** badge.
2. You will see 3 dots at right side then click **"Embed this Space"**. 
3. You will get a space URL. It looks like `https://yourusername-tds-q3-server.hf.space`.
4. Add `/effective-config` to the end of that URL (so it becomes `https://yourusername-tds-q3-server.hf.space/effective-config`).
5. Submit this exact URL to the exam platform to get your green checkmark!
