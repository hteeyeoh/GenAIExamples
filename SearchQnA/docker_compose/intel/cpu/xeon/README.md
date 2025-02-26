# Build Mega Service of SearchQnA on Xeon

This document outlines the deployment process for a SearchQnA application utilizing the [GenAIComps](https://github.com/opea-project/GenAIComps.git) microservice pipeline on Intel Xeon server.

## 🚀 Build Docker images

### 1. Build Embedding Image

```bash
git clone https://github.com/opea-project/GenAIComps.git
cd GenAIComps
docker build --no-cache -t opea/embedding:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/embeddings/src/Dockerfile .
```

### 2. Build Retriever Image

```bash
docker build --no-cache -t opea/web-retriever:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/web_retrievers/src/Dockerfile .
```

### 3. Build Rerank Image

```bash
docker build --no-cache -t opea/reranking:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/rerankings/src/Dockerfile .
```

### 4. Build LLM Image

```bash
docker build --no-cache -t opea/llm-textgen:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/llms/src/text-generation/Dockerfile .
```

### 5. Build MegaService Docker Image

To construct the Mega Service, we utilize the [GenAIComps](https://github.com/opea-project/GenAIComps.git) microservice pipeline within the `searchqna.py` Python script. Build the MegaService Docker image using the command below:

```bash
git clone https://github.com/opea-project/GenAIExamples.git
cd GenAIExamples/SearchQnA
docker build --no-cache -t opea/searchqna:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f Dockerfile .
```

### 6. Build UI Docker Image

Build frontend Docker image via below command:

```bash
cd GenAIExamples/SearchQnA/ui
docker build --no-cache -t opea/opea/searchqna-ui:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f ./docker/Dockerfile .
```

Then run the command `docker images`, you will have following images ready:

1. `opea/embedding:latest`
2. `opea/web-retriever:latest`
3. `opea/reranking:latest`
4. `opea/llm-textgen:latest`
5. `opea/searchqna:latest`
6. `opea/searchqna-ui:latest`

## 🚀 Set the environment variables

Before starting the services with `docker compose`, you have to recheck the following environment variables.

```bash
export host_ip=<your External Public IP>    # export host_ip=$(hostname -I | awk '{print $1}')
export GOOGLE_CSE_ID=<your cse id>
export GOOGLE_API_KEY=<your google api key>
export HUGGINGFACEHUB_API_TOKEN=<your HF token>

export EMBEDDING_MODEL_ID=BAAI/bge-base-en-v1.5
export TEI_EMBEDDING_ENDPOINT=http://${host_ip}:3001
export RERANK_MODEL_ID=BAAI/bge-reranker-base
export TEI_RERANKING_ENDPOINT=http://${host_ip}:3004
export BACKEND_SERVICE_ENDPOINT=http://${host_ip}:3008/v1/searchqna

export TGI_LLM_ENDPOINT=http://${host_ip}:3006
export LLM_MODEL_ID=Intel/neural-chat-7b-v3-3

export MEGA_SERVICE_HOST_IP=${host_ip}
export EMBEDDING_SERVICE_HOST_IP=${host_ip}
export WEB_RETRIEVER_SERVICE_HOST_IP=${host_ip}
export RERANK_SERVICE_HOST_IP=${host_ip}
export LLM_SERVICE_HOST_IP=${host_ip}

export EMBEDDING_SERVICE_PORT=3002
export WEB_RETRIEVER_SERVICE_PORT=3003
export RERANK_SERVICE_PORT=3005
export LLM_SERVICE_PORT=3007
```

## 🚀 Start the MegaService

```bash
cd GenAIExamples/SearchQnA/docker_compose/intel/cpu/xeon
docker compose up -d
```

## 🚀 Test MicroServices

```bash
# tei
curl http://${host_ip}:3001/embed \
    -X POST \
    -d '{"inputs":"What is Deep Learning?"}' \
    -H 'Content-Type: application/json'

# embedding microservice
curl http://${host_ip}:3002/v1/embeddings\
  -X POST \
  -d '{"text":"hello"}' \
  -H 'Content-Type: application/json'

# web retriever microservice
export your_embedding=$(python3 -c "import random; embedding = [random.uniform(-1, 1) for _ in range(768)]; print(embedding)")
curl http://${host_ip}:3003/v1/web_retrieval \
  -X POST \
  -d "{\"text\":\"What is the 2024 holiday schedule?\",\"embedding\":${your_embedding}}" \
  -H 'Content-Type: application/json'


# tei reranking service
curl http://${host_ip}:3004/rerank \
    -X POST \
    -d '{"query":"What is Deep Learning?", "texts": ["Deep Learning is not...", "Deep learning is..."]}' \
    -H 'Content-Type: application/json'

# reranking microservice
curl http://${host_ip}:3005/v1/reranking\
  -X POST \
  -d '{"initial_query":"What is Deep Learning?", "retrieved_docs": [{"text":"Deep Learning is not..."}, {"text":"Deep learning is..."}]}' \
  -H 'Content-Type: application/json'

# tgi service
curl http://${host_ip}:3006/generate \
  -X POST \
  -d '{"inputs":"What is Deep Learning?","parameters":{"max_new_tokens":17, "do_sample": true}}' \
  -H 'Content-Type: application/json'

# llm microservice
curl http://${host_ip}:3007/v1/chat/completions\
  -X POST \
  -d '{"query":"What is Deep Learning?","max_tokens":17,"top_k":10,"top_p":0.95,"typical_p":0.95,"temperature":0.01,"repetition_penalty":1.03,"stream":true}' \
  -H 'Content-Type: application/json'

```

## 🚀 Test MegaService

```bash
curl http://${host_ip}:3008/v1/searchqna -H "Content-Type: application/json" -d '{
     "messages": "What is the latest news? Give me also the source link.",
     "stream": "True"
     }'
```
