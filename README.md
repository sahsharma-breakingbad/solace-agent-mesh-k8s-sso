# Solace Agent Mesh (SAM) K8s configuration

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Components](#components)
  - [Solace Broker](#solace-broker)
  - [Core SAM Deployment](#core-sam-deployment)
  - [Built-in Agents](#built-in-agents)
  - [Configuration](#configuration)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Configuration Options](#configuration-options)
  - [Environment Variables](#environment-variables)
- [Accessing the Web UI](#accessing-the-web-ui)
- [Customization](#customization)
  - [Adding New Agents](#adding-new-agents)
  - [Scaling](#scaling)
- [Troubleshooting](#troubleshooting)
  - [Checking Logs](#checking-logs)
  - [Restart pod](#restart-pod)
  - [Delete pod](#delete-pod)
  - [Common Issues](#common-issues)

This repository contains Kubernetes configuration files for deploying and managing Solace Agent Mesh (SAM) on AKS via K8s deployment files. 

## Overview

Solace Agent Mesh is a framework for deploying, managing, and orchestrating AI agents. The system consists of:

- **Orchestrator Agent**: Manages tasks and coordinates multi-agent workflows
- **Web UI Gateway**: Provides a user interface for interacting with the agents
- **Built-in Specialized Agents**:
  - **Markdown Creator**: Converts various file types to Markdown
  - **Mermaid Diagram Generator**: Creates PNG images from Mermaid diagram syntax
  - **Web Agent**: Fetches content from web URLs
- **Custom Agents**:
  - **Native SAM**: Python based scaffolded using the SAM agent builder
  - **MCP**: Any MCP agent built with the MCP protocol
  - **A2A**: Agents built with the A2A protocols
  - **AWS Bedrock**: AWS Bedrock agents

More information on Solace Agent Mesh on this github repo https://github.com/SolaceLabs/solace-agent-mesh

## Repository Structure

```
.
├── bultin-agents/
│   └── built-in-agents.yaml    # Deployment for core agents
├── core-sam/
│   └── deployment_basic.yaml   # Main SAM deployment
├── configmap.yaml              # Configuration for agents and services
├── secret.yaml                 # Secret configuration (API keys, credentials)
└── service.yaml                # Kubernetes service definition
```

## Components

### Solace Broker

A deployment of the standard edition Solace message broker that provides the messaging infrastructure for the agent mesh. The broker is deployed as a Kubernetes service and is accessible to other pods within the cluster.

### Core SAM Deployment

The main Solace Agent Mesh deployment that runs the orchestrator and web UI.

### Built-in Agents

A separate deployment that runs specialized agents:

1. **Markdown Creator**: Converts files (PDF, DOCX, XLSX, HTML, CSV, PPTX, ZIP) to Markdown format
2. **Mermaid Diagram Generator**: Creates PNG images from Mermaid diagram syntax
3. **Web Agent**: Fetches and processes content from web URLs

### Configuration

The system is configured through:

- **ConfigMap**: Contains configuration for logging, broker connections, models, and agent definitions
- **Secret**: Stores sensitive information like API keys, credentials, and endpoints

## Prerequisites

- Kubernetes cluster
- kubectl configured to communicate with your cluster
- Access to the Solace Agent Mesh container image
- Namespace created on AKS

## Deployment

1. Create a `secret.yaml` file with the necessary environment variables:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: solace-agent-mesh-secret
   type: Opaque
   data:
     # Replace these placeholders with your actual base64-encoded values
     # You can generate them with: echo -n 'your-value' | base64
     LLM_SERVICE_ENDPOINT: ""           # e.g., https://api.openai.com/v1
     LLM_SERVICE_API_KEY: ""            # Your LLM service API key
     LLM_SERVICE_PLANNING_MODEL_NAME: "" # e.g., openai/gpt-4o
     LLM_SERVICE_GENERAL_MODEL_NAME: "" # e.g., openai/gpt-4o
     NAMESPACE: ""                      # e.g., sam-infra
     SOLACE_BROKER_URL: ""              # e.g., ws://localhost:8008 or for in-cluster software broker: ws://solace-broker-service:8008
                                        # Base64 encoded value for software broker: d3M6Ly9zb2xhY2UtYnJva2VyLXNlcnZpY2U6ODAwOAo=
     SOLACE_BROKER_VPN: ""              # e.g., default
     SOLACE_BROKER_USERNAME: ""         # e.g. admin
     SOLACE_BROKER_PASSWORD: ""         # e.g. admin
     SOLACE_DEV_MODE: ""                # true or false
     SESSION_SECRET_KEY: ""             # Secret key for session management, any key here
     FASTAPI_HOST: ""                   # e.g., 127.0.0.1
     FASTAPI_PORT: ""                   # e.g., 8000
     FASTAPI_HTTPS_PORT: ""             # e.g., 8443
     SSL_KEYFILE: ""                    # Path to SSL key file (if using HTTPS). Keep empty if not used
     SSL_CERTFILE: ""                   # Path to SSL certificate file (if using HTTPS). Keep empty if not used
     SSL_KEYFILE_PASSWORD: ""           # Password for SSL key file (if needed)
     ENABLE_EMBED_RESOLUTION: ""        # e.g., True
     LOGGING_CONFIG_PATH: ""            # e.g., configs/logging_config.ini
   ```

> [!NOTE]
> The `secret.yaml` file is included in `.gitignore` to prevent sensitive information from being committed to the repository.

2. Encode your values using base64:
   ```bash
   # Example for encoding values
   echo -n 'your-value' | base64
   ```

3. Apply the Kubernetes configurations:
   ```bash
   kubectl apply -f configmap.yaml
   kubectl apply -f secret.yaml
   kubectl apply -f service.yaml
   kubectl apply -f solace-broker/broker.yaml
   kubectl apply -f core-sam/deployment_basic.yaml
   kubectl apply -f bultin-agents/built-in-agents.yaml
   ```

3. Verify the deployment:
   ```bash
   kubectl get pods
   kubectl get services
   ```

## Configuration Options

### Environment Variables

Key environment variables that can be configured in the `secret.yaml` file:

- `LLM_SERVICE_ENDPOINT`: API endpoint for the LLM service
- `LLM_SERVICE_API_KEY`: API key for the LLM service
- `LLM_SERVICE_PLANNING_MODEL_NAME`: Model name for planning tasks
- `LLM_SERVICE_GENERAL_MODEL_NAME`: Model name for general tasks
- `NAMESPACE`: Namespace for the agents
- `SOLACE_BROKER_URL`: URL for the Solace message broker
- `SOLACE_BROKER_VPN`: Solace message VPN
- `SOLACE_BROKER_USERNAME`: Username for the Solace broker
- `SOLACE_BROKER_PASSWORD`: Password for the Solace broker
- `SESSION_SECRET_KEY`: Secret key for session management
- `FASTAPI_HOST`: Host for the FastAPI server
- `FASTAPI_PORT`: Port for the FastAPI server

## Accessing the Web GUIs

Once deployed, the SAM Web UI gateway and the Solace Broker Admin console can be accessed through the LoadBalancer service:

```bash
kubectl get service solace-agent-mesh-svc -n sam-infra
kubectl get service solace-broker-admin-svc -n sam-infra

```

Use the external IP and port 8000, and 8008 to access the Web UI and the broker management console respectively.


> [!TIP]
> You can port-forward the access to the WebUI GUI to your local host as follows `kubectl port-forward svc/solace-agent-mesh-svc 8000:8000 -n sam-infra` and `kubectl port-forward <broker_pod_name> 8080:8080 -n sam-infra`

> [!CAUTION]
> Do not use port forwarding in production!


## Customization

### Adding New Agents

To add new agents, update the `configmap.yaml` file with the agent configuration in the `core_agents.yaml` section.

### Scaling

To scale the number of replicas, modify the `replicas` field in the deployment files.

## Troubleshooting

### Checking Logs

```bash
# For the main deployment
kubectl logs -l app=solace-agent-mesh

# For the core agents
kubectl logs -l app=sam-core-agents
```
### Restart pod
```
kubectl delete pod <pod-name> -n <namespace>
```

### Delete pod
```
kubectl delete -f deployment.yaml
```

### Common Issues

- **Connection Issues**: Ensure the Solace broker URL and credentials are correct
- **API Key Issues**: Verify that the LLM service API keys are valid and properly encoded
