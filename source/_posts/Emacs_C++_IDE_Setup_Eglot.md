---
title: "Turning Emacs into a Modern C++ IDE on macOS: A Comprehensive Configuration Guide (Eglot Edition)"
categories: "Dev Environment"
comments: true
---
# Introduction

## Purpose and Vision

This guide aims to provide a detailed and precise roadmap for transforming Emacs, a powerful text editor, into a fully-featured C++ Integrated Development Environment (IDE) on macOS that rivals or surpasses mainstream commercial products. The focus will be on CMake-based projects, using MacPorts as the sole system-level package manager. The ultimate goal is to create a highly integrated, responsive, and fully customizable development environment tailored to the developer's needs.

## The Emacs Philosophy: Why Choose Emacs?

The choice of Emacs as the foundation for a C++ IDE is not driven by nostalgia or tradition but by its unparalleled core strengths. Emacs is designed as “an extensible, self-documenting, real-time display editor.” This means nearly every aspect of it can be customized and extended using Emacs Lisp (Elisp). This extreme flexibility, combined with seamless integration with command-line tools and a vast ecosystem of thousands of high-quality packages built by a global developer community, makes it an ideal platform for crafting the ultimate development environment. Unlike rigid, feature-locked IDEs, Emacs allows developers to sculpt their environment precisely to their preferences.

## Architecture Overview

The C++ IDE we will build is a sophisticated system composed of multiple interoperating components. Understanding these components and their roles is key to successful configuration. The architecture can be divided into three layers:

1. **macOS System Toolchain**: This is the foundation of our IDE, independent of Emacs. It includes the compiler (clang), language server (clangd), and debugger (lldb) provided by the LLVM project. These tools will be installed and managed via MacPorts to ensure version consistency and control.
2. **Emacs Backend Integration Clients**: This layer serves as the bridge between Emacs and the external toolchain. We will use `eglot` as the Language Server Protocol (LSP) client, known for its lightweight nature and tight integration with Emacs’s native tools. It communicates with `clangd` in real-time to provide code analysis, completion, and diagnostics. Additionally, `dap-mode` will serve as the Debug Adapter Protocol (DAP) client, interacting with `lldb`’s debug adapter to enable graphical debugging within Emacs.
3. **Emacs Frontend User Experience (UX) Enhancements**: This layer presents backend data to the user in an efficient, visually appealing, and intuitive manner. We will configure `company-mode` for smooth autocompletion popups and leverage Emacs’s built-in `flymake` for real-time error highlighting.

This guide will build this system layer by layer, with clear explanations and explicit instructions for each step.

# Part 1: Preparing the macOS Toolchain with MacPorts

Before configuring Emacs, we must establish a robust and reliable C++ development toolchain on macOS. All subsequent Emacs configurations depend on the correct installation and setup of these external tools. Per the requirements, we will use MacPorts exclusively for this task.

## 1.1. LLVM Toolchain: Compiler and Language Server

The core of a modern C++ IDE lies in its “intellisense” capabilities, driven by a language server. For C++, `clangd` is one of the most advanced and widely used language servers. It is part of the LLVM project and provides highly accurate code analysis.

**Principle**: `clangd` parses your code in the background to enable real-time error checking, code completion, definition jumping, and reference finding. To ensure compatibility between `clangd` and your compiler, the best practice is to install a complete LLVM toolchain that includes `clang`, `clang++`, and `clangd`.

**Instructions**: We will install a recent, stable version of Clang. At the time of writing, `clang-18` is a good choice. Execute the following command in a macOS terminal:

```bash
sudo port install clang-18
```

**Explanation**: This command downloads, compiles, and installs `clang-18` and its related tools via MacPorts. Upon completion, you will have the `clang` compiler, `clang++` C++ compiler, and, most importantly, the `clangd` language server.

## 1.2. LLDB Debugger

Code debugging is an essential IDE feature. On macOS, LLDB is the standard system-level debugger. While Xcode’s Command Line Tools provide a version of LLDB, to maintain consistency across the toolchain (i.e., ensuring the compiler, language server, and debugger are from the same or compatible LLVM versions), we recommend installing LLDB via MacPorts.

**Instructions**: Execute the following command to install a version of LLDB compatible with the Clang version installed earlier. For example, `lldb-11` is a stable and widely used version, but you can choose the latest stable version available in MacPorts.

```bash
sudo port install lldb-11
```

**Explanation**: This command installs the `lldb` debugger. Later, our Emacs debugging client, `dap-mode`, will control this `lldb` process through an intermediary layer (debug adapter) to enable breakpoint setting, stepping, and variable inspection.

## 1.3. Critical Step: Activating the MacPorts Toolchain

This is a crucial but often overlooked step. Simply installing packages via MacPorts does not automatically make them the system’s default commands. macOS’s default PATH environment variable typically prioritizes Xcode-provided tools (located in `/usr/bin`). If Emacs or the terminal cannot locate the newly installed tools in `/opt/local/bin`, all subsequent LSP and debugging configurations will fail or behave incorrectly.

