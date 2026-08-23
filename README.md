# Running Qwen3.8-27B with llama.cpp and GitHub Copilot CLI on Windows

This guide documents a working setup for running **Qwen3.8-27B locally with llama.cpp** on a Windows NVIDIA machine and using it remotely from **GitHub Copilot CLI** on a development PC.

> Llama.cpp can take advantage of both VRAM and system RAM to provide a much bigger context than VRAM alone can provide.

The tested machine uses an **NVIDIA RTX 5090 with 32 GB VRAM**. Other NVIDIA GPUs can work, but quantization, context size, and speed may need adjustment.

## 1. Architecture

``` text
Development PC
  GitHub Copilot CLI
        |
        | OpenAI-compatible API over LAN
        v
AI PC: http://<AI-PC-IP>:8080/v1
  llama-server
        |
        v
  Qwen3.8-27B Q4_K_M
        |
        v
  NVIDIA GPU + system RAM
```

The model inference itself is local.

## 2. Install llama.cpp on Windows

Download the latest Windows CUDA build from the official llama.cpp releases:

https://github.com/ggml-org/llama.cpp/releases

For a modern NVIDIA GPU, choose the **Windows x64 CUDA** build. If the release provides a separate matching CUDA DLL/runtime ZIP, download that too.

Create:

``` text
C:\llama.cpp
```

Extract both ZIPs into that directory. You should have files such as:

``` text
llama-server.exe
llama-cli.exe
cudart64_13.dll
cublas64_13.dll
cublasLt64_13.dll
```

### Verify CUDA

``` powershell
cd C:\llama.cpp
.\llama-server.exe --list-devices
```

You want to see your NVIDIA GPU as a CUDA device, for example:

``` text
CUDA0: NVIDIA GeForce RTX 5090
```

## 3. If Windows blocks llama.cpp

Some Windows 11 systems may report:

``` text
An Application Control policy has blocked this file.
```

First try:

``` powershell
Get-ChildItem C:\llama.cpp -Recurse | Unblock-File
```

To inspect Windows Application Control policies, run an elevated PowerShell:

``` powershell
CiTool.exe -lp
```

If this appears:

``` text
Friendly Name: VerifiedAndReputableDesktop
Is Currently Enforced: true
```

then Smart App Control is active.

Its UI is under:

``` text
Windows Security
  -> App & browser control
  -> Smart App Control settings
```

**Important:** Understand the Windows implications before disabling Smart App Control. On some installations, manually turning it off cannot be trivially reversed without resetting/reinstalling Windows.

## 4. Download and run Qwen3.8-27B

llama.cpp can download GGUF models directly from Hugging Face, so the `hf` command-line utility is not required.

The model used here is:

``` text
ggml-org/Qwen3.8-27B-GGUF:Q4_K_M
```

Start it with:

``` powershell
cd C:\llama.cpp

.\llama-server.exe `
  -hf ggml-org/Qwen3.8-27B-GGUF:Q4_K_M `
  -c 131072 `
  -np 1 `
  --jinja `
  -fa on `
  --host 0.0.0.0 `
  --port 8080 `
  --alias qwen38-27b
```

Meaning:

  Argument               Purpose

---------------------- --------------------------------------------

  `-hf ...`              Downloads/loads the GGUF from Hugging Face
  `-c 131072`            128K context
  `-np 1`                One concurrent context slot
  `--jinja`              Jinja chat/tool templates
  `-fa on`               Flash Attention
  `--host 0.0.0.0`       Accept LAN connections
  `--port 8080`          API port
  `--alias qwen38-27b`   Simple model ID exposed to clients

The first launch downloads the model. Later launches use the cached
copy.

## 5. Verify the server

Wait until llama-server reports that the model is loaded and listening.

Then:

``` powershell
curl.exe http://localhost:8080/v1/models
```

Verify that `qwen38-27b` appears and that the active context is approximately:

``` text
131072
```

The OpenAI-compatible base URL is:

``` text
http://localhost:8080/v1
```

The Chat Completions endpoint is:

``` text
http://localhost:8080/v1/chat/completions
```

## 6. Check VRAM

``` powershell
nvidia-smi
```

