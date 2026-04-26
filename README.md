## Continuous Integration (CI) Builder System

This repository utilizes a highly advanced, fully automated Continuous Integration (CI) pipeline powered by GitHub Actions. The workflow allows developers and users to compile the kernel directly from the GitHub interface without requiring a local build environment. 

The CI system is designed to be highly modular, supporting matrix builds, dynamic kernel configuration injection, and automated packaging.

### Workflow Inputs

The build process is triggered manually via the `workflow_dispatch` event. When initiating a build, the user is presented with several configuration parameters:

* **Kernel Source Repository URL:** The target repository to clone.
* **Kernel Branch:** The specific branch to compile.
* **Target Device:** Defines the defconfig name used for the build (e.g., `gki` resolves to `gki_defconfig`).
* **Custom Localversion:** An optional string appended to the kernel version.
* **Select KSU Variants:** Dictates which root variants to build. Options include:
    * `vanilla`: Pure stock kernel with no modifications.
    * `ksu`: Integrated with KernelSU-Next (without SuSFS system patches).
    * `ksus`: Integrated with KernelSU-Next and fully injected with SuSFS for advanced root hiding.
    * `all`: Compiles all three variants concurrently using a build matrix.
* **Kernel Tick Rate (Hz):** Allows overriding the default kernel timer frequency (`100Hz`, `250Hz`, `300Hz`, `1000Hz`).
* **Compiler Optimization Level:** Modifies the GCC/Clang optimization flags (`O2`, `O3`, `Os`).
* **Select Build Toolchain:** * `android-tools`: Utilizes the official AOSP manifest sync and specifically locks the compiler to Clang `r547379` for strict GKI compliance.
    * `custom-tools`: Fetches the latest custom toolchain (Greenforce Clang) via API.
* **Link Time Optimization (LTO):** Sets the LTO level (`thin` or `full`).

### Build Architecture Overview

The GitHub Actions workflow is divided into two primary jobs:

#### 1. Compilation Job (`build-kernel`)
This job utilizes a matrix strategy. If the user selects the `all` variant, GitHub Actions will spawn parallel virtual environments to compile the `vanilla`, `ksu`, and `ksus` kernels simultaneously.

**Phases within this job:**
* **Environment Preparation:** Installs necessary dependencies (build-essential, rsync, aria2, llvm, etc.) and provisions a 20GB swap space to prevent memory exhaustion during LTO linking.
* **Toolchain Synchronization:** Depending on user input, it either synchronizes the AOSP build scripts and downloads Clang `r547379` directly, or fetches a custom Clang archive.
* **Dynamic Patching Protocol:**
    * If a KSU variant is selected, the system clones KernelSU-Next (specifically the `dev-susfs` branch) and applies the "All Managers Support" patch.
    * The system downloads SuSFS components and provides the necessary headers to the kernel source tree to satisfy compilation dependencies for all KSU variants.
    * If the specific `ksus` variant is currently being built, the system applies the core SuSFS system patches directly to the kernel source.
* **Dynamic Configuration:** The workflow uses `sed` and `echo` to dynamically modify the selected `defconfig` on the fly. It injects the requested LTO type, tick rate, optimization levels, and the necessary `CONFIG_KSU_SUSFS` flags based on the active matrix variant.
* **Compilation:** Executes the build using the official AOSP `build/build.sh` script (for Android tools) or standard `make` directives (for custom tools).
* **Artifact Generation:** Extracts the compiled `Image` file and uploads it securely as a temporary GitHub Artifact.

#### 2. Packaging and Deployment Job (`package-release`)
This job executes only after all compilation matrices have successfully completed.

**Phases within this job:**
* **Artifact Retrieval:** Downloads all compiled kernel images from the previous job.
* **AnyKernel3 Integration:** Clones the AnyKernel3 repository (using the `GKI` branch).
* **Variant Separation:** Iterates through the downloaded artifacts. For each variant found (`vanilla`, `ksu`, `ksus`), it creates a dedicated AnyKernel3 directory, places the corresponding `Image` inside, and compresses it into a flashable ZIP file. The ZIP is named dynamically, excluding the branch name for a cleaner output format.
* **GitHub Releases Publishing:** Utilizes `softprops/action-gh-release` to automatically draft a new Release on the GitHub repository. It uploads the separated ZIP files and generates release notes detailing the build date, target device, compiler optimizations, and variants included.
* **Telegram Telemetry:** Upon successful release creation, a structured status report is sent to a designated Telegram chat via API, providing a summary of the build and a direct link to the GitHub Release page.

### Repository Secrets Configuration

To enable automated Telegram notifications upon successful builds, you must configure repository secrets within your GitHub repository settings. 

To add these secrets, navigate to your repository on GitHub, click on **Settings**, scroll down to **Security** in the left sidebar, and select **Secrets and variables** > **Actions**. Click on **New repository secret** to add the following:

* **TELEGRAM_BOT_TOKEN**
    * **Description:** The API token required to authenticate your Telegram bot to send messages.
    * **How to obtain:** Open Telegram, search for `@BotFather`, and send the `/newbot` command. Follow the prompts to create a new bot. Once created, BotFather will provide an HTTP API token (e.g., `123456789:ABCdefGhIJKlmNoPQRstuVWxyz`). Copy this exact string into the secret value.

* **TELEGRAM_CHAT_ID**
    * **Description:** The unique identifier for the target chat (personal, group, or channel) where the build status notifications will be delivered.
    * **How to obtain for a personal chat:** Message a bot such as `@userinfobot` or `@RawDataBot` on Telegram to retrieve your personal account ID.
    * **How to obtain for a group or channel:** Add your newly created bot to the target group or channel as an administrator. Send a test message in the chat, then visit `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates` in a web browser. Locate the `"chat":{"id":...}` field in the JSON response. Group and channel IDs typically start with a `-` or `-100`.

**Note regarding GITHUB_TOKEN:** The deployment workflow utilizes `${{ secrets.GITHUB_TOKEN }}` to automatically publish the final packaged ZIP files to GitHub Releases. You do not need to create or configure this secret manually. GitHub automatically provisions this token dynamically during the workflow run, and the CI script explicitly grants it `contents: write` permissions to authorize the release creation process securely.
