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

---

## 🎤 Part 4: Interview Preparation (Explaining Your Pipelines)

If an interviewer asks you to explain the CI/CD pipelines you built for the App Store and Play Store, use this guide to answer confidently.

### The High-Level Pitch
**"I built automated CD pipelines using GitHub Actions for our Flutter app. For Android, I used an Ubuntu runner to generate and push `.aab` bundles directly to the Google Play Store. For iOS, I used a macOS runner to compile the `.ipa` and used Apple's `altool` to upload it to TestFlight."**

### Explaining the Android Pipeline (Step-by-Step)
1. **Trigger:** The pipeline runs on a push to `main` or via a manual run where I select the track (internal, alpha, etc).
2. **CI phase:** Calls the reusable `ci.yml` pipeline to ensure Tests and Linting pass.
3. **Caching:** Caches Gradle dependencies.
4. **Signing:** Pulls base64-encoded Keystore credentials from GitHub Secrets, decodes them, and creates a `key.properties` file locally on the runner.
5. **Build & Deploy:** Runs `flutter build appbundle` and uses `upload-google-play` to push to the store.

**Secrets Required:** `GCP_WORKLOAD_IDENTITY_PROVIDER`, `GCP_SERVICE_ACCOUNT_EMAIL`, `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_PASSWORD`, `ANDROID_KEY_ALIAS`.

### Explaining the iOS Pipeline (Step-by-Step)
1. **Trigger & CI:** Runs on `main`, executes `ci.yml` first.
2. **Setup:** Spins up a macOS VM, sets up Xcode, and caches CocoaPods.
3. **Keychain Sandbox:** Creates a temporary Apple Keychain exclusively for the CI runner.
4. **Signing Injection:** Decodes the `.p12` Certificate into the keychain and drops the `.mobileprovision` file into the system folder.
5. **Build & Deploy:** Generates an `ExportOptions.plist` formatting file, runs `flutter build ipa`, and uploads using Apple's official `xcrun altool`.

**Secrets Required:** `IOS_CERTIFICATE_P12_BASE64`, `IOS_CERTIFICATE_PASSWORD`, `KEYCHAIN_PASSWORD` (Runner Sandbox specific), `IOS_PROVISIONING_PROFILE_BASE64`, `APP_STORE_CONNECT_API_KEY_BASE64`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_ISSUER_ID`.

### "What challenges did you face?" (Crucial for sounding Senior)
Pick any of these to mention:
*   **Challenge 1: Binary Files in strict Secrets Management.** "GitHub Secrets only accept plain text, but Keystores and Certificates are binary files. I solved this by Base64-encoding the files locally, saving that text string in GitHub Secrets, and then writing a `base64 --decode` step in the pipeline to securely rebuild the files inside the runner."
*   **Challenge 2: Duplicate Build Numbers from merge collisions.** "App stores reject duplicate build numbers. I wrote a bash script to dynamically parse our base version using regex, then append `${{ github.run_number }}` with a rigid +1000 offset for Android and +2000 offset for iOS to guarantee they increment automatically and never collide."
*   **Challenge 3: Build Times.** "macOS build minutes are expensive and slow. I implemented strict `actions/cache` using `hashFiles('Podfile.lock')` which chopped iOS build times down by over 70%."

### 💡 Extra Interview Q&A (Deep Dives)

**Q: "If a unit test fails, what happens to the pipeline?"**
**A:** "The pipeline fails fast. Because the `build` jobs are configured with `needs: ci`, the pipeline immediately stops and throws a red 'X'. It never even attempts to build the APK or IPA, which guarantees that broken code is fundamentally blocked from reaching the App Store or Play Store."

**Q: "Why did you use Workload Identity Federation (WIF) instead of a JSON Service Account Key for Google Play deployment?"** 
*(Note: Interviewers will be highly impressed if you bring this up!)*
**A:** "Historically, people export a static JSON password file for Google Cloud. But these keys never expire, which is a major security risk if leaked. Instead, I configured **Workload Identity Federation**, which uses OIDC (OpenID Connect). GitHub essentially talks to Google Cloud directly to generate a temporary, short-lived token that is only valid for the exact duration of the pipeline run. It's significantly more secure."

**Q: "Why didn't you use Fastlane? A lot of mobile devs use Fastlane for App Stores."**
**A:** "While Fastlane is great, it adds a massive Ruby dependency layer. By using native GitHub Actions (like the `r0adkll` action for Google Play, and Apple's native `xcrun altool` CLI for TestFlight), I minimized the pipeline bloat. This gets rid of unnecessary dependencies, makes the pipelines faster, and keeps the configuration 100% in pure YAML."

**Q: "How do you ensure someone doesn't accidentally log out a password in the pipeline?"**
**A:** "GitHub Actions automatically masks any value registered as a 'Secret'. If a junior developer accidentally types `echo ${{ secrets.PASSWORD }}`, GitHub intercepts the log in real-time and prints `***` to the console."

**Q: "What are 'Reusable Workflows' and why bother with them?"**
**A:** "Before, I had to copy the exact same testing and linting commands into both the Android and iOS release files. If we added a new test, I had to update multiple files. By using `workflow_call` to create a reusable `ci.yml` file, I made the architecture completely DRY (Don't Repeat Yourself). Now, all platforms dynamically inherit the exact same testing standards from one single source of truth."