llama.cpp can place model data in both VRAM and system RAM when necessary. Large context windows also consume memory, so model quantization, context size, and GPU residency are a tradeoff.

To inspect llama-server system-memory use:

``` powershell
Get-Process llama-server | Select-Object `
    ProcessName,
    @{N='RAM_GB';E={[math]::Round($_.WorkingSet64/1GB,2)}},
    @{N='Private_GB';E={[math]::Round($_.PrivateMemorySize64/1GB,2)}}
```

## 7. Allow LAN access through Windows Firewall

If Copilot CLI runs on another machine, allow inbound TCP 8080 on the AI PC.

Run elevated:

``` powershell
New-NetFirewallRule `
  -DisplayName "llama.cpp Server" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 8080 `
  -Action Allow `
  -Profile Private
```

Restrict this to trusted/private networks. Do not expose an unauthenticated llama-server directly to the public Internet.

## 8. Find the AI PC address

``` powershell
ipconfig
```

Find the IPv4 address of the active network adapter, for example:

``` text
192.168.1.23
```

From the development PC:

``` powershell
curl.exe http://192.168.1.23:8080/v1/models
```

If that returns the model list, networking is ready.

## 9. Configure GitHub Copilot CLI

Copilot CLI supports OpenAI-compatible custom providers. You must set environment variables before opening PowerShell and running Copilot CLI

On the **development PC**, set:

``` powershell
$env:COPILOT_PROVIDER_TYPE="openai"
$env:COPILOT_PROVIDER_BASE_URL="http://192.168.1.23:8080/v1"
$env:COPILOT_MODEL="qwen38-27b"
```

Replace `192.168.1.23` with the actual AI-PC address.

An API key is not normally required for an unauthenticated local llama.cpp server.

**Use `http`, not `https`, unless you have explicitly configured TLS.**

## 10. Make the Copilot settings persistent

PowerShell `$env:` assignments disappear with that process. To save the settings for the current Windows user:

``` powershell
[Environment]::SetEnvironmentVariable(
    "COPILOT_PROVIDER_TYPE",
    "openai",
    "User"
)

[Environment]::SetEnvironmentVariable(
    "COPILOT_PROVIDER_BASE_URL",
    "http://192.168.1.23:8080/v1",
    "User"
)

[Environment]::SetEnvironmentVariable(
    "COPILOT_MODEL",
    "qwen38-27b",
    "User"
)
```

Open a PowerShell terminal in your repo folder.

Verify:

``` powershell
$env:COPILOT_PROVIDER_TYPE
$env:COPILOT_PROVIDER_BASE_URL
$env:COPILOT_MODEL
```

Then run

```
Copilot
```

> Tip: You probably want to run /yolo on first load so you're not constantly granting permissions.

## 11. Why model choice matters for Copilot CLI

A coding model can be excellent at generating code and still be poor in an agent harness.

Copilot CLI depends heavily on:

-   streaming
-   tool/function calling
-   reading/searching files
-   shell execution
-   editing files
-   build/test loops
-   large context

GitHub recommends at least a 128K context window for best results with custom models.

Tool-call reliability is therefore just as important as coding benchmarks.

## 12. Vision support

Qwen3.8-27B is multimodal and supports image input.

Its GGUF repositories include a multimodal projector (`mmproj`). Current llama.cpp versions can automatically obtain the projector when using `-hf`.

When loading files manually, a vision setup may require:

``` powershell
--mmproj C:\Models\mmproj-Qwen3.8-27B-BF16.gguf
```

Current llama.cpp also provides `--mmproj-auto` behavior with `-hf`.

In the tested setup, clipboard image input through Copilot CLI worked with Qwen3.8-27B.

## 13. Persistent project memory

Before running Copilot at a particular repo, download the README.md file from https://github.com/carlfranklin/manualcompaction and save it to your repo root as STARTUP.md. This document gives Copilot a superpower to remember where it was in case you run out of context.

This should be your first prompt:

```
Read STARTUP.md and make sure you follow the instructions throughout the session.
```

STARTUP.md instructs the model to keep three small Markdown files in each repository:

``` text
AI_RULES.md
AI_MEMORY.md
AI_CURRENT.md
```