**Problem Analysis**:
1. After running `sudo port install clang-18`, the relevant executables (e.g., `clang-mp-18`, `clangd-mp-18`) are installed in `/opt/local/bin`.
2. Running `which clangd` in a new terminal may still return `/usr/bin/clangd`, indicating the older Xcode-provided version.
3. Emacs inherits this incorrect PATH, causing `eglot` to launch the wrong `clangd`, which may fail to recognize modern C++ syntax or project configurations, leading to confusing errors.
4. The solution is to use MacPorts’ `port select` mechanism, which creates versionless symbolic links in `/opt/local/bin` (e.g., `clang`, `clangd`) to ensure they are prioritized in the PATH.

**Instructions**:
We will use `port select` to set the system’s default `clang` to the newly installed `mp-clang-18` version.

1. List available Clang versions:
```bash
port select --list clang
```
You should see entries like `mp-clang-18` in the output.

2. Set the default Clang version:
```bash
sudo port select --set clang mp-clang-18
```
This command creates symbolic links, such as `/opt/local/bin/clang` pointing to `clang-mp-18` and `/opt/local/bin/clangd` pointing to `clangd-mp-18`.

3. Repeat for LLDB (if supported by `port select`):
Check if LLDB has a selection group:
```bash
port select --list lldb
```
If available, set the default LLDB version similarly.

## 1.4. Verification

Before proceeding with Emacs configuration, we must verify that the toolchain foundation is solid.

**Instructions**: Open a new terminal window (critical, as old windows may not reflect the updated PATH) and run:

```bash
which clangd
clangd --version
which lldb
lldb --version
```

**Expected Output**: `which clangd` should return `/opt/local/bin/clangd`. `clangd --version` should show the installed version (e.g., 18.x.x). `lldb` commands should also point to `/opt/local/bin/`. If the output matches expectations, the system-level toolchain is ready.

# Part 2: Building a Modular Emacs Configuration Architecture

A clear, maintainable Emacs configuration is key to long-term efficiency. We will adopt modern Emacs community best practices to create a powerful and manageable configuration structure.

## 2.1. The Power of `use-package`

While Emacs configuration can be achieved with simple `require` and `setq` statements, the modern approach strongly recommends the `use-package` macro. It offers the following core advantages:
- **Organization**: Groups all configurations related to a single package (variable settings, keybindings, hooks) into a single block, improving readability and maintainability.
- **Performance Optimization**: Supports lazy-loading, meaning Emacs only loads a package’s code when needed (e.g., when a command is invoked or a specific mode is entered), significantly speeding up startup.
- **Automation**: Automatically downloads and installs missing packages from repositories like MELPA, simplifying deployment.

A simple `use-package` example:

```emacs-lisp
(use-package magit
  :ensure t  ; Ensure package is installed
  :bind ("C-x g" . magit-status) ; Bind a key
  :config
  ;; Configuration executed after magit is loaded
  (setq magit-display-buffer-function #'magit-display-buffer-same-window-except-diff-v1))
```

## 2.2. Configuration Structure: `init.el` and `init-cpp.el`

Per your requirements, we will separate C++-related configurations into a dedicated `init-cpp.el` file, loaded by the main `init.el` configuration. This modular approach enhances clarity, maintainability, and shareability.

**Loading Method**: Emacs supports two primary methods for loading sub-configurations: `load-file` and `require`.
- `load-file`: Directly executes the contents of a specified file. It is simple and ideal for personal configuration files, as you control the loading timing and content.
- `require`: Checks if a “feature” has been provided (via the `provide` function in the loaded file). It prevents duplicate loading, useful for complex, multi-file Elisp libraries.

For our scenario—loading a single C++ configuration module at startup—`load-file` is the best choice due to its simplicity and transparency.

**Instructions**:
1. Create the main configuration file `~/.emacs.d/init.el`:
If it doesn’t exist, create it. This is the default file loaded by Emacs at startup. It will contain core settings, including package manager initialization and loading of the C++ module.

```emacs-lisp
;; ~/.emacs.d/init.el

;; --- Package Manager Setup ---
;; Add MELPA package repository, the main source for up-to-date Emacs packages
(require 'package)
(add-to-list 'package-archives '("melpa" . "https://melpa.org/packages/"))
(package-initialize)

;; --- Ensure use-package is Available ---
;; Automatically install use-package if not present
(unless (package-installed-p 'use-package)
  (package-refresh-contents)
  (package-install 'use-package))
(require 'use-package)
(setq use-package-always-ensure t) ; Make use-package auto-install packages

;; --- Load C++ Configuration Module ---
;; Use load-file to load our C++-specific configuration
;; expand-file-name ensures path correctness
(load-file (expand-file-name "init-cpp.el" user-emacs-directory))

;; --- Other general configurations can go here ---
```

2. Create the C++ configuration file `~/.emacs.d/init-cpp.el`:
Create this empty file. All C++-related `use-package` blocks provided in later sections will be written to this file.

## 2.3. Core Emacs Packages and Their IDE Roles

Before diving into specific configuration code, the following table provides an overview of the IDE ecosystem we will build, clarifying each core package’s role to help you understand how they work together.

