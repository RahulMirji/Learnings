# 🚀 Daily CI/CD Knowledge Cheatsheet

This document contains everything you need to know about CI/CD, from the absolute basics down to advanced practices used in our exact repository. Keep this handy and review it whenever you need a refresher!

---

## 📖 Part 1: The Basics

### What is CI (Continuous Integration)?
**CI** is the practice of automatically and frequently integrating small code changes from developers into a shared repository.
* **Goal:** Catch bugs and integration errors early.
* **How it works:** Every time you push code, an automated system checks out the code, builds the app, and runs tests (linting, type-checking, unit tests). If it fails, the code is blocked.

### What is CD (Continuous Delivery / Continuous Deployment)?
* **Continuous Delivery:** Your code is always in a "deployable" state (e.g., the `.aab` or `.ipa` is built automatically), but a human must manually click to deploy it.
* **Continuous Deployment:** Every change that passes tests is deployed to production automatically without human intervention.
* **Goal:** Deliver software reliably and quickly.

### Why do we need it?
* Removes manual repetitive tasks (no building locally).
* Prevents broken code from being merged or deployed.
* Provides a consistent, clean sandbox environment unaffected by "it works on my machine" issues.

### Standard Pipeline Steps (Chronological)
1. **Linting & Formatting** (e.g., Prettier/Flutter Analyze)
2. **Type Checking**
3. **Automated Testing**
4. **Build** (Generate APK, AAB, IPA)
5. **Deploy** (Upload to TestFlight, Play Store, etc.)

---

## 🛠️ Part 2: Environments & Sandboxes

### The `actions/checkout` Step
```yaml
steps:
  - name: Checkout Code
    uses: actions/checkout@v3
```
When a pipeline starts, GitHub spins up a completely **empty, brand new virtual machine**. It has absolutely none of your files. The `checkout` step acts as an automated `git clone` to pull your repository into this empty sandbox so the pipeline has source code to work with. Without it, there is no code to compile.

### Ubuntu vs MacOS Runners
* **Android Builds (`ubuntu-latest`):** You can (and should) run Android builds on Linux. It perfectly supports the Android SDK and is significantly faster and cheaper to run.
* **Apple Builds (`macos-latest`):** You **must** use a macOS runner for iOS builds. Apple's build tools (Xcode, macOS SDK) are strictly proprietary and only work on the macOS operating system. When you use `macos-latest`, GitHub literally spins up a fresh Apple Mac Virtual Machine in their server farm, completely pre-installed with Xcode and ready to compile your app. This ephemeral Mac is destroyed as soon as the build finishes.

---

## 🧠 Part 3: Advanced Pipeline Practices (Found in your repo!)

### 1. Reusable Workflows (`workflow_call`)
Instead of copy/pasting your lint and test steps into every pipeline, you write one `ci.yml` pipeline. Other pipelines like `android-build.yml` can trigger and wait for `ci.yml` to succeed before they build. This keeps the configuration DRY (Don't Repeat Yourself).

### 2. State Caching (`actions/cache`)
Downloading packages (like CocoaPods or Gradle plugins) takes minutes. Caching saves the downloaded folders to GitHub's storage. It uses unique fingerprints (like hashing `Podfile.lock`). If no dependencies change, it restores instantly; if they do, it downloads fresh.

### 3. Automatic Concurrency Cancellation
```yaml
concurrency:
  group: android-build-${{ github.ref }}
  cancel-in-progress: true
```
Prevents race conditions and saves money. If you push code, then push a quick fix 10 seconds later, GitHub automatically kills the first pipeline build so you aren't running two at the same time.

### 4. Secrets Management
You never type passwords directly in code. Code like `${{ secrets.APP_STORE_CONNECT_API_KEY_BASE64 }}` securely pulls encrypted credentials from GitHub Settings straight into the virtual machine's memory, signs your app, masks it in the logs, and then destroys the machine.

### 5. Manual Triggers (`workflow_dispatch`)
Allows you to manually run a pipeline by clicking "Run Workflow" on GitHub's website instead of having to wait for a `push`. You can even configure UI dropdowns to select environments (e.g., `prod` vs `dev`).
