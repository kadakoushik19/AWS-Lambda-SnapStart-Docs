<p align="center">
  <img src="assets/banner.svg" alt="Reducing Java Cold Starts with AWS Lambda SnapStart" width="100%">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-Lambda-FF9900?logo=awslambda&logoColor=white" alt="AWS Lambda">
  <img src="https://img.shields.io/badge/Java-21-007396?logo=openjdk&logoColor=white" alt="Java 21">
  <img src="https://img.shields.io/badge/Build-Maven-C71A36?logo=apachemaven&logoColor=white" alt="Maven">
  <img src="https://img.shields.io/badge/SnapStart-PublishedVersions-2EC46B" alt="SnapStart">
  <img src="https://img.shields.io/badge/Monitoring-CloudWatch-FF4F8B?logo=amazoncloudwatch&logoColor=white" alt="CloudWatch">
  <img src="https://img.shields.io/badge/Region-ap--south--1-0038A8" alt="Region ap-south-1">
</p>

<p align="center">
  <b>A hands-on demonstration of the Java cold-start problem in AWS Lambda,<br>
  and how SnapStart reduces it by restoring a pre-initialized snapshot.</b>
</p>

---

## ⚡ Result at a glance

| | Without SnapStart | With SnapStart | Change |
|---|:---:|:---:|:---:|
| **Startup phase** (CloudWatch) | `Init Duration` **3347.04 ms** | `Restore Duration` **549.09 ms** | 🟢 **−2797.95 ms (83.6%)** |
| Handler `Duration` | 10.49 ms | 48.39 ms | first run after restore |
| **Total time before response** | ≈ 3357.5 ms | ≈ 597.5 ms | 🟢 **≈ 5.6× faster** |
| Version invoked | `$LATEST` | `Version 4` | |

<p align="center">
  <img src="assets/results.svg" alt="Bar chart: Init Duration 3347.04 ms vs Restore Duration 549.09 ms" width="100%">
</p>

---

## 📑 Table of contents

