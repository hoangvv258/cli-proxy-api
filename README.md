 # CLI Proxy API Service

This service provides an OpenAI-compatible API proxy for Google Gemini models, running on Docker.

## Directory Structure

```
Proxy/
├── auths/                # Directory for login tokens (Mounted specific to the container)
├── logs/                 # Directory for log files
├── config.yaml           # Service configuration file
├── docker-compose.yml    # Docker configuration file
└── README.md             # This guide
```

## 1. Installation & Setup

### Prerequisites
- Docker & Docker Compose installed.

### `config.yaml`
Ensure `config.yaml` has the following content to fix the API port and auth path:

```yaml
auth-dir: "/root/.cli-proxy-api"
port: 8317
```

### `docker-compose.yml`
Ensure `docker-compose.yml` exposes the necessary ports (specifically `8085` for OAuth callback and `8317` for API):

```yaml
services:
  cli-proxy-api:
    image: eceasy/cli-proxy-api:latest
    container_name: cli-proxy-api
    pull_policy: always
    restart: unless-stopped
    ports:
      - "8317:8317"   # Main API port
      - "54545:54545" # Auth port
      - "8085:8085"   # OAuth callback port
    volumes:
      - ./config.yaml:/CLIProxyAPI/config.yaml
      - ./auths:/root/.cli-proxy-api
      - ./logs:/CLIProxyAPI/logs
    environment:
      - TZ=Asia/Ho_Chi_Minh
```

### Usage

In the `Proxy` directory, run:

```bash
docker compose up -d
```

## 2. Authentication

Run the following command for the first run or when the token has expired:

```bash
docker exec -it cli-proxy-api ./CLIProxyAPI -login -no-browser
```

1. The terminal will display an authentication URL.
2. Open that URL in your browser.
3. Login to Google and grant permissions.
4. Upon completion, the browser will redirect to `localhost:8085` (mapped to the container) and show a success message.
5. Back in the terminal, select a Project ID if prompted (enter `ALL` or a specific ID).


### 2.1 Custom Credentials (Optional)

If you want to use your own Google Cloud credentials:

1. Add `client-id` and `client-secret` to `config.yaml`:
   ```yaml
   auth-dir: "/root/.cli-proxy-api"
   port: 8317
   client-id: "YOUR_CLIENT_ID"
   client-secret: "YOUR_CLIENT_SECRET"
   ```
2. Restart the container:
   ```bash
   docker compose restart cli-proxy-api
   ```
3. Re-login using the command above.

## 3. API Usage

The service provides an OpenAI-compatible API on port `8317`.

- **Base URL:** `http://localhost:8317/v1`
- **Models Support:** `gemini-2.5-pro`, `gemini-2.5-flash`, `gemini-2.5-flash-lite`, ...

### Connection Check (Curl)

```bash
curl http://localhost:8317/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-test" \
  -d '{
    "model": "gemini-3-pro-preview",
    "messages": [{"role": "user", "content": "Hello, who are you?"}]
  }'
```

### Usage with Python (OpenAI SDK)

```python
from openai import OpenAI

# Configure client to point to local proxy
client = OpenAI(
    base_url="http://localhost:8317/v1",
    api_key="sk-test" # API Key is not important, proxy handles auth
)

# Call API as usual
response = client.chat.completions.create(
    model="gemini-3-pro-preview",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Write a python hello world code."}
    ]
)

print(response.choices[0].message.content)
```

## 4. Troubleshooting

- **OAuth Connection Error (localhost refused to connect):** Ensure port `8085` is added to `docker-compose.yml` and the container has been restarted (`docker compose restart`).
- **API Not Responding:** Check logs using `docker logs cli-proxy-api` to see if the server is listening on the correct port `8317`.