| Package                | Role in IDE                                           | Category         |
|------------------------|-------------------------------------------------------|------------------|
| eglot                  | Core LSP client. Manages communication with `clangd` for code intelligence. | Backend Integration |
| company                | General autocompletion framework (UI) for displaying `eglot` completion suggestions. | User Interface |
| flymake                | Emacs’s built-in real-time syntax checking framework, driven by `eglot` for diagnostics. | User Interface |
| projectile             | Project management tool for project-level operations and recognition. | Project Management |
| treemacs               | Tree-based file and project browser, providing a sidebar view. | User Interface |
| treemacs-projectile    | Integration package for `treemacs` and `projectile`.   | UI Enhancement   |
| dap-mode               | Core DAP client. Manages communication with the debug adapter. | Backend Integration |
| dap-gdb-lldb           | Bridges `dap-mode` with the `lldb` debug adapter.      | Debugging        |
| modern-cpp-font-lock   | Enhances syntax highlighting for modern C++ (C++11 and later) to improve readability. | Editing |

# Part 3: Enabling Intelligent Code Editing with Eglot

This section is the heart of our IDE, configuring Emacs to leverage the powerful features of the `clangd` language server for code completion, definition jumping, and real-time diagnostics.

## 3.1. LSP Architecture: Emacs and `clangd` Collaboration

The Language Server Protocol (LSP) is an open standard defining communication between an editor (client) and a language intelligence tool (server). This client-server model is the backbone of modern IDEs, offering:
- **Decoupling**: Heavy tasks like code parsing are handled by the high-performance `clangd` server in a background process. Emacs acts as a lightweight client, sending requests (e.g., “Where is this symbol defined?”) and receiving responses, keeping Emacs responsive.
- **Reusability**: Any LSP-compliant editor can use the same `clangd` server, fostering a robust ecosystem.

In our setup, `eglot` is the LSP client in Emacs, responsible for launching `clangd` and exchanging JSON-RPC messages via standard input/output. `eglot` is favored for its lightweight design and seamless integration with Emacs’s native tools like `xref` and `flymake`.

## 3.2. The Critical Hub: `compile_commands.json`

The effectiveness of the LSP system hinges on the language server’s ability to understand your project. For a complex language like C++, a source file is not self-contained—its correct parsing depends on external factors like header search paths (`-I`), preprocessor macros (`-D`), and other compiler flags.

The `compile_commands.json` file is a standardized solution to this problem. It is a JSON compilation database listing the exact compilation commands for each source file in the project. When `eglot` starts `clangd`, `clangd` automatically looks for this file in the project root. If found, it uses the correct context to parse your code, just like a compiler. Without this file, `clangd` resorts to guesswork, often leading to false errors (e.g., complaining that the standard library header `<vector>` cannot be found), rendering LSP functionality ineffective. Thus, ensuring your build system generates this file is an absolute prerequisite before configuring the editor.

## 3.3. Generating `compile_commands.json` with CMake

Fortunately, since your project uses CMake, generating this file is straightforward. Modern CMake versions have built-in support for this feature.

**Instructions**: Add the following line to your project’s main `CMakeLists.txt` file:

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

**Explanation**:
- Adding this line to `CMakeLists.txt` enables native LSP support for your project. Any developer using Emacs, VS Code, or another LSP-supporting editor will automatically generate `compile_commands.json` in the build directory when building the project.
- Alternatively, you can pass a command-line argument when invoking `cmake`: `cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON`. This is equally effective but requires remembering to include the flag each time. Adding it to `CMakeLists.txt` is more robust and permanent.
- This feature is well-supported for Makefile and Ninja generators, covering typical macOS use cases.

## 3.4. Configuring the `eglot` Client

Now, we can configure `eglot` in the `init-cpp.el` file, using the verified direct hook method for reliability.

**Instructions**: Add the following code block to your `~/.emacs.d/init-cpp.el`:

```emacs-lisp
;; Ensure eglot package is installed
(use-package eglot :ensure t)

;; Use the verified configuration to ensure eglot auto-starts
(require 'eglot)
(add-hook 'c-mode-hook 'eglot-ensure)
(add-hook 'c++ Hawkins)
(add-hook 'c-or-c++-mode-hook 'eglot-ensure)

;; Ensure stability by explicitly specifying clangd for C/C++ modes
(with-eval-after-load 'eglot
  (add-to-list 'eglot-server-programs '((c++-mode c-mode) "clangd")))
```

**Configuration Details**:
- `(use-package eglot :ensure t)`: Ensures the `eglot` package is downloaded and installed.
- `(require 'eglot)` and `(add-hook...)`: The core configuration, verified as effective. It tells Emacs to run `eglot-ensure` automatically when opening a C or C++ file, starting `clangd` and establishing an LSP session.
- `(with-eval-after-load 'eglot...)`: Enhances stability by ensuring that after `eglot` is loaded, it updates the server list to specify `clangd` for `c++-mode` and `c-mode`.

## 3.5. Seamless Code Completion with `company-mode`

