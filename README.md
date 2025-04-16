# 🧪 Garak AI - LLM Security Testing

This repo sets up a testing environment for [Garak](https://github.com/NVIDIA/garak), a vulnerability scanner for LLMs, with support for running tests against hosted models, or models that have a REST APIs.
It uses [OpenWebUI](https://github.com/open-webui/open-webui) for a front end to [Ollama](https://ollama.com) which is a tool that allows users to run large language models (LLMs) locally on their own computers.


## 🚀 File & Directory Structure

```text
├── docker-compose.yml             # Spins up the full test environment (Garak + model)
├── garak/                         # All things Garak, including Dockerfile and config
│   ├── Dockerfile.garak           # Dockerfile for building the Garak image
│   ├── local_images               # Pre-built Docker images for different OS platforms
│   ├── ollama_generator/          # Config for using Garak with Ollama
│   │   └── ollama_options.json    # JSON config file pointing Garak to the Ollama host
│   └── rest_generator/            # Config for using Garak with REST-based LLM APIs
│       ├── rest_request.json      # REST generator config (used with llama for demo)
│       └── README                 # Instructions for creating and using the REST generator
├── README.md                      # You're here! Overview of the project
└── results/                       # Output directory for test reports
```

## Prereqs to Testing

**Install Docker & Docker Compose**
- Follow the Docker docs to [Get Docker](https://docs.docker.com/get-started/get-docker/)

**Adjust Permissions**

The results are provided through a bind mount for easy viewing on the host computer. However, the Garak container runs with UID 1001 and will need write access to the results directory. Running the command below will allow any user in any group to read and write to the directory. Only do this for testing.

```sh
chmod 777 ./results
```

**Bring up the containers**

This can be done using a pre-built image (good for those behind a corporate proxy), or you can let docker compose build it for you.

docker compose build (preferred method)
```sh
docker compose up --build -d

```
To use a pre-built image follow the steps below

```sh
# build the image manually.
docker buildx build -f garak/Dockerfile.garak -t garak_testing-garak --load .

# save it to a tar
docker save garak_testing-garak:latest > linux-garak.tar 

# move it to garak/local_images/garak.tar on the host computer
mv /src/path/linux-garak.tar garak/local_images/linux-garak.tar 

# load the image into docker
docker load --input garak/local_images/garak.tar

# start the containers
docker compose up -d

```

**Set Up OpenWebUI**
- Navigate to http://localhost:3000
- Create a testing account
- Log into the testing account for the next steps

**Download Model(s) for Testing**

The compose file has a helper container that preloads the llama3 model, but if you want to use another one, follow the steps below.
- Click on the test account on the top right
- Select `Admin panel`
- Select the `Settings` tab at the top
- Choose the `Models` tab on the left
- Click the download icon
- In the URI field to pull a model from Ollama.com enter `llama3.2`
  - If you want another model instead, Ollama has [many to choose from](https://ollama.com/library) (just check the license first)
- Click download
- Start a new chat with the selected model

**Corporate**
- If you're behind a corporate proxy and/or using WSL you may have to fiddle with some of the commands and config files

## What You Need to Know to Use Garak Effectively

| **Component** | **What it does**                         | **Why you care**                         |
|---------------|------------------------------------------|------------------------------------------|
| **[Probes](https://github.com/NVIDIA/garak/tree/main/garak/probes)**    | Provide the attacks/payloads             | Define _what_ is being tested            |
| **[Generators](https://github.com/NVIDIA/garak/tree/main/garak/generators)**| Connect to the model                     | Define _how_ you talk to it              |
| **[Detectors](https://docs.garak.ai/garak/garak-components/understanding-detectors)** | Evaluate model responses                 | Define _what counts as failure_          |
| **[Evaluators](https://docs.garak.ai/garak/garak-components/scan-evaluation)**| Score performance                        | Optional, more nuanced metrics           |
| **[Reports](https://reference.garak.ai/en/latest/report.html)**   | Save outputs                             | Useful for audits & dashboards           |

## How to Run Garak Probes Against Your Installed LLM

**Log into the garak container**

```sh
docker exec -it garak /bin/bash
```
**Run Some Garak Tests**

Pick a generator, choose your probe, and fire it at the model!

> ℹ️ **Note:** LLMs are inconsistent — they don’t always give the same answer twice. That’s where the `--generations` flag comes in. It tells Garak how many times to run each test. The default is 10 (which is great for thoroughness but kind of a time hog). Somewhere around 3–4 is usually enough to catch interesting stuff without waiting forever, and 1-2 is great for testing.

Here’s an example command to get you rolling:
```sh
garak --model_type ollama \
      --model_name llama3.2:latest \
      --probes xss \
      --generations 2 \
      --generator_option_file /app/ollama_options.json \
      --report_prefix /app/results/$(date +%Y-%m-%dT%H%M%S) \
      --verbose
```
Happy probing 😈


**Viewing the Results**

When all probe tests have finished, two reports will be generated:

📜 report closed :) /app/results/2025-04-12T092409-.report.jsonl

📜 report html summary being written to /app/results/2025-04-12T092409-.report.html

These will be moved to the results directory in the repository directory on the host computer.

The HTML report is bare bones and has very high level results.

The jsonl report contains all the tests run and the result of each. You can format this file in a different program, or if you just want to grep it from the CLI you can use jq

```sh
jq -c '.' /path/to/workspace/garak_demo/results/2025-04-12T082159.hitlog.jsonl | jq
```

**Shut It Down**

Once you're done testing, you'll want to stop all the running services.

```sh
docker compose down --remove-orphans
```

## Testing REST Services That Don't Have Native Support

Head over to the [REST README](./garak/rest_generator/README.md) to learn how to build custom REST requests that Garak can use to run the tests.

## Common Garak Commands

You'll need to be logged into the garak container for these to work

```sh
## Log into the garak container
docker exec -it garak /bin/bash

## List Loaded Probes.
## The stars 🌟 indicate a whole plugin
garak --list_probes

## List Supported Generators.
garak --list_generators

## Run All Probes (Danger Zone™)
## 🛑 Heads-up: this takes a while and can generate a lot of output.
garak --model_type ollama \
      --model_name llama3.2:latest \
      --probes all \
      --generations 3 \
      --generator_option_file /app/ollama_options.json \
      --report_prefix /app/results/$(date +%Y-%m-%dT%H%M%S) \
      --verbose
```