1. [Project Overview](#1-project-overview)
2. [Problem Statement](#2-problem-statement)
3. [What is AWS Lambda SnapStart?](#3-what-is-aws-lambda-snapstart)
4. [Project Objectives](#4-project-objectives)
5. [Technologies Used](#5-technologies-used)
6. [System Architecture and Workflow](#6-system-architecture-and-workflow)
7. [AWS Lambda Configuration](#7-aws-lambda-configuration)
8. [Java Lambda Function and Testing](#8-java-lambda-function-and-testing)
9. [Cold Start and Initialization (without SnapStart)](#9-cold-start-and-initialization-without-snapstart)
10. [SnapStart Snapshot and Restore (with SnapStart)](#10-snapstart-snapshot-and-restore-with-snapstart)
11. [CloudWatch Monitoring and Observations](#11-cloudwatch-monitoring-and-observations)
12. [Without SnapStart vs With SnapStart](#12-without-snapstart-vs-with-snapstart)
13. [Project Structure and Screenshots](#13-project-structure-and-screenshots)
14. [Conclusion and Key Learning](#14-conclusion-and-key-learning)

---

## 1. Project Overview

This project demonstrates **AWS Lambda SnapStart** using **Java 21**.

The goal is to understand the **Java cold start problem** in AWS Lambda and show, with real CloudWatch logs, how SnapStart uses a **pre-initialized snapshot** to avoid repeating initialization work.

> [!NOTE]
> Everything was built and tested from the **AWS Management Console**, using **Java 21**, **Maven**, **AWS Lambda** and **Amazon CloudWatch**.

---

## 2. Problem Statement

> **Our problem is Java cold start in AWS Lambda.**
> When there is no ready environment, Lambda has to create an environment and initialize Java before it can run our code. This takes time and can **delay the first response**. Our project uses **AWS Lambda SnapStart** to reduce this startup delay.

```mermaid
flowchart LR
    A([📨 Request arrives]) --> B{Warm environment<br/>available?}
    B -- Yes --> F([⚡ Run handler])
    B -- No --> C[🏗️ Create new<br/>execution environment]
    C --> D[☕ Start JVM,<br/>load classes]
    D --> E[⚙️ Initialize<br/>application]
    E --> F
    style C fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style D fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style E fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style F fill:#D6F5E3,stroke:#2EC46B,color:#000
```

The orange steps are the **cold start**. The user is waiting during all of them, before any of our business logic runs.

### 🧑‍💻 Real-life example: Tatkal ticket booking at 10:00 AM

Thousands of users tap **"Book Now"** in the same second. Lambda scales out by creating many new environments, and **every new environment is a cold start**.

| ⏱️ | 🔴 Without SnapStart | 🟢 With SnapStart |
|---|---|---|
| 0.0 s | User taps "Pay & Book" | User taps "Pay & Book" |
| Startup | JVM starts, classes load, app initializes: **≈ 3.3 s** of spinner | Lambda restores the ready snapshot: **≈ 0.55 s** |
| Meanwhile | Impatient user taps again → duplicate request, more cold starts | Handler runs straight away |
| Outcome | Slow first response, the seat may already be gone | Fast, predictable first response |

> [!TIP]
> **Simple analogy:** a laptop that is **shut down** takes a long time to start, because the OS and apps must load again. A laptop in **sleep mode** is ready almost instantly. SnapStart is like waking from sleep: it restores an environment that was already initialized.

---

## 3. What is AWS Lambda SnapStart?

**AWS Lambda SnapStart** improves startup performance by doing the expensive initialization **once**, when you publish a version, instead of on every cold start.

```mermaid
flowchart LR
    subgraph P["📦 When a version is published (one time)"]
        direction LR
        A[Run init code] --> B[📸 Snapshot memory<br/>and disk state]
        B --> C[🔐 Encrypt and cache<br/>the snapshot]
    end
    subgraph I["🚀 On every cold start"]
        direction LR
        D[♻️ Restore snapshot] --> E[⚡ Run handler]
    end
    C -. reused .-> D
    style P fill:#EEF3FF,stroke:#0038A8
    style I fill:#EAFBF1,stroke:#2EC46B
```

Instead of repeating the complete Java startup, Lambda **resumes from the initialized state** saved in the snapshot.

> [!IMPORTANT]
> SnapStart works on **published versions** (for example `SNAPSTART:4`), not on `$LATEST`. That is why the setting is called `PublishedVersions`.

---

## 4. Project Objectives

| # | Objective | Status |
|:-:|---|:-:|
| 1 | Understand Java cold starts | ✅ |
| 2 | Create a Java Lambda function on the **Java 21** runtime | ✅ |
| 3 | Observe Lambda **initialization** in CloudWatch | ✅ |
| 4 | Enable **SnapStart** (`PublishedVersions`) | ✅ |
| 5 | **Publish** a Lambda version and invoke it | ✅ |
| 6 | Observe the **restore** process | ✅ |
| 7 | Analyze CloudWatch logs: **initialization vs restore** | ✅ |

---

## 5. Technologies Used

| Technology | Purpose |
|---|---|
| <img src="https://img.shields.io/badge/-AWS%20Lambda-FF9900?logo=awslambda&logoColor=white"> | Serverless function execution |
| <img src="https://img.shields.io/badge/-SnapStart-2EC46B"> | Snapshot-based startup optimization |
| <img src="https://img.shields.io/badge/-Java%2021-007396?logo=openjdk&logoColor=white"> | Lambda runtime |
| <img src="https://img.shields.io/badge/-Maven-C71A36?logo=apachemaven&logoColor=white"> | Build and package the Lambda JAR |
| <img src="https://img.shields.io/badge/-CloudWatch-FF4F8B?logo=amazoncloudwatch&logoColor=white"> | Monitoring and log analysis |
| <img src="https://img.shields.io/badge/-AWS%20Console-232F3E?logo=amazonwebservices&logoColor=white"> | Configuration and testing |
| <img src="https://img.shields.io/badge/-GitHub-181717?logo=github&logoColor=white"> | Source code and documentation |

---

## 6. System Architecture and Workflow

### 🏛️ Architecture

<p align="center">
  <img src="assets/architecture.svg" alt="Total project architecture: Build and publish lane creates the snapshot; the cold start lanes without and with SnapStart differ only in full Java init vs snapshot restore; both report to CloudWatch Logs" width="100%">
</p>

| Lane | What happens |
|---|---|
| 🛠️ **Build & publish** (one time) | Maven packages the Java 21 code → we publish a version with SnapStart on → Lambda runs init **once** and stores an encrypted snapshot |
| 🔴 **Without SnapStart** | Every new environment repeats the **full Java init** before the handler can run |
| 🟢 **With SnapStart** | Every new environment **restores the snapshot** and goes straight to the handler |
| 📊 **CloudWatch Logs** | Records `Init Duration` and `Restore Duration` so we can compare the two paths |

The two cold-start lanes are identical except for one step: **full Java init** (orange) vs **restore snapshot** (green).

### 🔄 Complete workflow

| Step | Action | Evidence |
|:-:|---|---|
| 1️⃣ | Create the Lambda function `SNAPSTART` | [📸 01](#7-aws-lambda-configuration) |
| 2️⃣ | Configure the **Java 21** runtime and handler | [📸 02](#7-aws-lambda-configuration) |
| 3️⃣ | Build with Maven and upload the JAR (SnapStart still `None`) | [📸 03](#7-aws-lambda-configuration) |
| 4️⃣ | Create a test event and invoke `$LATEST` | [📸 04 · 05](#8-java-lambda-function-and-testing) |
| 5️⃣ | Observe **initialization** in CloudWatch: `Init Duration` | [📸 06](#9-cold-start-and-initialization-without-snapstart) |
| 6️⃣ | Enable **SnapStart** → `PublishedVersions` | [📸 07](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 7️⃣ | **Publish** a new version (Version 4) | [📸 08](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 8️⃣ | Invoke **Version 4** | [📸 09](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 9️⃣ | Observe **restore** in CloudWatch: `Restore Duration` | [📸 10](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 🔟 | Compare initialization vs restore | [Section 12](#12-without-snapstart-vs-with-snapstart) |

---

## 7. AWS Lambda Configuration

| Setting | Value |
|---|---|
| Function name | `SNAPSTART` |
| Runtime | **Java 21** |
| Handler | `example.LambdaHandler::handleRequest` |
| Architecture | `x86_64` |
| Memory | **512 MB** |
| Ephemeral storage | 512 MB |
| Timeout | **10 seconds** |
| Package | Zip (Maven JAR, 13.5 kB) |
| SnapStart | `PublishedVersions` → tested on **Version 4** |
| Region | Asia Pacific (Mumbai) `ap-south-1` |

<details open>
<summary><b>📸 Lambda function list</b>: both test functions use the Java 21 runtime</summary>
<br>
<img src="screenshots/01-lambda-functions.png" alt="Lambda functions list showing SNAPSTART on Java 21">
</details>

<details open>
<summary><b>📸 Runtime settings</b>: Java 21, handler <code>example.LambdaHandler::handleRequest</code>, x86_64</summary>
<br>
<img src="screenshots/02-runtime-settings.png" alt="Runtime settings: Java 21, handler and architecture">
</details>

<details open>
<summary><b>📸 General configuration before SnapStart</b>: 512 MB memory, 10 s timeout, SnapStart <code>None</code></summary>
<br>
<img src="screenshots/03-config-before-snapstart.png" alt="General configuration with SnapStart set to None">
</details>

---

## 8. Java Lambda Function and Testing

The function is intentionally simple. Its **initialization phase is made deliberately slow** so the cold-start cost is easy to see in the logs. The project is about **initialization vs restore behavior**, not about business logic.

```java
package example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import java.util.Map;

public class LambdaHandler implements RequestHandler<Map<String, Object>, String> {

    // Runs during the INIT phase (or once, before the snapshot, with SnapStart)
    static {
        System.out.println("===== APPLICATION INITIALIZATION STARTED =====");
        long start = System.currentTimeMillis();
        try {
            Thread.sleep(3000); // simulate heavy startup: frameworks, config, clients
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("Initialization completed in " + (System.currentTimeMillis() - start) + " ms");
        System.out.println("===== APPLICATION INITIALIZATION COMPLETED =====");
    }

    @Override
    public String handleRequest(Map<String, Object> event, Context context) {
        System.out.println("===== LAMBDA HANDLER EXECUTED =====");
        System.out.println("Request ID: " + context.getAwsRequestId());
        return "Hello " + event.getOrDefault("name", "World");
    }
}
```

**Build the deployment package:**

```bash
mvn clean package
```

Upload the JAR from `target/` in **Code → Upload from → .zip or .jar file**.

### 🧪 Testing steps

1. Open the Lambda function and go to the **Test** tab.
2. Create a test event (for example `TEST0`) with this JSON:
   ```json
   { "name": "Koushik" }
   ```
3. Click **Test** and confirm `Executing function: succeeded`.
4. Open **CloudWatch Logs** from the `logs` link.

<details open>
<summary><b>📸 Test event</b>: synchronous invocation with a simple JSON event</summary>
<br>
<img src="screenshots/04-test-event.png" alt="Test event configuration with JSON payload">
</details>

---

## 9. Cold Start and Initialization (without SnapStart)

On the **first invocation of `$LATEST`**, Lambda had no ready environment, so it went through the full cold start:

```mermaid
flowchart LR
    A[New execution<br/>environment] --> B[Start Java<br/>runtime] --> C[Load<br/>classes] --> D[Initialize<br/>application] --> E[Execute<br/>function]
    style B fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style C fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style D fill:#FFE3D1,stroke:#FF8A3D,color:#000
```

<details open>
<summary><b>📸 Test on <code>$LATEST</code></b>: executing function succeeded</summary>
<br>
<img src="screenshots/05-test-latest-success.png" alt="Successful test execution on $LATEST">
</details>

<details open>
<summary><b>📸 CloudWatch logs: initialization</b>: <code>Initialization completed in 3001 ms</code> and <code>Init Duration: 3347.04 ms</code></summary>
<br>
<img src="screenshots/06-cloudwatch-init-logs.png" alt="CloudWatch logs showing INIT_START and Init Duration 3347.04 ms">
</details>

```text
INIT_START Runtime Version: java:21.mainline.v83
===== APPLICATION INITIALIZATION STARTED =====
Initialization completed in 3001 ms
===== APPLICATION INITIALIZATION COMPLETED =====
START RequestId: a3da16be-...  Version: $LATEST
===== LAMBDA HANDLER EXECUTED =====
REPORT ... Duration: 10.49 ms  Billed Duration: 3358 ms  Memory Size: 512 MB  Max Memory Used: 104 MB  Init Duration: 3347.04 ms
```

> [!WARNING]
> The handler itself took only **10.49 ms**, but the user waited **3347.04 ms** for initialization first. That is the cold-start problem in one line.

---

## 10. SnapStart Snapshot and Restore (with SnapStart)

### Step 1: Enable SnapStart

**Configuration → General configuration → Edit → SnapStart: `PublishedVersions`**

<details open>
<summary><b>📸 SnapStart enabled</b>: <code>PublishedVersions</code></summary>
<br>
<img src="screenshots/07-snapstart-enabled.png" alt="General configuration with SnapStart set to PublishedVersions">
</details>

### Step 2: Publish a version

**Actions → Publish new version.** Lambda runs the initialization **once**, takes the snapshot and caches it.

<details open>
<summary><b>📸 Published Version 4</b>: SnapStart optimization status <code>On</code></summary>
<br>
<img src="screenshots/08-published-version.png" alt="Version 4 with SnapStart optimization status On">
</details>

### Step 3: Invoke the published version

<details open>
<summary><b>📸 Test on Version 4</b>: executing function succeeded</summary>
<br>
<img src="screenshots/09-test-version-success.png" alt="Successful test execution on Version 4">
</details>

<details open>
<summary><b>📸 CloudWatch logs: restore</b>: <code>RESTORE_START</code>, <code>RESTORE_REPORT Restore Duration: 549.09 ms</code></summary>
<br>
<img src="screenshots/10-cloudwatch-restore-logs.png" alt="CloudWatch logs showing RESTORE_START and Restore Duration 549.09 ms">
</details>

```text
RESTORE_START Runtime Version: java:21.mainline.v83
RESTORE_REPORT Restore Duration: 549.09 ms
START RequestId: 7d3eecf1-...  Version: 4
===== LAMBDA HANDLER EXECUTED =====
REPORT ... Duration: 48.39 ms  Billed Duration: 192 ms  Memory Size: 512 MB  Max Memory Used: 106 MB  Restore Duration: 549.09 ms  Billed Restore Duration: 143 ms
```

> [!NOTE]
> The `APPLICATION INITIALIZATION` lines **do not appear** here. The static initializer already ran before the snapshot was taken, so Lambda **restored** that state instead of running it again.

---

## 11. CloudWatch Monitoring and Observations

| Log entry | Seen in | What it means |
|---|---|---|
| `INIT_START` | `$LATEST` | Lambda started a fresh environment and the full Java init |
| `Initialization completed in 3001 ms` | `$LATEST` | Our deliberately slow static block ran |
| `Init Duration: 3347.04 ms` | `$LATEST` | Total init time: JVM startup, class loading and our code |
| `RESTORE_START` | Version 4 | Lambda began restoring the SnapStart snapshot |
| `RESTORE_REPORT Restore Duration: 549.09 ms` | Version 4 | Time to resume the already-initialized environment |
| `Billed Restore Duration: 143 ms` | Version 4 | Restore time shown as billed in the REPORT line |

**Overall observation**

```diff
- Initialization → creating and preparing the environment from scratch   (3347.04 ms)
+ Restore        → recovering the previously initialized state           ( 549.09 ms)
```

---

## 12. Without SnapStart vs With SnapStart

<table>
<tr>
<th width="50%">🔴 Without SnapStart</th>
<th width="50%">🟢 With SnapStart</th>
</tr>
<tr>
<td>

```mermaid
flowchart TB
    A([Request]) --> B[Create new environment]
    B --> C[Initialize Java]
    C --> D[Initialize application]
    D --> E([Execute function])
    style C fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style D fill:#FFE3D1,stroke:#FF8A3D,color:#000
```

</td>
<td>

```mermaid
flowchart TB
    A([Request]) --> B[Create new environment]
    B --> C[♻️ Restore snapshot]
    C --> E([Execute function])
    style C fill:#D6F5E3,stroke:#2EC46B,color:#000
```

</td>
</tr>
</table>

| Aspect | 🔴 Without SnapStart | 🟢 With SnapStart |
|---|---|---|
| What happens on a cold start | Full Java initialization | Snapshot restoration |
| Startup time (measured) | **3347.04 ms** `Init Duration` | **549.09 ms** `Restore Duration` |
| Init code runs | On **every** new environment | **Once**, when the version is published |
| Snapshot | ❌ None | ✅ Encrypted, cached snapshot |
| Works on | `$LATEST` or any version | **Published versions only** |

> [!CAUTION]
> These numbers come from **our own test runs** with a deliberately slow initializer. Real improvements depend on the application, its dependencies and its configuration, so always measure.

### Things to keep in mind

- 🏷️ **Versioning discipline:** SnapStart needs published versions (or an alias that points to one).
- 🔁 **Uniqueness:** anything created during init (random seeds, unique IDs, timestamps) is **shared by every restored environment**. Generate unique values inside the handler.
- 🌐 **Connections:** network connections opened during init may be stale after a restore. Re-validate or reconnect.

---

## 13. Project Structure and Screenshots

```text
AWS-Lambda-SnapStart-Docs/
├── README.md
├── assets/
│   ├── banner.svg
│   ├── architecture.svg
│   └── results.svg
└── screenshots/
    ├── 01-lambda-functions.png
    ├── 02-runtime-settings.png
    ├── 03-config-before-snapstart.png
    ├── 04-test-event.png
    ├── 05-test-latest-success.png
    ├── 06-cloudwatch-init-logs.png
    ├── 07-snapstart-enabled.png
    ├── 08-published-version.png
    ├── 09-test-version-success.png
    └── 10-cloudwatch-restore-logs.png
```

> [!NOTE]
> The deployed Lambda JAR (`JavaSnapStartDemo-1.0.jar`) and the project presentation are in the main project repository: **[kadakoushik19/AWS-Lambda-SnapStart](https://github.com/kadakoushik19/AWS-Lambda-SnapStart)**.


| # | Screenshot | Shows | Section |
|:-:|---|---|:-:|
| 01 | `01-lambda-functions.png` | Lambda functions on Java 21 | [7](#7-aws-lambda-configuration) |
| 02 | `02-runtime-settings.png` | Runtime, handler, architecture | [7](#7-aws-lambda-configuration) |
| 03 | `03-config-before-snapstart.png` | 512 MB, 10 s, SnapStart `None` | [7](#7-aws-lambda-configuration) |
| 04 | `04-test-event.png` | Test event JSON | [8](#8-java-lambda-function-and-testing) |
| 05 | `05-test-latest-success.png` | Successful run on `$LATEST` | [9](#9-cold-start-and-initialization-without-snapstart) |
| 06 | `06-cloudwatch-init-logs.png` | `Init Duration: 3347.04 ms` | [9](#9-cold-start-and-initialization-without-snapstart) |
| 07 | `07-snapstart-enabled.png` | SnapStart `PublishedVersions` | [10](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 08 | `08-published-version.png` | Version 4, optimization `On` | [10](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 09 | `09-test-version-success.png` | Successful run on Version 4 | [10](#10-snapstart-snapshot-and-restore-with-snapstart) |
| 10 | `10-cloudwatch-restore-logs.png` | `Restore Duration: 549.09 ms` | [10](#10-snapstart-snapshot-and-restore-with-snapstart) |

---

## 14. Conclusion and Key Learning

```mermaid
flowchart LR
    A[☕ Java Lambda] --> B["🥶 Cold start<br/>3347 ms init"]
    A --> C["📸 SnapStart<br/>snapshot on publish"]
    C --> D["♻️ Restore<br/>549 ms"]
    B --> E[⚡ Handler]
    D --> E
    E --> F[📊 CloudWatch]
    style B fill:#FFE3D1,stroke:#FF8A3D,color:#000
    style D fill:#D6F5E3,stroke:#2EC46B,color:#000
```

- ✅ The Java cold start in this project cost **3347.04 ms** of initialization before a **10 ms** handler could run.
- ✅ With SnapStart on a published version, startup became a **549.09 ms** restore: **83.6% less** startup time.
- ✅ **Lambda versions, SnapStart and CloudWatch logs** work together, and they make the whole lifecycle visible: `INIT_START` → `RESTORE_START` → `RESTORE_REPORT`.

> **Key learning:** SnapStart removes repeated Java initialization from the request path by restoring a previously initialized execution environment.
>
> 💻 *Cold start = starting a laptop from shutdown.* &nbsp; 😴 *SnapStart restore = waking it from sleep.*
> (This is only an analogy. Lambda restores a snapshot of an initialized microVM; it does not literally put a computer to sleep.)

---

## 👥 Team

| Name | Roll No. |
|---|---|
| N Ravindra | 2400030019 |
| M V Sai Kalyan | 2400030027 |
| Kada Koushik | 2400030046 |
| Jami Jaswanth | 2400030079 |

**Batch:** 117 &nbsp;·&nbsp; **Guided by:** Suneetha Bandeela

<p align="center"><sub>Project: <b>AWS Lambda SnapStart: Java Cold Start Demonstration</b> · Runtime: Java 21 · Platform: AWS Lambda</sub></p>