`eglot` retrieves completion data from `clangd`, but it needs a frontend (UI) framework to display suggestions as a popup. `company-mode` (COMPlete ANYthing) is the most popular and mature completion framework in Emacs, integrating seamlessly with `eglot` via Emacs’s standard `completion-at-point` mechanism.

**Instructions**: Add the following `use-package` configuration to `init-cpp.el`:

```emacs-lisp
(use-package company
  :ensure t
  :hook (prog-mode . company-mode)
  :bind (:map company-active-map
         ("<tab>" . company-complete-selection))
  :custom
  (company-minimum-prefix-length 1)
  (company-idle-delay 0.1))
```

**Configuration Details**:
- `:hook (prog-mode . company-mode)`: Automatically enables `company-mode` in all programming modes (as `prog-mode` is their parent mode).
- `company-backends`: `eglot` provides completions via the standard `completion-at-point-functions` hook, and `company-mode`’s default backend, `company-capf`, uses this hook automatically, so no manual backend configuration is needed.
- `:bind`: Binds the Tab key to select a completion when options are available, mimicking modern IDE behavior.
- `:custom`: Fine-tunes `company-mode` behavior. `company-minimum-prefix-length 1` starts completion after one character; `company-idle-delay 0.1` displays the completion popup 0.1 seconds after typing stops.

## 3.6. Effortless Code Navigation

`eglot` seamlessly integrates with Emacs’s built-in `xref` framework, enabling immediate use of standard navigation keybindings without additional configuration.

**Core Navigation Commands**:
- `M-.` (Alt + .): Runs `xref-find-definitions` to jump to the definition of the symbol under the cursor (e.g., function, variable, or class), fulfilling your “function definition jump” requirement.
- `M-?` (Alt + ?): Runs `xref-find-references` to list all references to the symbol in the project in a new window.
- `M-,` (Alt + ,): Runs `xref-pop-marker-stack` to return to the position before the jump, enabling seamless code navigation.

# Part 4: Crafting an Exceptional User Experience

A powerful IDE needs not only intelligent functionality but also an attractive, intuitive interface. This section adds optional but highly recommended packages to provide the visual feedback and interaction enhancements expected of a modern IDE.

## 4.1. Enhanced Syntax Highlighting for Modern C++

Emacs’s built-in `c++-mode` provides decent syntax highlighting. However, with the evolution of C++ standards (C++11, 14, 17, 20, 23), new keywords (e.g., `constexpr`, `noexcept`, `concept`), attributes (e.g., `[[deprecated]]`, `[[nodiscard]]`), and syntactic constructs have been introduced. A dedicated package can provide more precise and richer color highlighting for these features, improving code readability.

**Instructions**: We recommend the `modern-cpp-font-lock` package. Add the following to `init-cpp.el`:

```emacs-lisp
(use-package modern-cpp-font-lock
  :ensure t
  :hook (c++-mode . modern-c++-font-lock-mode))
```

**Explanation**:
- This configuration auto-installs `modern-cpp-font-lock`.
- The `:hook` ensures that `modern-c++-font-lock-mode`, a minor mode, activates automatically for C++ files.
- It enhances `c++-mode`’s base highlighting by coloring modern C++-specific syntax, making code more readable.

## 4.2. Real-Time Diagnostics with `flymake`

`eglot` integrates with Emacs’s built-in `flymake` framework for real-time code diagnostics (errors and warnings). When `eglot` receives diagnostics from `clangd`, it passes them to `flymake`, which highlights problematic code lines in the buffer.

**Configuration and Usage**:
- **No configuration needed**: `eglot` handles `flymake` integration automatically; no additional packages or configuration are required.
- **View errors**: Hover the mouse over code with red underlines or move the cursor to the line to see detailed error messages in the minibuffer.
- **Browse errors**: Run `M-x flymake-show-buffer-diagnostics` to open a buffer listing all diagnostics for the current file, allowing you to jump to and fix issues.

## 4.3. Customizing C++ Indentation Style: Handling Namespaces

By default, Emacs’s `c++-mode` indents code within namespace blocks. While this is desired in some coding standards, others prefer no extra indentation for namespace contents. We can easily adjust this behavior.

**Instructions**: Set the `innamespace` value in `c-offsets-alist` to control namespace indentation. A value of 0 disables indentation.

```emacs-lisp
(use-package cc-mode
  :ensure nil ;; cc-mode is built-in
  :config
  (defun my-cpp-indent-hook ()
    "Custom settings for C++ indentation."
    ;; Do not indent code inside namespaces.
    (c-set-offset 'innamespace 0))
  (add-hook 'c++-mode-hook 'my-cpp-indent-hook))
```

**Explanation**:
- Uses `use-package` to organize configuration for the built-in `cc-mode`.
- Via `c++-mode-hook`, ensures `my-cpp-indent-hook` runs for every C++ file.
- `c-set-offset 'innamespace 0` sets the namespace indentation offset to 0, achieving the desired effect.

## 4.4. Project File Browsing with `treemacs`

To provide a modern IDE-like project file tree view, we integrate `treemacs`, a powerful, modern-looking file browser that integrates seamlessly with `projectile`.

