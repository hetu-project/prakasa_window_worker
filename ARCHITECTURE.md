# Prakasa Windows CLI Architecture Analysis

## Core Architecture: C++ Shell + Python Core

**Yes, your understanding is completely correct!** Prakasa Windows CLI indeed uses **C++ as a shell**, ultimately calling the **Python version of Prakasa**.

## Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Windows User Layer                        │
│                  prakasa.exe (C++ CLI)                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ├── check    (Environment Check)
                     ├── install  (Environment Setup)
                     ├── config   (Configuration)
                     │
                     └── Core Commands (WSL Forwarding)
                         ├── run   ──┐
                         ├── join  ──┤
                         ├── chat  ──┤
                         └── cmd   ──┤
                              │      │
┌─────────────────────────────┼──────┼──────────────────────┐
│              WSL2 (Ubuntu)  │      │                       │
│                             ▼      ▼                       │
│  ~/prakasa/                                              │
│  ├── venv/                 (Python Virtual Environment)   │
│  │   └── bin/activate                                     │
│  ├── src/prakasa/                                        │
│  │   └── launch.py         (Python Core)                 │
│  └── setup.py              (pip install -e .[gpu])       │
│                                                            │
│  Actual Execution:                                        │
│  $ cd ~/prakasa                                          │
│  $ source ./venv/bin/activate                            │
│  $ prakasa run [args]     ← Python CLI                  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## Detailed Workflow

### 1. Environment Setup Phase (install command)

The C++ code executes in WSL2:

```bash
# 1. Clone Python version of Prakasa
cd ~ && git clone https://github.com/hetu-project/prakasa.git

# 2. Create Python virtual environment
cd ~/prakasa
python3 -m venv ./venv

# 3. Install Python version of Prakasa (editable mode)
source ./venv/bin/activate
pip install -e '.[gpu]'
```

**Key Code Location:**

- `src/prakasa/environment/software_installer2.cpp` (lines 200-280)

### 2. Execution Phase (run/join/chat commands)

When user executes `prakasa.exe run -m Qwen/Qwen3-0.6B`:

```cpp
// 1. C++ builds WSL command
std::string command =
    "cd ~/prakasa && "
    "export PATH=/usr/local/cuda-12.8/bin:$PATH && "
    "source ./venv/bin/activate && "
    "prakasa run -m Qwen/Qwen3-0.6B";  // ← Calls Python CLI

// 2. Execute via WSL
wsl.exe -d Ubuntu-24.04 -u root -- bash -c "$command"
```

**Key Code Locations:**

- `src/prakasa/cli/commands/model_commands.cpp` (lines 40-70)
- `src/prakasa/cli/commands/base_command.h` (lines 214-220)

### 3. Real-time Output Forwarding

C++ uses `WSLProcess` class to read Python program output in real-time:

```cpp
WSLProcess wsl_process;
int exit_code = wsl_process.Execute(wsl_command);
// Real-time printing of Python's stdout/stderr
```

**Key Code Locations:**

- `src/prakasa/utils/wsl_process.cpp`
- `src/prakasa/utils/process.cpp`

## C++ Shell Responsibilities

### 1. **Windows Environment Management** (Pure C++)

- ✅ Check Windows version and permissions
- ✅ Enable WSL2 and Virtual Machine Platform
- ✅ Install WSL2 kernel updates
- ✅ Detect NVIDIA GPU and drivers
- ✅ Verify CUDA Toolkit version

**Code Locations:**

- `src/prakasa/environment/system_checker.cpp`
- `src/prakasa/environment/windows_feature_manager.cpp`

### 2. **WSL2 Environment Configuration** (via WSL commands)

- ✅ Install Ubuntu distribution
- ✅ Set proxy environment variables
- ✅ Clone Python version of Prakasa repository
- ✅ Create Python virtual environment
- ✅ Install Python dependencies

### 3. **Command Forwarding & Proxy** (Core Functionality)

- ✅ Parse user command-line arguments
- ✅ Build WSL execution commands
- ✅ Activate Python virtual environment
- ✅ Call Python version of prakasa CLI
- ✅ Forward output and errors in real-time
- ✅ Support Ctrl+C interruption

### 4. **Configuration Management** (Pure C++)

- ✅ Manage `prakasa_config.txt`
- ✅ Proxy URL configuration
- ✅ WSL distribution selection
- ✅ Git repository URL configuration

## Python Core Responsibilities

The Python version of Prakasa handles all actual inference logic:

