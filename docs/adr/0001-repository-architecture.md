# ADR-0001: Repository and Application Architecture

**Status:** Proposed

## 1. Context

**What are we building?**
We are building a vulnerability detection and triage platform that integrates multiple security scanners and uses AI agents to perform evidence-grounded triage and suggest remediation steps. Instead of claiming to definitively prove exploitability, the platform categorizes findings into four evidence-based outcomes:
* **Supported finding:** Available evidence supports the vulnerability.
* **Likely false positive:** Evidence suggests the alert is not applicable.
* **Insufficient evidence:** More runtime or application context is needed.
* **Dynamically verified:** Authorized runtime testing produced confirming evidence.

We are building the system in two parts: an intentionally vulnerable shop application to benchmark detection accuracy, and a standalone security platform that orchestrates scanners and AI triage. Both live in a single Git monorepo for ease of demonstration as a learning proof of concept, though they run as separate applications.

**Why do we need both a vulnerable application and an AI security platform?**
Because scanning arbitrary public-facing targets without explicit permission is off-limits, we need a controlled local lab target. The vulnerable shop provides known security issues and non-exploitable alerts so we can evaluate whether our AI triage accurately distinguishes supported findings from likely false positives.

**What problems are we trying to avoid?**
We want loose coupling and clear module boundaries so adding new scanners or triage logic does not require rewriting existing core features. Each scanner produces its own output format; rather than forcing our database schema to mirror an external standard like SARIF, we need an internal domain model tailored to our platform's workflows (such as correlation, AI evidence, and scan status), while deferring final normalization details to a future ADR.

## 2. Decision

* **Why we use one Git repository:** Keeping both the vulnerable shop and the security platform in a monorepo makes local testing, version tracking, and proof-of-concept demonstrations straightforward while keeping their runtime lifecycles independent.
* **Why the security platform and vulnerable shop are separate applications:** Each serves a distinct purpose—one is the target under test, and the other is the analysis platform. Keeping them separate mirrors real-world usage against authorized targets in a controlled local lab.
* **Why the security platform will be a modular monolith:** A modular monolith gives us simple deployment and well-defined internal boundaries without the operational overhead of microservices.

## 3. Alternatives Considered

* **Microservices:** Unnecessary for a solo-developer learning project. Splitting the platform into distributed services early on would introduce deployment, networking, and operational complexity without meaningful gain.
* **Unstructured / Traditional Monolith:** A monolith is simply an application built and deployed as a single unit—it does not automatically cause tight coupling. However, a *poorly structured* monolith where modules directly import each other's internal state or database connections risks tight coupling, making future changes harder to isolate and maintain. Enforcing modular boundaries avoids this pitfall.

## 4. Benefits

* **Low operational complexity with clean boundaries:** We avoid distributed-system overhead while keeping scanner adapters, triage logic, and storage cleanly separated.
* **Platform-driven internal data model:** Preserving raw scanner outputs and mapping them through scanner adapters into an internal `SecurityFinding` domain model lets us store application-specific data (finding correlations, supporting AI evidence, scan failure states) and optionally export to formats like SARIF later. While new scanners may still occasionally require manageable schema updates, our core platform logic remains insulated from vendor-specific formats.
* **Straightforward testing:** Unit and integration tests can verify each module and scanner adapter independently.

## 5. Tradeoffs

* **Deployment and resource coupling:** Initially deploying the platform as a single application simplifies development and operations, without preventing us from invoking external binaries (written in Go, Rust, or C++) via subprocesses or native extensions. However, because components share a single deployment unit and resource pool, resource-heavy scanner workers cannot be scaled or released independently unless we later extract them into separate worker processes or containers.

## 6. Dependency Boundaries

* **Runtime and code isolation:** The vulnerable shop and security platform have separate runtime responsibilities. The security platform never imports the vulnerable shop's private Python packages or business logic.
* **Supported interaction interfaces:** The security platform interacts with the target strictly through stable, authorized boundaries:
  * **Static analysis:** Reading a repository snapshot or source files of the target application (which will require isolated, resource-limited execution when scanning untrusted code).
  * **Dynamic testing:** Sending authorized HTTP requests to the running target application through its public endpoints.
* **Internal normalization boundary:** Scanner adapters translate tool-specific outputs into our internal representation; the final normalization and SARIF export specification will be finalized in a dedicated ADR in Week 8.

## 7. Future Changes

Suppose we add a Java Spring Boot target application.

The core platform workflow—ingesting a repository snapshot, running scanner adapters, and performing evidence-grounded AI triage—remains unchanged. However, we will need to configure Java-specific scanners (such as Semgrep rulesets or dependency checkers for Maven/Gradle), adjust build or artifact inspection settings if compiled bytecode is required, and update dynamic test endpoints to match the new application.