**Instructions**: First, configure `projectile` for project recognition, then set up `treemacs` and its `projectile` integration plugin, `treemacs-projectile`.

```emacs-lisp
(use-package projectile
  :ensure t
  :config
  (projectile-mode +1))

(use-package treemacs
  :ensure t
  :defer t
  :config
  (treemacs-follow-mode t)
  (treemacs-git-mode 'extended)
  :bind
  (:map global-map
    ("M-0"     . treemacs-select-window)
    ("C-x t 1" . treemacs-delete-other-windows)
    ("C-x t t" . treemacs)))

(use-package treemacs-projectile
  :after (treemacs projectile)
  :ensure t)
```

**Explanation**:
- `projectile-mode +1` enables `projectile` globally.
- `treemacs` is deferred (`:defer t`) to improve startup speed.
- `treemacs-follow-mode` auto-positions the `treemacs` window to the file being edited.
- `treemacs-git-mode 'extended'` displays different icons based on file Git status.
- Global keybindings like `C-x t t` quickly open/close the `treemacs` window.
- `treemacs-projectile` integrates `treemacs` with `projectile`. After installation, you can access `treemacs-projectile` via `projectile`’s command menu (`C-c p`) or run `M-x treemacs-projectile` to open the current project.

# Part 5: Integrating the Debugger with `dap-mode` and LLDB

This is the final cornerstone of our IDE, integrating the powerful LLDB debugger directly into Emacs for a fully interactive debugging experience. This configuration is independent of the LSP client (`eglot` or `lsp-mode`) and works seamlessly with it.

## 5.1. DAP Architecture: Parallels with LSP

Understanding DAP (Debug Adapter Protocol) integration involves recognizing its similarity to LSP. DAP is to debugging what LSP is to code analysis.

**Architecture Comparison**:
- **LSP Model**: Emacs (editor) <-> `eglot` (client) <-> `clangd` (language server)
- **DAP Model**: Emacs (editor) <-> `dap-mode` (client) <-> Debug Adapter <-> `lldb` (debugger)

**Core Concepts**:
1. Like LSP, DAP is an open standard (championed by Microsoft) defining a universal protocol for communication between development tools and debuggers.
2. `dap-mode` is the DAP client in Emacs, sending debugging commands like “step next” or “set breakpoint.”
3. The **Debug Adapter** is a critical intermediary process, translating DAP’s generic protocol commands into specific instructions that `lldb` understands.
4. This adapter is typically provided by VS Code debugging extensions, and `dap-mode` leverages these high-quality adapters.

Understanding this parallel architecture demystifies debugging configuration and explains the need for additional components.

## 5.2. Installing the `lldb-dap` Debug Adapter

`dap-mode` requires a debug adapter to communicate with `lldb`. The `dap-gdb-lldb` Emacs package provides a convenient command to download and install a compatible adapter from the VS Code marketplace.

**Instructions**: In Emacs, run the following interactive command:

```
M-x dap-gdb-lldb-setup
```

**Explanation**:
- This command downloads a VS Code extension (e.g., `webfreak.code-debug`) containing GDB and LLDB debug adapters, extracting it to a local folder in Emacs’s configuration directory.
- `dap-mode` then locates and uses this adapter.
- Note: The adapter’s name may evolve (e.g., from `lldb-vscode` to `lldb-dap`), but `dap-gdb-lldb-setup` typically handles these details. If issues arise, manually set the `dap-lldb-debug-program` variable to point to the correct adapter executable path.

## 5.3. Configuring `dap-mode`

Add the `dap-mode` configuration to `init-cpp.el`.

**Instructions**:

```emacs-lisp
(use-package dap-mode
  :ensure t
  :config
  ;; Required to enable LLDB support
  (require 'dap-gdb-lldb)
  ;; Enable dap-mode UI components (locals, call stack, etc.)
  (dap-ui-mode 1)
  ;; Show variable values on hover
  (dap-tooltip-mode 1)
  ;; (Optional) Auto-show debug control panel (hydra) when program pauses
  (add-hook 'dap-stopped-hook
            (lambda (arg) (call-interactively #'dap-hydra)))
```

**Configuration Details**:
- `(require 'dap-gdb-lldb)`: Essential for loading LLDB-specific configuration and templates.
- `(dap-ui-mode 1)` and `(dap-tooltip-mode 1)`: Enable `dap-mode`’s graphical UI, automatically splitting the window to show call stack, locals, and watch expressions when debugging starts.
- `dap-stopped-hook`: Adds a hook to pop up the `dap-hydra` control panel when the program pauses, allowing single-key commands (e.g., `n` for next, `i` for step-in) to control debugging.

## 5.4. Defining Reusable Debug Launch Templates

To start a debug session, `dap-mode` needs the executable to debug, the working directory, and command-line arguments. Manually entering this information each time is tedious. The solution is a reusable “launch template.”

**Key Insight**: `dap-mode`’s launch templates use flexible variable substitution syntax (e.g., `${workspaceFolder}`) borrowed from VS Code’s variable reference specification, not Elisp. Understanding this unlocks powerful, project-agnostic debugging configurations.

