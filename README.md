# Simple macOS AI Query Script (macOS 10.12 – macOS 12)

This repository provides a lightweight Bash script (`ask`) that enables **local AI querying with text-to-speech output** on **older macOS versions from macOS 10.12 (Sierra) to macOS 12 (Monterey)** — systems that are now deprecated and cannot run modern AI tools like Ollama natively.

The solution uses **[Multipass](https://multipass.run)** to run an **Ubuntu VM** on your Mac, where **Ollama** serves a compact 1.5B parameter model. Your macOS host sends prompts via `curl`, receives clean responses, prints them, and speaks them aloud using the built-in `say` command.

---

## Features

- Fully functional on **macOS 10.12 to macOS 12**
- Runs **Ollama + AI model in an isolated Ubuntu VM**
- Uses a **fast, distilled 1.5B model** (`DeepSeek-R1-Distill-Qwen-1.5B`)
- **Removes `<think>...</think>` reasoning tags** automatically
- **Speaks answers aloud** with macOS `say`
- Simple one-line usage: `ask "Explain quantum entanglement"`

---

## Prerequisites

| Requirement | Notes |
| --- | --- |
| **macOS** | macOS 10.12 Sierra → macOS 12 Monterey |
| **Multipass** | Compitable with macOS 10.12–12 |
| **jq** | Required for parsing JSON responses |
| **RAM** | ≥8GB recommended (4GB VM + host) |
| **Disk** | ≥20GB free |
| **Terminal** | Built-in Terminal.app |

---

## Installation & Setup

### Install Multipass

Download a compatible version from the [Multipass GitHub Releases](https://github.com/canonical/multipass/releases):

```bash
# Example: Download v1.5.0 (works on macOS 10.12)
curl -L https://github.com/canonical/multipass/releases/download/v1.5.0/multipass-1.5.0+mac-Darwin.pkg -o multipass-1.5.0+mac-Darwin.pkg
sudo installer -pkg multipass-1.5.0+mac-Darwin.pkg -target /
```

Verify:

```bash
multipass version
```

Launch Ubuntu VM:

```bash
multipass launch --name primary --mem 16G --disk 1000G
```

> Tip: Use `primary` as the name — the script assumes this.  
> You can change it later by editing the script.

---

### Install Ollama in the VM

```bash
multipass sh
```

Inside the VM:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Download the model (~900MB)
ollama pull erwan2/DeepSeek-R1-Distill-Qwen-1.5B

# Start Ollama server in background
nohup ollama serve > ollama.log 2>&1 &
```

**Binding Ollama to** `0.0.0.0` **is essential to ensure that the service is accessible from the host macOS system when running inside a Multipass virtual machine.**

```bash
export OLLAMA_HOST=0.0.0.0:11434
```

---

### Install `jq`

The `ask` script **depends on** `jq` to parse JSON output from the Ollama API. Without it, the script cannot extract the AI’s response.

We are working on **older macOS versions**. On these systems, common package managers such as **Homebrew**, **MacPorts**, and **Anaconda/Conda** may not be reliable or even supported, so alternative installation methods are required.

---

#### 🔹 Option 1: Download Precompiled Binary

Go to the [jq GitHub Releases page](https://github.com/jqlang/jq/releases).

Download the macOS binary (e.g., `jq-osx-amd64`).

Install it manually:

```bash
curl -L -o jq https://github.com/jqlang/jq/releases/download/jq-1.5/jq-osx-amd64
chmod +x jq
sudo mv jq /usr/local/bin/
```

Verify:

```bash
jq --version
```

---

#### 🔹 Option 2: Use the Conda prebuilt tarball

That tarball is essentially a precompiled Conda package. By extracting it manually, you can grab the `jq` binary without needing Conda itself. It’s a neat workaround for older macOS systems where package managers are tricky.

Download the prebuilt package directly:

```bash
curl -L -o jq-1.5-0.tar.bz2 https://anaconda.org/conda-forge/jq/1.5/download/osx-64/jq-1.5-0.tar.bz2
```

Extract it:

```bash
tar -xvjf jq-1.5-0.tar.bz2
```

This will create a directory structure containing `bin/jq`.

Move the binary into your PATH (e.g., `/usr/local/bin`):

```bash
chmod +x bin/jq
sudo mv bin/jq /usr/local/bin/jq
```

Verify installation:

```bash
jq --version
```

You should see `jq-1.5`.

---

## The `ask` Script

Save this as `ask` in your project folder:

```bash
#!/bin/bash

# Default values
VM_NAME="primary"

# Dynamically retrieve the VM IP
VM_IP=$(multipass info "$VM_NAME" | grep IPv4 | awk '{print $2}')
if [ -z "$VM_IP" ]; then
    echo "Error: Failed to get VM IP for '$VM_NAME'. Check if VM exists and is running."
    exit 1
fi

MODEL="erwan2/DeepSeek-R1-Distill-Qwen-1.5B"
VOICE=""

# Parse command-line flags
while [ $# -gt 0 ]; do
    case $1 in
        --vm)
            VM_NAME="$2"
            shift 2
            ;;
        --model)
            MODEL="$2"
            shift 2
            ;;
        --voice)
            VOICE="$2"
            shift 2
            ;;
        *)
            PROMPT="$1"
            shift
            ;;
    esac
