# Kestra Step-by-Step Setup Guide

This guide walks you through setting up and running Kestra locally with your specific API credentials.

---

## Step 1: Export Environment Variables & Secrets
Kestra reads credentials using environment variables prefixed with `SECRET_`. The values must be base64-encoded. 

### macOS / Linux (bash/zsh)
Run the following commands in your terminal to export your API keys:

```bash
# 1. Export your raw Gemini API Key
export GEMINI_API_KEY="AQ.Ab8RN6KecGXjQtXLXuK1RoEKwUEML4xvLyjhb3LxVRd0jKoBsw"

# 2. Base64 encode the Gemini key for Kestra's secret engine
export SECRET_GEMINI_API_KEY=$(echo -n $GEMINI_API_KEY | base64)

# 3. Base64 encode and export the Tavily Search API key (used for web search workflows)
export SECRET_TAVILY_API_KEY=$(echo -n "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" | base64)
```

### Windows (PowerShell)
Run the following commands in PowerShell to export your API keys:

```powershell
# 1. Export your raw Gemini API Key
$env:GEMINI_API_KEY="AQ.Ab8RN6KecGXjQtXLXuK1RoEKwUEML4xvLyjhb3LxVRd0jKoBsw"

# 2. Base64 encode the Gemini key for Kestra's secret engine
$env:SECRET_GEMINI_API_KEY=[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($env:GEMINI_API_KEY))

# 3. Base64 encode and export the Tavily Search API key (used for web search workflows)
$env:SECRET_TAVILY_API_KEY=[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("your-key"))
```

> [!NOTE]
> *   **Why base64 encode?** Kestra requires secrets injected via environment variables to be base64-encoded to protect special characters and structure.
> *   **Referencing Secrets:** Inside Kestra flows, you access these values using `{{ secret('GEMINI_API_KEY') }}` or `{{ secret('TAVILY_API_KEY') }}` without the `SECRET_` prefix.

---

## Step 2: Start Kestra
Navigate to the directory containing your `docker-compose.yml` file and start Kestra in detached (background) mode:

```bash
docker compose up -d
```

*   **`docker compose up`**: Starts the containers defined in the Compose file.
*   **`-d` (detached)**: Runs the containers in the background, freeing up your terminal.

---

## Step 3: Access Kestra UI & Login
Once the containers are running:
1. Open your browser and navigate to: **[http://localhost:8080](http://localhost:8080)**
2. Log in using the default credentials configured in `docker-compose.yml`:
   *   **Username**: `admin@kestra.io`
   *   **Password**: `Admin1234!`

---

## Step 4: Import Example Flows
You can load the predefined flows into Kestra using `curl` directly from your terminal:

### macOS / Linux (bash/zsh)
```bash
# Navigate to the workspace directory
cd /Users/username/03-orchestration-G15

# Import flow examples
curl -X POST -u 'admin@kestra.io:Admin1234!' http://localhost:8080/api/v1/flows/import -F fileUpload=@flows/1_chat_without_rag.yaml
curl -X POST -u 'admin@kestra.io:Admin1234!' http://localhost:8080/api/v1/flows/import -F fileUpload=@flows/2_chat_with_rag.yaml
curl -X POST -u 'admin@kestra.io:Admin1234!' http://localhost:8080/api/v1/flows/import -F fileUpload=@flows/3_rag_with_websearch.yaml
```

### Windows (PowerShell)
```powershell
# Navigate to the workspace directory
cd C:\path\to\03-orchestration-G15

# Import flow examples using curl.exe
curl.exe -X POST -u "admin@kestra.io:Admin1234!" http://localhost:8080/api/v1/flows/import -F "fileUpload=@flows/1_chat_without_rag.yaml"
curl.exe -X POST -u "admin@kestra.io:Admin1234!" http://localhost:8080/api/v1/flows/import -F "fileUpload=@flows/2_chat_with_rag.yaml"
curl.exe -X POST -u "admin@kestra.io:Admin1234!" http://localhost:8080/api/v1/flows/import -F "fileUpload=@flows/3_rag_with_websearch.yaml"
```

---

## Step 5: Shutting Down
To stop the Kestra services and free up system memory:

```bash
docker compose down
```

> [!WARNING]
> Do not use `docker compose down -d`. The `down` command does not accept a `-d` flag (this will throw an `unknown shorthand flag` error).