**Instructions**: Register a template named “C++ (LLDB) Launch” in `init-cpp.el` using `dap-register-debug-template`.

```emacs-lisp
(dap-register-debug-template
 "C++ (LLDB) Launch"
 (list :type "lldb-vscode"      ; Must match the debug adapter
       :request "launch"         ; "launch" starts a new process
       :name "C++ (LLDB) Launch" ; Name shown in selection list
       ;; Key part: Use variables to dynamically determine executable path
       ;; Assumes executable shares source file’s name (without extension)
       ;; and is in the project root’s "build/debug/" subdirectory
       :program "${workspaceFolder}/build/debug/${fileBasenameNoExtension}"
       :cwd "${workspaceFolder}" ; Set working directory to project root
       :args                     ; Command-line arguments for the program
       :stopOnEntry nil))        ; Whether to pause at program entry
```

**Template Details**:
- `:type "lldb-vscode"`: Specifies the debug adapter type, matching `dap-gdb-lldb`’s registered type. It may be called `lldb-dap`, but `dap-gdb-lldb` handles compatibility.
- `:request "launch"`: Indicates launching a new process. “attach” is an alternative for attaching to running processes.
- `:program`: The executable’s path, using VS Code-style variables:
  - `${workspaceFolder}`: The project root (typically the Git repository root).
  - `${fileBasenameNoExtension}`: The current source file’s base name without extension.
- `:cwd`: Sets the program’s working directory to `${workspaceFolder}`.
- A full list of variables is in VS Code’s documentation, enabling flexible, portable debug configurations.

# Part 6: Complete Solution and Example Workflow

This section integrates all configurations and provides a practical workflow example demonstrating how our carefully crafted IDE works together.

## 6.1. Final `init-cpp.el` Configuration File

Below is the complete, annotated `init-cpp.el` file, combining all `use-package` blocks from previous sections—a ready-to-use final product.

```emacs-lisp
;;; init-cpp.el --- Emacs C++ IDE Configuration (Eglot) -*- lexical-binding: t; -*-

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; 1. Intelligent Code Analysis and Navigation (Eglot)
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;; Ensure eglot package is installed
(use-package eglot :ensure t)

;; Use the verified configuration to ensure eglot auto-starts
(require 'eglot)
(add-hook 'c-mode-hook 'eglot-ensure)
(add-hook 'c++-mode-hook 'eglot-ensure)
(add-hook 'c-or-c++-mode-hook 'eglot-ensure)

;; Ensure stability by explicitly specifying clangd for C/C++ modes
(with-eval-after-load 'eglot
  (add-to-list 'eglot-server-programs '((c++-mode c-mode) "clangd")))

(use-package company
  :ensure t
  :hook (prog-mode . company-mode)
  :bind (:map company-active-map
         ("<tab>" . company-complete-selection))
  :custom
  (company-minimum-prefix-length 1)
  (company-idle-delay 0.1))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; 2. User Experience Enhancements (UI & Highlighting)
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(use-package modern-cpp-font-lock
  :ensure t
  :hook (c++-mode . modern-c++-font-lock-mode))

(use-package cc-mode
  :ensure nil ;; cc-mode is built-in
  :config
  (defun my-cpp-indent-hook ()
    "Custom settings for C++ indentation."
    ;; Do not indent code inside namespaces.
    (c-set-offset 'innamespace 0))
  (add-hook 'c++-mode-hook 'my-cpp-indent-hook))

(use-package projectile
  :ensure t
  :config
  (projectile-mode +1))

(use-package treemacs
  :ensure t
  :defer t
  :config
  (treemacs-follow-mode t)
  (treemacs-git-mode 'extended)
  :bind
  (:map global-map
    ("M-0"     . treemacs-select-window)
    ("C-x t 1" . treemacs-delete-other-windows)
    ("C-x t t" . treemacs)))

(use-package treemacs-projectile
  :after (treemacs projectile)
  :ensure t)

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; 3. Integrated Debugging (DAP)
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(use-package dap-mode
  :ensure t
  :config
  ;; Required to enable LLDB support
  (require 'dap-gdb-lldb)
  ;; Enable dap-mode UI components (locals, call stack, etc.)
  (dap-ui-mode 1)
  ;; Show variable values on hover
  (dap-tooltip-mode 1)
  ;; (Optional) Auto-show debug control panel (hydra) when program pauses
  (add-hook 'dap-stopped-hook
            (lambda (arg) (call-interactively #'dap-hydra)))

  ;; Register a reusable C++ debug launch template
  (dap-register-debug-template
   "C++ (LLDB) Launch"
   (list :type "lldb-vscode"
         :request "launch"
         :name "C++ (LLDB) Launch"
         :program "${workspaceFolder}/build/debug/${fileBasenameNoExtension}"
         :cwd "${workspaceFolder}"
         :args
         :stopOnEntry nil)))

(provide 'init-cpp)
;;; init-cpp.el ends here
```

## 6.2. Developer Workflow: From Project to Debugging

