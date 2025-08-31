# WSL : Another Alternative for Working with Linux from VSCode in Windows

As developers, we often need to work in Linux environments — whether it’s building cloud-native apps, running Python data workflows, or compiling C++ projects. But if your primary workstation is **Windows**, you have multiple options for accessing Linux:

### Common Ways to Work with Linux from VSCode in Windows


When your project code lives in Linux but you’re working from Windows, there are a few ways to bridge the gap using Visual Studio Code:

1. **SSH into a Remote Linux Server**
   Use VS Code’s *Remote - SSH* extension to connect directly to a Linux machine. Your editor runs on Windows, while the code, compilers, and tools execute on the remote server.

   * **Pros:** Access to a full Linux environment; ideal if your project already runs on a remote host.
   * **Cons:** Requires stable network connectivity; files and dependencies live outside your local machine.

2. **Run Linux in a Virtual Machine (VM)**
   You can install Linux inside VirtualBox, VMware, or Hyper-V, then use VS Code to connect to it (often via SSH).

   * **Pros:** Full OS isolation, complete control over the Linux system.
   * **Cons:** Higher CPU and memory usage; file sharing between Windows and Linux VM can be clunky.

3. **Use Windows Subsystem for Linux (WSL)**
   Microsoft’s modern option: run a Linux distribution **directly inside Windows**, without the overhead of a full VM. With VS Code’s *Remote - WSL* extension, you can open Linux folders directly from Windows and edit them seamlessly.

   * **Pros:** Lightweight, fast startup, smooth file integration (`/mnt/c/...` for Windows, `\\wsl.localhost\...` for Linux), officially supported by Microsoft.
   * **Cons:** Some advanced kernel-level features may differ from a full Linux server.



This blog focuses on **WSL**, a new approach I recently explored, that makes it easy to edit Linux-based projects in VS Code while staying on a Windows workstation.




## What is WSL?

[Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/about) is a compatibility layer that allows you to run a real Linux distribution on Windows. Unlike virtual machines, WSL is tightly integrated into Windows:

* Run Linux commands in a terminal side-by-side with PowerShell.
* Access Linux files directly from Windows Explorer (`\\wsl$`).
* Choose between WSL 1 (translation layer) and WSL 2 (real Linux kernel with virtualization).

WSL 2 is now the recommended option — it gives you near-native Linux performance with full system call compatibility.



## Why Pair WSL with Visual Studio Code?

[Visual Studio Code’s Remote - WSL extension](https://code.visualstudio.com/docs/remote/wsl) makes the Linux experience seamless. Here’s what happens behind the scenes:

* You open a Linux project by running `code .` inside your WSL terminal.
* VS Code launches on Windows, but it starts a lightweight **VS Code Server** inside WSL.
* Extensions, debuggers, and compilers now run **inside Linux**, not Windows.

This means your code executes in the same environment you’d deploy to in production, but you keep the comfort of Windows desktop tools.

---

## Benefits of VS Code + WSL

* **Single IDE, two worlds**: Use VS Code’s Windows interface while compiling and running in Linux.
* **Accurate dependencies**: Python, Node.js, C++, and other languages run against Linux libraries.
* **Extension support**: Most VS Code extensions work seamlessly inside WSL.
* **No dual-boot or heavy VM overhead**: Start a Linux shell instantly.



## Key Tip: Launch VS Code from WSL

A common gotcha: always open VS Code **from your WSL terminal** with:

```bash
code .
```

This ensures VS Code connects to Linux instead of Windows paths. If you open VS Code directly from the Windows Start Menu, you’ll still be in Windows, which can cause toolchain mismatches.

---

## Learning Note: File Access Between WSL and Windows

One of WSL’s superpowers is its **seamless file system integration**.

* From WSL (Linux → Windows):
  Your Windows drives are automatically mounted under `/mnt`. For example:

  ```bash
  cd /mnt/c/Users/yourName
  ls
  ```

  This lists the contents of your Windows `C:\Users\yourName` directory.

* From Windows (Windows → Linux):
  WSL stores its Linux filesystem separately. For example, files in `/home/yourName` live inside WSL’s virtual hard disk.
  You can access these Linux files from Windows Explorer or Run (`Win + R`) using:

  ```
  \\wsl.localhost\Ubuntu\home\yourName
  ```

### Why `\\wsl.localhost`?

`\\wsl.localhost` is a **special UNC path** that Windows uses to expose the WSL virtual file system. It works like a network share, even though the files are on the same machine.

* The `wsl.localhost` part acts as a “virtual hostname” for the WSL environment.
* The next path segment (`Ubuntu`, `Debian`, etc.) is your WSL distribution name.
* From there, you can navigate the Linux filesystem as if it were a shared folder.

This mechanism allows Windows applications (e.g., Explorer, Notepad, VS Code) to open and edit files that physically live inside the WSL environment.

---


## References

* [Microsoft Docs: About WSL](https://learn.microsoft.com/en-us/windows/wsl/about)
* [Visual Studio Code Docs: Remote - WSL](https://code.visualstudio.com/docs/remote/wsl)