done

if [ -z "$PROMPT" ]; then
    echo "Usage: ask [--vm <VM_NAME>] [--model <MODEL_NAME>] [--voice <VOICE_NAME>] 'your question'"
    exit 1
fi

# Send request to Ollama API
RESPONSE=$(curl -s -X POST "http://$VM_IP:11434/api/generate" \
    -H "Content-Type: application/json" \
    -d "{\"model\": \"$MODEL\", \"prompt\": \"$PROMPT\", \"stream\": false}" \
    | jq -r '.response' \
    | perl -0777 -pe 's#<think>.*?</think>##gs' \
    | awk 'NF')

# Error handling
if [ "$RESPONSE" = "null" ] || [ -z "$RESPONSE" ]; then
    echo "Error: Failed to get response."
    echo "Check:"
    echo "  • VM is running: multipass list"
    echo "  • Ollama is serving: multipass exec $VM_NAME -- curl http://localhost:11434"
    echo "  • Model exists: multipass exec $VM_NAME -- ollama list"
    exit 1
fi

# Output + speak with selected voice
echo "$RESPONSE"
if [ -n "$VOICE" ]; then
    say -v "$VOICE" "$RESPONSE"
else
    say "$RESPONSE"
fi
```

---

## Make It Executable

```bash
chmod +x ask
```

Optional: Move to `/usr/local/bin` for global use:

```bash
sudo mv ask /usr/local/bin/
```

---

## Usage

**Input example**:

```bash
ask "赵达功更接近兵勇还是贼匪" --voice Ting-Ting --model erwan2/DeepSeek-R1-Distill-Qwen-1.5B
```

**Output example**:

```
赵达功更接近“贼匪”一类，不是“兵勇”的正面硬拼，而是“贼匪”的镰刀手法—— 避实击虚、暗中割裂、借力杀人。
```

**Input example**:

```bash
ask "ラインハルトのペンダントって何が入ってるの" --voice Kyoko --model schroneko/gemma-2-2b-jpn-it
```

**Output example**:

```
ラインハルトのペンダントには、彼の親友であり側近だったジークフリード・キルヒアイスの遺髪が入っています。
```

*(And your Mac speaks it aloud)*

---

## Acknowledgments

- [Ollama](https://ollama.com) – Local LLM server
- [Multipass](https://multipass.run) – Instant Ubuntu VMs
- Model: [`erwan2/DeepSeek-R1-Distill-Qwen-1.5B`](https://ollama.com/erwan2/DeepSeek-R1-Distill-Qwen-1.5B)

---

**Enjoy AI on your vintage Mac.**  
*No cloud. No subscription. Just local, private, and spoken answers.*