This narrative example ties together all configured components, illustrating their synergy in a real development scenario.

1. **Project Setup**:
   - In the terminal, navigate to your CMake project directory.
   - Configure and generate build files, ensuring `compile_commands.json` is generated and debugging symbols (`-g`) are included.
     ```bash
     # Create a build directory
     cmake -S. -B build -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
     ```
   - Build the project:
     ```bash
     cmake --build build
     ```

2. **Open the Project in Emacs**:
   - Start Emacs and open any `.cpp` source file in the project.
   - Observe the minibuffer message: `[eglot] Connected! Server 'clangd' now managing...`, indicating `eglot` has successfully connected to `clangd`.
   - Press `C-x t t` to open a `treemacs` window on the left, displaying the project’s file tree.

3. **Code Navigation**:
   - Move the cursor to a function call and press `M-.` to jump to its definition.
   - After reading, press `M-,` to return to the original position instantly.

4. **Code Completion**:
   - Type `.` or `->` after an object instance; `company-mode`’s completion popup appears, listing available members and functions provided by `clangd`. Use Tab or arrow keys to select and press Enter to complete.

5. **Real-Time Diagnostics**:
   - Introduce a syntax error (e.g., missing semicolon).
   - Upon saving, a red wavy line appears under the offending code. Move the cursor to the line to see detailed error messages in the minibuffer.

6. **Start Debugging**:
   - Navigate to the desired pause point and run `M-x dap-breakpoint-toggle` to set a breakpoint, marked by a red dot in the gutter.
   - Run `M-x dap-debug` to start a debug session.
   - Select the “C++ (LLDB) Launch” template when prompted.
   - Emacs splits the window to display the DAP debugging interface: source code on the top left, call stack and locals on the right, and debug output/REPL below.
   - If `dap-hydra` is configured, a control panel appears. Press `n` for next (`dap-next`), `i` for step-in (`dap-step-in`), or `c` to continue to the next breakpoint (`dap-continue`). Your C++ IDE is now fully operational.

## 6.3. C++ IDE Core Keybindings Reference Table

To help you get started quickly, the following table summarizes the most commonly used and critical keybindings in our configured IDE.

| Operation                | Keybinding | Emacs Command                     |
|--------------------------|------------|-----------------------------------|
| **Code Navigation and Editing** |            |                                   |
| Jump to definition       | M-.        | xref-find-definitions            |
| Find references          | M-?        | xref-find-references             |
| Return from jump         | M-,        | xref-pop-marker-stack            |
| Rename symbol            | M-x eglot-rename | eglot-rename               |
| Format buffer            | M-x eglot-format-buffer | eglot-format-buffer   |
| **Project and File Browsing** |            |                                   |
| Open/close Treemacs      | C-x t t    | treemacs                         |
| Select Treemacs window   | M-0        | treemacs-select-window           |
| Open current project in Treemacs | M-x treemacs-projectile | treemacs-projectile |
| **Debugging Controls**   |            |                                   |
| Start debug session      | M-x dap-debug | dap-debug                     |
| Toggle breakpoint        | M-x dap-breakpoint-toggle | dap-breakpoint-toggle |
| Step over (next)         | M-x dap-next | dap-next                       |
| Step into function       | M-x dap-step-in | dap-step-in                   |
| Step out of function     | M-x dap-step-out | dap-step-out                 |
| Continue execution       | M-x dap-continue | dap-continue                 |
| Terminate debugging      | M-x dap-disconnect | dap-disconnect               |

## 6.4. Troubleshooting and Future Exploration

**Common Issues**:
- **Eglot not starting (no connection message in minibuffer)**:
  1. Check the `*Messages*` buffer for detailed logs.
  2. Ensure `compile_commands.json` exists in the project root (or a symlink points to it). `eglot` searches upward automatically.
  3. Run `which clangd` in a new terminal to confirm it points to `/opt/local/bin/clangd`.
- **Debugging fails to start**:
  1. Verify the `:program` path in the debug template matches your project’s build structure.
  2. Ensure the project was compiled in Debug mode (`CMAKE_BUILD_TYPE=Debug`) with `-g` symbols.
  3. Rerun `M-x dap-gdb-lldb-setup` and check `*Messages*` for errors.

**Future Exploration**:
You’ve built a powerful C++ IDE, but Emacs��s extensibility offers more possibilities. Consider:
- **magit**: A renowned Git client for Emacs, offering a more efficient and intuitive Git experience than graphical or command-line tools.
- **Custom snippets**: Use the `yasnippet` package to create templates for common code patterns (e.g., for loops, class definitions) triggered by simple keywords.

By following this guide, you’ve transformed Emacs from a general-purpose text editor into a fully-featured, highly extensible C++ IDE tailored for macOS development. Welcome to the world of Emacs, where the only limit is your imagination.

