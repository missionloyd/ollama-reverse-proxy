# Ollama Proxy with Nginx and Cloudflare Tunnel

This project provides a Dockerized Nginx server configured to act as a
reverse proxy for [Ollama](https://github.com/jmorganca/ollama), a local
AI model serving platform. The proxy includes built-in authentication
using a custom `Authorization` header and exposes the Ollama service
over the internet using a Cloudflare Tunnel.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
  - [Authorization Token](#authorization-token)
  - [Cloudflare Tunnel Token](#cloudflare-tunnel-token)
  - [Environment Variables](#environment-variables)
- [Usage](#usage)
- [Testing the Service](#testing-the-service)
- [Renaming Ollama Models](#renaming-ollama-models)
- [Files in this Project](#files-in-this-project)
- [License](#license)

## Features

- **Nginx Reverse Proxy**: Proxies requests to Ollama running on the host machine at port `11434`.
- **Authentication**: Requires clients to provide a specific `Authorization` header to access the service.
- **CORS Support**: Configured to handle Cross-Origin Resource Sharing (CORS) for web-based clients.
- **Cloudflare Tunnel Integration**: Exposes the local Ollama service securely over the internet using Cloudflare Tunnel.
- **Dockerized Setup**: Easily deployable using Docker and Docker Compose.

## Prerequisites

- **Docker and Docker Compose**: Ensure Docker and Docker Compose are installed on your system.
- **Cloudflare Account**: Required to obtain a Cloudflare Tunnel token.
- **Ollama**: Install and run [Ollama](https://github.com/jmorganca/ollama) on your host machine at port `11434`.

## Installation

1. **Clone the Repository**

   ```bash
   git clone https://github.com/kesor/ollama-reverse-proxy.git
   cd ollama-reverse-proxy
   ```

2. **Copy and Configure Environment Variables**

   Copy the example environment file and modify it with your own values:

   ```bash
   cp .env.local-example .env.local
   ```

   Edit `.env.local` and set your Cloudflare Tunnel token and Ollama secret API key:

   ```bash
   CLOUDFLARE_TUNNEL_TOKEN="your_cloudflare_tunnel_token"
   OLLAMA_SECRET_API_KEY="your_made_up_ollama_secret_api_key"
   ```

## Configuration

### Authorization Token

- **Environment Variable**: `OLLAMA_SECRET_API_KEY`
- **Usage**: This token is required in the `Authorization` header for clients to access the Ollama service through the proxy.
- **Format**: Typically starts with `sk-` followed by your made up secret key.

### Cloudflare Tunnel Token

- **Environment Variable**: `CLOUDFLARE_TUNNEL_TOKEN`
- **How to Obtain**: Log in to your Cloudflare Zero Trust account and [create a tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/) to get the token.

### Environment Variables

The project uses a `.env.local` file to manage sensitive environment variables.

- **`.env.local`**: Contains the following variables:

  ```bash
  CLOUDFLARE_TUNNEL_TOKEN="your_cloudflare_tunnel_token"
  OLLAMA_SECRET_API_KEY="your_made_up_ollama_secret_api_key"
  ```

## Usage

### Step 0a: Install local model of your choice

```bash
docker volume create ollama
docker run -d --name ollama-init -v ollama:/root/.ollama ollama/ollama
docker exec -it ollama-init ollama pull llama3.2
docker stop ollama-init
docker rm ollama-init
```

### Step 0b: Create ollama-network for relevant device(s)

For example, to set up 5 devices:

```bash
docker stop ollama
docker rm ollama

docker network create ollama-network

docker run -d --runtime=nvidia --gpus '"device=0"' -v ollama:/root/.ollama -p 11435:11434 --restart always --name ollama1 --network ollama-network ollama/ollama
docker run -d --runtime=nvidia --gpus '"device=1"' -v ollama:/root/.ollama -p 11436:11434 --restart always --name ollama2 --network ollama-network ollama/ollama
docker run -d --runtime=nvidia --gpus '"device=2"' -v ollama:/root/.ollama -p 11437:11434 --restart always --name ollama3 --network ollama-network ollama/ollama
docker run -d --runtime=nvidia --gpus '"device=3"' -v ollama:/root/.ollama -p 11438:11434 --restart always --name ollama4 --network ollama-network ollama/ollama
docker run -d --runtime=nvidia --gpus '"device=4"' -v ollama:/root/.ollama -p 11439:11434 --restart always --name ollama5 --network ollama-network ollama/ollama
```

This set of commands will create ollama servers within the same network for each GPU.

### Step 0c: (Optional) Check all models running within each container

```bash
docker exec -it ollama1 ollama run llama3.2
docker exec -it ollama2 ollama run llama3.2
docker exec -it ollama3 ollama run llama3.2
docker exec -it ollama4 ollama run llama3.2
docker exec -it ollama5 ollama run llama3.2
```

### Step 0d: (Optional) Connect other apps to the same network

```bash
docker network connect ollama-network ragflow-server
```

1. **Build and Run the Docker Container Using Docker Compose**

   ```bash
   docker compose up -d
   ```

   This command will build the Docker image and start the container defined in `docker-compose.yml`.

2. **Look at the logs for signs of any errors**

   ```bash
   docker compose logs
   ```

2. **Access the Ollama Service**

   - The service is now exposed over the internet via the Cloudflare Tunnel at the endpoint: `http://<your-server-ip-or-domain>:11433/ollama/v1/chat/completions`
   - Clients must include the correct `Authorization` header in their requests.

## Testing the Service

You can test the setup using the following `curl` command:

```bash
curl -i https://opxy.example.net/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_made_up_ollama_secret_api_key" \
  -d '{
    "model":"llama3.2",
    "messages":[
      {"role":"system","content":"You are a helpful assistant."},
      {"role":"user","content":"Test prompt to check your response."}
    ],
    "temperature":1,
    "max_tokens":10,
    "stream":false
  }'
```

Replace `your_made_up_ollama_secret_api_key` with your made up secret API key.

## Renaming Ollama Models

For systems that expect OpenAI's models to "be there", it is useful to
rename Ollama models by copying them to a new name using the Ollama CLI.

For example:

```bash
ollama cp llama3.2:3b-instruct-fp16 gpt-4o
```

This command copies the model `llama3.2:3b-instruct-fp16` to a new model
named `gpt-4o`, making it easier to reference in API requests.

## Files in this Project

- **`Dockerfile`**: The Dockerfile used to build the Docker image.
- **`docker-compose.yml`**: Docker Compose configuration file to set up the container.
- **`nginx-default.conf.template`**: Template for the Nginx configuration file.
- **`40-entrypoint-cloudflared.sh`**: Entry point script to install and start Cloudflare Tunnel.
- **`.env.local-example`**: Example environment file containing placeholders for sensitive variables.

## Privacy Considerations

This project relies on Cloudflare as a middleman for the Cloudflare Tunnel. If you trust Cloudflare, the setup ensures that no one else can eavesdrop on your traffic or access your data.

- **SSL Encryption**: The public endpoint opened by Cloudflare has SSL enabled, meaning that any communication between your computer and this endpoint is encrypted.
- **Cloudflare Tunnel Encryption**: Requests received at the Cloudflare endpoint are securely forwarded to your local Nginx instance through Cloudflare's tunnel service, which also encrypts this communication.
- **Local Network Traffic**: Inside the container, requests between the Cloudflare tunnel process and Nginx, as well as between Nginx and the Ollama process, occur over the local device network in clear text over HTTP. Since this traffic stays within the local network, it is not exposed externally.

If privacy beyond this is a concern, note that local traffic within the container is not encrypted, although it is isolated from external networks.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

**Note**: The Nginx configuration has been carefully set up to handle CORS
headers appropriately. You can refer to the `nginx-default.conf.template`
file to understand the specifics.

**Disclaimer**: Replace placeholders like `your_cloudflare_tunnel_token`,
`your_ollama_secret_api_key`, and `opxy.example.net` with your actual values.

## Credits

This project is a modified version of the single ollama server code base written by @kesor ~ https://github.com/kesor/ollama-proxy/