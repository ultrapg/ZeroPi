# ZeroPi: Portable Pi Coding Agent Suite

ZeroPi is a portable, self-contained Rust executable designed for Windows to download, configure, and orchestrate the lifecycle of the **Pi coding agent** alongside a **llama.cpp** backend server, keeping everything strictly isolated in a single directory.

---

## Key Features

1. **Strict Portability & Isolation**
   - Zero dependencies on system-wide software.
   - Automatically downloads and installs a portable Node.js runtime and llama.cpp backend inside the folder.
   - Sets up the Pi agent's home configurations locally, keeping your system clean.

2. **Advanced Process Lifecycle Management (Windows Job Objects)**
   - Utilizes standard Windows FFI bindings for **Job Objects**.
   - Spawns child processes (`llama-server.exe` and the `node.exe` Pi agent) inside a shared Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
   - **Graceful Termination**: Closing the main console window, force-killing `zeropi.exe`, or any sudden crash immediately guarantees that the background model server and the agent CLI processes are terminated cleanly by the Windows kernel.

3. **6000-Token Default Context Window**
   - Automatically sets the default context size (`ctx_size`) of newly configured or downloaded models to **6000 tokens** inside their generated `config.json`.
   - Propagates this setting seamlessly, instructing the `llama-server` to run with `-c 6000` and configures the Pi agent's internal model settings (`contextWindow: 6000` / `maxTokens: 3000`).

4. **Multi-Backend Support (Vulkan & CPU)**
   - Fully supports Vulkan hardware acceleration (default) and standard CPU fallbacks for `llama.cpp`.

5. **Configurable Console Hiding**
   - Hides the secondary `llama-server.exe` terminal automatically for a clean workspace, while providing an optional setting to display logs in a second window.

---

## Directory Structure

When running ZeroPi, it establishes and maintains the following clean structure within its directory:

```text
zeropi\
├── README.md               # Detailed project documentation
├── Cargo.toml              # Rust project description
├── zeropi_config.json      # Suite configuration
├── zeropi.exe              # Orchestrator binary
├── llama\
│   ├── cpu\
│   │   └── llama-server.exe
│   └── vulkan\
│       └── (vulkan binaries)
├── gguf\
│   └── <model-name>\
│       ├── <model-name>.gguf
│       └── config.json     # Model config (defaults to 6000 ctx_size)
└── workspace\              # Home of the Pi Agent
    ├── node\               # Isolated Node.js runtime
    ├── node_modules\       # Installed agent modules
    ├── home\               # Agent workspace directories
    └── run_pi.bat          # Execution wrapper
```

---

## Configuration (`zeropi_config.json`)

The global configuration file `zeropi_config.json` allows configuring the backend, target model, and runner options:

```json
{
  "default_model": "nvidia-nemotron-3-nano-4b",
  "llama_port": 8080,
  "llama_host": "127.0.0.1",
  "backend": "vulkan",
  "hide_second_terminal": true
}
```

### Config Options
* `default_model`: Folder name under `gguf/` containing the model GGUF and config.
* `llama_port`: The network port the `llama-server` listens on (default: `8080`).
* `llama_host`: The address host the backend binds to (default: `127.0.0.1`).
* `backend`: Inference backend to download and use (`vulkan` or `cpu`).
* `hide_second_terminal`: Set to `true` (default) to run the `llama-server` invisibly in the background. Set to `false` to open a second command prompt displaying real-time generation and token inference logs.

---

## Model Config (`gguf/<model-name>/config.json`)

If a model folder is scanned and does not contain a configuration, ZeroPi automatically generates one with safe defaults:

```json
{
  "name": "nvidia-nemotron-3-nano-4b",
  "filename": "NVIDIA-Nemotron3-Nano-4B-Q4_K_M.gguf",
  "download_url": "https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Nano-4B-GGUF/resolve/main/NVIDIA-Nemotron3-Nano-4B-Q4_K_M.gguf",
  "ctx_size": 6000,
  "n_gpu_layers": 99,
  "temperature": 0.0,
  "thinking": false
}
```

* `thinking`: A boolean (defaults to `false`). Set to `true` for models that support native reasoning/thinking output patterns (e.g. Qwen-3-Thinking or deepseek-r1 reasoning outputs). When set to `true`, ZeroPi configures the Pi agent's model profile with `"reasoning": true` so it properly parses and utilizes the model's thinking sequences.

---

## Automatic Context Compaction Resolution

To prevent context window overflows, the Pi coding agent automatically compacts and summarizes older history when the active session reaches the model's context threshold. 
* By default, the Pi agent keeps the most recent **20,000 tokens** unsummarized.
* Since ZeroPi configures a smaller, resource-efficient context size of **6,000 tokens**, standard runs trigger compaction loops throwing: `Compaction failed: Nothing to compact (session too small)`.
* ZeroPi fixes this by dynamically injecting `"compaction": { "enabled": true, "keepRecentTokens": ctx_size / 2 }` (e.g., keeping `3000` tokens) into the Pi agent's `settings.json`. This enables clean, error-free context summarization optimized for local resource-constrained models.

---

## Building and Running

### Prerequisites
* Rust compiler (2024 edition or newer)
* Active Internet Connection (only during the first launch or auto-setup)

### 1. Build the Release Binary
Run standard cargo build to generate the optimized executable:
```powershell
cargo build --release
```

### 2. Copy the Executable
Copy the compiled binary from the target directory to your root directory:
```powershell
Copy-Item -Path "target/release/zeropi.exe" -Destination "zeropi.exe" -Force
```

### 3. Run
Launch the application:
```powershell
./zeropi.exe
```
*On the first run, ZeroPi will detect missing runtimes and automatically trigger the auto-setup routine to download Node.js, the specified llama backend, and configure the workspace environment.*

---

## Credits

ZeroPi orchestrates and depends upon the following excellent open-source projects:

* **[Pi Coding Agent](https://github.com/earendil-works/pi-coding-agent)**: The interactive terminal-based AI coding assistant.
* **[llama.cpp](https://github.com/ggerganov/llama.cpp)**: The highly optimized C/C++ LLM inference engine powering the local backend server.

---

## License

GNU General Public License v3.0