# Works Cited
1. Install clang-18 on macOS with MacPorts, accessed July 5, 2025, https://ports.macports.org/port/clang-18/
2. Install clang-11 on macOS with MacPorts, accessed July 5, 2025, https://ports.macports.org/port/clang-11/
3. Install lldb-11 on macOS with MacPorts, accessed July 5, 2025, https://ports.macports.org/port/lldb-11/
4. emacs-lsp/dap-mode: Emacs Debug Adapter Protocol - GitHub, accessed July 5, 2025, https://github.com/emacs-lsp/dap-mode
5. How to use clang-format from macports? - Stack Overflow, accessed July 5, 2025, https://stackoverflow.com/questions/39069542/how-to-use-clang-format-from-macports
6. How to modularize an emacs configuration? - Stack Overflow, accessed July 5, 2025, https://stackoverflow.com/questions/2079095/how-to-modularize-an-emacs-configuration
7. Should I use "require" or "load" when writing my own configuration? [duplicate], accessed July 5, 2025, https://emacs.stackexchange.com/questions/47771/should-i-use-require-or-load-when-writing-my-own-configuration
8. Installation - LSP Mode - LSP support for Emacs, accessed July 5, 2025, https://emacs-lsp.github.io/lsp-mode/page/installation/
9. CMAKE_EXPORT_COMPILE_COMMANDS — CMake 4.1.0-rc1 ..., accessed July 5, 2025, https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html
10. Configuring Emacs as a C/C++ IDE - LSP Mode - LSP support for ..., accessed July 5, 2025, https://emacs-lsp.github.io/lsp-mode/tutorials/CPP-guide/
11. Help with basic setup of LSP with Clangd : r/emacs - Reddit, accessed July 5, 2025, https://www.reddit.com/r/emacs/comments/gc8f90/help_with_basic_setup_of_lsp_with_clangd/
12. Generating a JSON compilation database | JetBrains Fleet, accessed July 5, 2025, https://www.jetbrains.com/help/fleet/generating-a-json-compilation-database.html
13. Generate compile_commands.json · Issue #76 · colcon/colcon-cmake - GitHub, accessed July 5, 2025, https://github.com/colcon/colcon-cmake/issues/76
14. CMake not generating compile_commands.json - c++ - Stack Overflow, accessed July 5, 2025, https://stackoverflow.com/questions/23960835/cmake-not-generating-compile-commands-json
15. Getting started - What is clangd? - LLVM, accessed July 5, 2025, https://clangd.llvm.org/installation
16. Build Your Own IDE with lsp-mode - System Crafters, accessed July 5, 2025, https://systemcrafters.net/emacs-from-scratch/build-your-own-ide-with-lsp-mode/
17. Emacs from Source Part 4: IDE Features with lsp-mode, company-mode, and go-mode - YouTube, accessed July 5, 2025, https://www.youtube.com/watch?v=UFPD7icMoHY
18. lsp-mode looks cool but how do you set it up????? : r/emacs - Reddit, accessed July 5, 2025, https://www.reddit.com/r/emacs/comments/i6ebl8/lspmode_looks_cool_but_how_do_you_set_it_up/
19. Emacs C++ mode coding aids, accessed July 5, 2025, https://web.physics.utah.edu/~detar/lessons/emacs/emacs/node7.html
20. CC Mode Manual - GNU.org, accessed July 5, 2025, https://www.gnu.org/software/emacs/manual/html_mono/ccmode.html
21. ludwigpacifici/modern-cpp-font-lock: C++ font-lock for Emacs - GitHub, accessed July 5, 2025, https://github.com/ludwigpacifici/modern-cpp-font-lock
22. C++ Programming in Emacs, accessed July 5, 2025, https://olddeuteronomy.github.io/post/cpp-programming-in-emacs/
23. c++ - Emacs - override indentation - Stack Overflow, accessed July 5, 2025, https://stackoverflow.com/questions/2619853/emacs-override-indentation
24. [question] how to disable indent in namespace c++-mode · Issue #3868 · syl20bnr/spacemacs - GitHub, accessed July 5, 2025, https://github.com/syl20bnr/spacemacs/issues/3868
25. Alexander-Miller/treemacs - GitHub, accessed July 5, 2025, https://github.com/Alexander-Miller/treemacs
26. Treemacs Manual | Cyven Chaney, accessed July 5, 2025, https://www.cybertheye.com/posts/treemacs-manual/
27. Debian -- Details of package elpa-treemacs-projectile in sid, accessed July 5, 2025, https://packages.debian.org/sid/elpa-treemacs-projectile
28. Confused about treemacs-projectile, want to keep them in sync always : r/emacs - Reddit, accessed July 5, 2025, https://www.reddit.com/r/emacs/comments/b50wmx/confused_about_treemacsprojectile_want_to_keep/
29. Configuration - DAP Mode, accessed July 5, 2025, https://emacs-lsp.github.io/dap-mode/page/configuration/
30. Dap-Mode LLDB Debug Template for Rust : r/emacs - Reddit, accessed July 5, 2025, https://www.reddit.com/r/emacs/comments/1b1d9l6/dapmode_lldb_debug_template_for_rust/
31. dap-lldb: depends on deprecated vscode extension · Issue #799 · emacs-lsp/dap-mode, accessed July 5, 2025, https://github.com/emacs-lsp/dap-mode/issues/799