- 🐍 Distributed inference scheduling
- 🐍 Model loading and sharding
- 🐍 GPU management and CUDA calls
- 🐍 Network communication (gRPC/HTTP)
- 🐍 Web UI server
- 🐍 Tensor parallelism and pipeline parallelism

## Command Mapping

| Windows Command       | C++ Behavior                       | Command Executed in WSL                                           |
| --------------------- | ---------------------------------- | ----------------------------------------------------------------- |
| `prakasa check`       | Pure C++ environment check         | _(Does not call Python)_                                          |
| `prakasa install`     | C++ install + clone Python version | `git clone ...` + `pip install`                                   |
| `prakasa config`      | Pure C++ configuration management  | _(Does not call Python)_                                          |
| `prakasa run [args]`  | **Forward to Python**              | `cd ~/prakasa && source venv/bin/activate && prakasa run [args]`  |
| `prakasa join [args]` | **Forward to Python**              | `cd ~/prakasa && source venv/bin/activate && prakasa join [args]` |
| `prakasa chat [args]` | **Forward to Python**              | `cd ~/prakasa && source venv/bin/activate && prakasa chat [args]` |
| `prakasa cmd <cmd>`   | Forward any command to WSL         | `wsl bash -c "<cmd>"`                                             |

## Why This Architecture?

### ✅ Advantages

1. **User Experience Optimization**

   - Windows users only need to double-click `.exe` to install
   - No manual WSL2 or Python environment configuration needed
   - One-click environment check and installation

2. **Windows System Integration**

   - Requires Windows API to enable system features
   - Requires administrator privileges for registry operations
   - Requires PowerShell to manage WSL2

3. **Environment Isolation**

   - C++ handles Windows-side logic
   - Python handles Linux-side inference logic
   - Clear separation of concerns

4. **Performance Considerations**
   - C++ shell starts quickly
   - System-level operations more efficient in C++
   - GPU computation handled by Python/CUDA

### ⚠️ Trade-offs

1. **Dual-Language Maintenance**

   - Need to maintain both C++ and Python code
   - Interface changes require synchronization

2. **Complex Error Handling**

   - Need to pass error information across processes
   - Encoding conversion issues (UTF-8 ↔ UTF-16)

3. **Debugging Difficulty**
   - Need to debug both Windows + WSL environments
   - Cross-process communication troubleshooting

## Key Technical Implementations

### 1. WSL Command Execution

```cpp
// Build command
std::string command =
    "cd ~/prakasa && "
    "source ./venv/bin/activate && "
    "prakasa run";

// Execute via wsl.exe
std::string wsl_cmd =
    "wsl.exe -d Ubuntu-24.04 -u root -- bash -c \"" + command + "\"";

// Real-time output
WSLProcess::Execute(wsl_cmd);
```

### 2. Encoding Conversion

```cpp
// PowerShell output is UTF-16 LE
std::string utf8_output =
    ConvertPowerShellOutputToUtf8(powershell_raw_output);

// WSL output is UTF-8
std::string converted =
    ConvertWslOutputToUtf8(wsl_raw_output);
```

### 3. Proxy Configuration Synchronization

```cpp
// C++ reads configuration
std::string proxy_url = config.GetConfigValue("proxy_url");

// Pass to Python
std::string command =
    "HTTP_PROXY='" + proxy_url + "' "
    "HTTPS_PROXY='" + proxy_url + "' "
    "prakasa run";
```

## Summary

Prakasa Windows CLI is a typical **"Local Shell + Remote Core"** architecture:

- **C++ Shell**: Handles Windows-specific environment configuration and system integration
- **Python Core**: Handles actual distributed inference logic
- **WSL2 Bridge**: Connects Windows and Linux environments

This design allows Windows users to enjoy the same Prakasa functionality as Linux users, while providing a native Windows installation and configuration experience.

## Related File Index

**C++ Shell Core Files:**

- `src/prakasa/main.cpp` - Entry point
- `src/prakasa/cli/command_parser.cpp` - Command routing
- `src/prakasa/cli/commands/model_commands.cpp` - run/join/chat forwarding
- `src/prakasa/environment/software_installer2.cpp` - Python environment installation

**Python Core (in WSL2):**

- `~/prakasa/` - Python version of Prakasa repository
- `~/prakasa/venv/` - Python virtual environment
- `~/prakasa/src/prakasa/launch.py` - Python CLI entry point

**Configuration Files:**

- `prakasa_config.txt` - Configuration file read by C++
- `~/.bashrc` - WSL environment variables (CUDA paths, etc.)