**AI_RULES.md** --- architecture, conventions, permanent constraints and instructions.

**AI_MEMORY.md** --- durable discoveries, decisions, fixes, APIs, dependencies, configuration details, and non-obvious behavior.

**AI_CURRENT.md** --- current task, relevant files, work completed, failures already investigated, unresolved issues, and next steps.

## 14. Context management

A 128K context is the maximum working context at one time; it is not a lifetime allowance.

A fresh Copilot CLI session provides a fresh context while llama-server and the model can remain loaded. 

You do **not** need to restart llama.cpp simply because a Copilot conversation becomes full.

Useful Copilot CLI commands include:

``` text
/context
```

to inspect context usage, and:

``` text
/compact
```

to compact conversation state when supported by the installed Copilot CLI version.

Persistent Markdown memory files make fresh sessions much less painful.

## 15. Useful diagnostics

CUDA devices:

``` powershell
.\llama-server.exe --list-devices
```

Local server:

``` powershell
curl.exe http://localhost:8080/v1/models
```

Remote connectivity:

``` powershell
curl.exe http://192.168.1.23:8080/v1/models
```

VRAM:

``` powershell
nvidia-smi
```

Windows Application Control:

``` powershell
CiTool.exe -lp
```

## 16. Starting after a reboot

On the AI PC:

``` powershell
cd C:\llama.cpp

.\llama-server.exe `
  -hf ggml-org/Qwen3.8-27B-GGUF:Q4_K_M `
  -c 131072 `
  -np 1 `
  --jinja `
  -fa on `
  --host 0.0.0.0 `
  --port 8080 `
  --alias qwen38-27b
```

Wait for the server to finish loading.

On the development PC:

``` powershell
copilot
```

If the environment variables were persisted, Copilot should reconnect to the local model automatically.

## 17. Manual model-file alternative

Instead of `-hf`, models can be stored explicitly under a directory such as:

``` text
C:\Models
```

Then:

``` powershell
.\llama-server.exe `
  -m C:\Models\Qwen3.8-27B-Q4_K_M.gguf `
  --mmproj C:\Models\mmproj-Qwen3.8-27B-BF16.gguf `
  -c 131072 `
  -np 1 `
  --jinja `
  -fa on `
  --host 0.0.0.0 `
  --port 8080 `
  --alias qwen38-27b
```

For a split GGUF, point `-m` at the first shard and keep all shards together with their original names.

## 18. Security notes

This simple setup assumes a trusted LAN.

-   `--host 0.0.0.0` exposes llama-server to the machine's network interfaces.
-   Keep the Windows Firewall rule on the Private profile.
-   Do not port-forward 8080 through the router.
-   Do not expose an unauthenticated endpoint directly to the Internet.
-   Use authentication and/or a secured reverse proxy for less-trusted networks.

## 19. Final working configuration

``` text
Model:          Qwen3.8-27B
Quantization:   Q4_K_M
Server:         llama.cpp / llama-server
Context:        131072 tokens (128K)
Parallel slots: 1
GPU:            NVIDIA RTX 5090 32 GB
API:            OpenAI-compatible
Harness:        GitHub Copilot CLI
Vision:         Supported
```

Server:

``` powershell
.\llama-server.exe `
  -hf ggml-org/Qwen3.8-27B-GGUF:Q4_K_M `
  -c 131072 `
  -np 1 `
  --jinja `
  -fa on `
  --host 0.0.0.0 `
  --port 8080 `
  --alias qwen38-27b
```

Copilot CLI:

``` powershell
$env:COPILOT_PROVIDER_TYPE="openai"
$env:COPILOT_PROVIDER_BASE_URL="http://<AI-PC-IP>:8080/v1"
$env:COPILOT_MODEL="qwen38-27b"
copilot
```

## References

-   llama.cpp: https://github.com/ggml-org/llama.cpp
-   llama.cpp releases: https://github.com/ggml-org/llama.cpp/releases
-   Qwen3.8-27B GGUF: https://huggingface.co/ggml-org/Qwen3.8-27B-GGUF
-   GitHub Copilot CLI BYOK documentation:
    https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-byok-models
-   Manual Compaction Instructions: https://github.com/carlfranklin/manualcompaction
