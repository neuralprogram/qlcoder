# QLCoder
Agentic Framework for Synthesizing CodeQL Queries

## Table of Contents
- [Overview](#overview)
- [Installation](#installation)
  - [Docker (Recommended)](#docker-recommended)
  - [Native (Linux)](#native-linux)
- [Usage](#usage)
- [Quick Start](#quick-start)
- [Development Tooling](#development-tooling)
- [Examples](#examples)
- [Paper Environment](#paper-environment)
- [Contributions](#contributions)
- [Team](#team)
- [Citation](#citation)
- [Affiliated Projects](#affiliated-projects)

## Overview

![QLCoder Iterative Refinement](img/iterative-refinement-figure.png)

QLCoder is a framework for using LLMs to synthesize end-to-end CodeQL queries for vulnerability detection. Given an existing CVE's metadata, LLM, and coding agent, QLCoder iteratively synthesizes a CodeQL query to detect the existing CVE. The starting query is a CodeQL path query template populated by an extracted AST of the diff. While synthesizing the query, the coding agent has access to tools to interface with a RAG database and the CodeQL language server. Afterwards, the query can be used for multivariant analysis, regression testing, or guidance for writing CodeQL queries. 

## Installation

### Docker (Recommended)

#### Step 1: Install CodeQL
Note - In the paper, CodeQL version 2.22.2 was used. However, any version (and language) can be used. QLCoder stores the local CodeQL version's QL packs in the vector database. Paths are configured in `.env`.

Download an appropriate version of the CodeQL Action bundle from the [CodeQL Action releases page](https://github.com/github/codeql-action/releases).

- **For the latest version:**
  Visit the [latest release](https://github.com/github/codeql-action/releases/latest) and download the appropriate bundle for your OS:
  - `codeql-bundle-osx64.tar.gz` for macOS
  - `codeql-bundle-linux64.tar.gz` for Linux

- **For a specific version (e.g., 2.22.2):**
  Go to the [CodeQL Action releases page](https://github.com/github/codeql-action/releases), find the release tagged `codeql-bundle-v2.22.2`, and download the appropriate bundle for your platform.

Extract to `~/codeql` (or another path — update `CODEQL_HOME` in `.env` accordingly):

```sh
tar -xzf codeql-bundle-<platform>.tar.gz -C ~/
```

#### Step 2: Install the CodeQL LSP MCP server
Clone the [CodeQL LSP MCP server](https://github.com/neuralprogram/codeql-lsp-mcp) and build it.

```sh
git clone https://github.com/neuralprogram/codeql-lsp-mcp ~/codeql-lsp-mcp
cd ~/codeql-lsp-mcp
npm install
npm run build
```

#### Step 3: Configure and start services

```sh
cp .env.example .env
echo "APP_UID=$(id -u)" >> .env
echo "APP_GID=$(id -g)" >> .env
```

Fill in your API key and CodeQL paths in `.env`:
```
ANTHROPIC_API_KEY=...

# QL pack paths depend on your CodeQL version.
# Find the version numbers with:
#   ls ~/codeql/qlpacks/codeql/java-queries/   → use for SECURITY_QLPACK_PATH
#   ls ~/codeql/qlpacks/codeql/java-all/        → use for LIBRARY_QLPACK_PATH
SECURITY_QLPACK_PATH=~/codeql/qlpacks/codeql/java-queries/<version>/Security/CWE
LIBRARY_QLPACK_PATH=~/codeql/qlpacks/codeql/java-all/<version>/semmle/code/java
```

Then start the QLCoder app and ChromaDB:
```sh
docker compose up -d
```

#### Step 4: Retrieve CVE repositories

The CVE must be listed in `data/project_info.csv`. This clones the repository at the buggy commit and generates the fix diff.

```sh
docker compose run --rm app python3 scripts/get_cve_repos.py --cve CVE-2025-27818
# or multiple at once:
docker compose run --rm app python3 scripts/get_cve_repos.py --cves CVE-2025-27818,CVE-2025-0851
# process CVEs from a file (one CVE ID per line)
docker compose run --rm app python3 scripts/get_cve_repos.py --cve-file cves.txt
# process all CVEs
docker compose run --rm app python3 scripts/get_cve_repos.py --all
# force regenerate existing diffs
docker compose run --rm app python3 scripts/get_cve_repos.py --cve CVE-2018-9159 --force
```

#### Step 5: Create CodeQL databases

Databases are created with `--build-mode=none` — no build toolchain required.

```sh
# to build a specific CVE's CodeQL databases
docker compose run --rm app python3 scripts/build_codeql_dbs.py --cve-id CVE-2025-27818
```

This creates `cves/CVE-2025-27818/CVE-2025-27818-vul` and `cves/CVE-2025-27818/CVE-2025-27818-fix`.

```sh
# to build all of the fetched CVE repos' CodeQL databases
docker compose run --rm app python3 scripts/build_codeql_dbs.py
```

#### Step 6: Populate RAG database

Run these scripts to populate the vector database. `codeql_docs_fetcher.py` and `cwe_fetcher.py` are one-time setup; `cves_fetcher.py` should be re-run after adding new CVEs.

```sh
docker compose run --rm app python3 scripts/codeql_docs_fetcher.py
docker compose run --rm app python3 scripts/cwe_fetcher.py
docker compose run --rm app python3 scripts/cves_fetcher.py
```

### Native (Linux)

#### Step 1: Install CodeQL
Note - In the paper, CodeQL version 2.22.2 was used. However, any version (and language) can be used. QLCoder stores the local CodeQL version's QL packs in the vector database. Paths are configured in `.env`.

Download an appropriate version of the CodeQL Action bundle from the [CodeQL Action releases page](https://github.com/github/codeql-action/releases).

- **For the latest version:**
  Visit the [latest release](https://github.com/github/codeql-action/releases/latest) and download the appropriate bundle for your OS:
  - `codeql-bundle-linux64.tar.gz` for Linux

- **For a specific version (e.g., 2.22.2):**
  Go to the [CodeQL Action releases page](https://github.com/github/codeql-action/releases), find the release tagged `codeql-bundle-v2.22.2`, and download the appropriate bundle for your platform.

After downloading, extract the archive in the project root directory:

```sh
tar -xzf codeql-bundle-<platform>.tar.gz
```

This should create a sub-directory `codeql/` with the executable `codeql` inside.

Add the path of this executable to your `PATH` environment variable:

```sh
export PATH="$PWD/codeql:$PATH"
```

#### Step 2: Install the CodeQL LSP MCP server
Clone the [CodeQL LSP MCP server](https://github.com/neuralprogram/codeql-lsp-mcp) and build it.

```sh
git clone https://github.com/neuralprogram/codeql-lsp-mcp
cd codeql-lsp-mcp
npm install
npm run build
```

#### Step 3: Setup Conda environment
```sh
conda env create -f environment.yml
conda activate qlcoder
```

#### Step 4: Configure `.env`

```sh
cp .env.example .env
```

Fill in your API key and CodeQL paths in `.env`:
```
ANTHROPIC_API_KEY=...
CODEQL_HOME=~/codeql
CODEQL_LSP_MCP_HOME=~/codeql-lsp-mcp

# QL pack paths depend on your CodeQL version.
# Find the version numbers with:
#   ls ~/codeql/qlpacks/codeql/java-queries/   → use for SECURITY_QLPACK_PATH
#   ls ~/codeql/qlpacks/codeql/java-all/        → use for LIBRARY_QLPACK_PATH
SECURITY_QLPACK_PATH=~/codeql/qlpacks/codeql/java-queries/<version>/Security/CWE
LIBRARY_QLPACK_PATH=~/codeql/qlpacks/codeql/java-all/<version>/semmle/code/java
```

#### Step 5: Retrieve CVE repositories

The CVE must be listed in `data/project_info.csv`. This clones the repository at the buggy commit and generates the fix diff.

```sh
python3 scripts/get_cve_repos.py --cve CVE-2025-27818
# or multiple at once:
python3 scripts/get_cve_repos.py --cves CVE-2025-27818,CVE-2025-0851
# process CVEs from a file (one CVE ID per line)
python3 scripts/get_cve_repos.py --cve-file cves.txt
# process all CVEs
python3 scripts/get_cve_repos.py --all
# force regenerate existing diffs
python3 scripts/get_cve_repos.py --cve CVE-2018-9159 --force
```

#### Step 6: Create CodeQL databases

Databases are created with `--build-mode=none` — no build toolchain required.

```sh
# to build a specific CVE's CodeQL databases
python3 scripts/build_codeql_dbs.py --cve-id CVE-2025-27818
```

```sh
# to build all of the fetched CVE repos' CodeQL databases
python3 scripts/build_codeql_dbs.py 
```

This creates `cves/CVE-2025-27818/CVE-2025-27818-vul` and `cves/CVE-2025-27818/CVE-2025-27818-fix`.

#### Step 7: Start ChromaDB

Start ChromaDB in a separate terminal and keep it running for this step and whenever running the agent.

```sh
chroma run --path data/chroma_db
```

#### Step 8: Populate RAG database

Run these scripts to populate the vector database. `codeql_docs_fetcher.py` and `cwe_fetcher.py` are one-time setup; `cves_fetcher.py` should be re-run after adding new CVEs.

```sh
python3 scripts/codeql_docs_fetcher.py
python3 scripts/cwe_fetcher.py
python3 scripts/cves_fetcher.py
```

## Quick Start
After following the Installation instructions, the quick start walks through an example of synthesizing a CodeQL query for a given CVE.

1. Retrieve the CVE repository and CVE fix diff.
```sh
python3 scripts/get_cve_repos.py --cve CVE-2025-27818
```
2. Create CodeQL databases for the CVE.
```sh
python3 scripts/build_codeql_dbs.py --cve-id CVE-2025-27818
```
3. Populate or update the RAG database.
```sh
python3 scripts/cves_fetcher.py
```
4. Run the pipeline.
```sh
./run_cve.sh CVE-2025-27818
```

Additional options can be passed after the CVE ID:
```sh
./run_cve.sh CVE-2025-27818 --model sonnet-4.5 --max-iteration 10
```
## Usage
Below are the available configurations for QLCoder.

> **Timeout:** Each agent context window has a default shell timeout (e.g. 300s). Increase the timeout in the relevant backend's execution method if needed when running into "Context window failed" errors.

> **Note:** Agent support is tested against the versions listed in [Paper Environment](#paper-environment). Newer versions of coding agents may require updates to the backend. PRs adding support for newer versions, other coding agents, and more models are welcome!

**Models** (`--model`): `sonnet-4` (default), `sonnet-4.5` (Claude); `gemini-2.5-pro`, `gemini-2.5-flash` (Gemini); `gpt-5` (Codex) 

**Agents** (`--agent`): `claude` (default), `gemini` (Gemini CLI), `codex` (OpenAI models and open source models)

**Ablation modes** (`--ablation-mode`):

| Mode | Description | Available Agents
|---|---| --- |
| `full` | All QLCoder tools enabled (default) and AST extraction | Claude Code, Codex (GPT, GPT-OSS), Gemini 
| `no_tools` | No tools and no AST extraction | Claude Code, Codex (GPT, GPT-OSS), Gemini 
| `no_lsp` | No CodeQL LSP tools | Claude Code
| `no_docs` | No CodeQL documentation retrieval | Claude Code
| `no_ast` | No AST extraction from diff | Claude Code

### Model-specific notes (GPT-5)
By default we set the reasoning effort to medium. You can override this in [`codex_backend.py`](src/agent_backends/codex_backend.py#L365).

### No Tools, No Docs mode (Paper specifics)
When Chroma isn't used for fetching the CVE description, a pre-fetched description is injected directly into the prompt via `task.cve_description`. Use `scripts/cves_fetcher.py` to populate a local JSON file of descriptions:

```sh
python scripts/cves_fetcher.py --descriptions-file data/cve_descriptions.json 
```

The file maps CVE IDs to their CVE description strings and is appended to on each run (existing entries are skipped). When running with `--ablation-mode no_tools` or `--ablation-mode no_docs`, QLCoder automatically loads this file and sets `task.cve_description` for the CVE being analyzed.

## Development Tooling 
The following tools are recommended while using QLCoder: 

[Delete collections from QLCoder runs](scripts/delete_cve_analysis_collections.py) - to clean up Chroma, here is a script to delete collections from using QLCoder.

[chromadb-ops](https://github.com/amikos-tech/chromadb-ops) - CLI tool for inspecting and maintaining Chroma. 

```sh
# useful for cleaning up chroma
chops db clean data/chroma_db
```

## Examples 
### QLCoder Generated Query Examples 
- [Java](query-examples-java)
- [C](query-examples-c)

### QLCoder MCP Configurations 
Here are examples of MCP configurations when using QLCoder. The configuration should be similar to these files in the agent's workspace. 
- [Claude Code](example-mcp-configs/claude_config.json)
- [Codex](example-mcp-configs/codex_config.toml)
- [Gemini CLI](example-mcp-configs/gemini_settings.json)

## Paper Environment
The following versions were used to produce the results in the QLCoder paper.

| Tool | Version |
|---|---|
| CodeQL | 2.22.2 |
| Claude Code | 1.0.120 |
| Gemini CLI | 0.6.0 |
| Codex CLI | 0.38.0 |

## Contributions
We welcome any contributions, pull requests, or issues!
If you would like to contribute, please either file a new pull request or issue. Feel free to take on an existing issue too.

## Team 
QLCoder is collaborative effort between researchers at Cornell University, Johns Hopkins University, and the University of Pennsylvania. Please reach out to us if you have any questions. 

[Claire Wang](https://clairewang.net) - CS PhD Student at University of Pennsylvania

[Ziyang Li](https://liby99.github.io/) - Professor at Johns Hopkins University

[Saikat Dutta](https://www.cs.cornell.edu/~saikatd/) - Professor at Cornell University 

[Mayur Naik](https://www.cis.upenn.edu/~mhnaik/) - Professor at University of Pennsylvania

## Citation
Consider citing our ICLR'26 paper:
```
@misc{wang2025qlcoderquerysynthesizerstatic,
      title={QLCoder: A Query Synthesizer For Static Analysis of Security Vulnerabilities}, 
      author={Claire Wang and Ziyang Li and Saikat Dutta and Mayur Naik},
      year={2025},
      eprint={2511.08462},
      archivePrefix={arXiv},
      primaryClass={cs.CR},
      url={https://arxiv.org/abs/2511.08462}, 
}
```
[Arxiv Link](https://arxiv.org/abs/2511.08462)

## Affiliated Projects 
The following are projects affiliated with the QLCoder authors. Feel free to check them out. 

- [IRIS](https://github.com/iris-sast/iris) - LLM-identified sources/sinks appended to existing CodeQL security queries for a given repository. QLCoder is an extension of some of the IRIS ideas. [Arxiv Link](https://arxiv.org/abs/2405.17238)
- [CWE-Bench-Java](https://github.com/iris-sast/iris/tree/v2/data) - Benchmark of Java security vulnerabilities containing CVE metadata, repos, and source/sink labels.