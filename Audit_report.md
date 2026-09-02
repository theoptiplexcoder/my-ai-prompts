Perform a strict, comprehensive, and highly critical audit of the existing codebase as it is today.
Do NOT refer to any PRD, requirements document, or intended feature list. Assume the current codebase is the only source of truth.
Your objective is to determine:
* Whether the system is truly production-ready
* What is broken, risky, incomplete, or fragile
* What will fail under real production usage
* What violates engineering and security standards

Core Objective
You are acting as a senior staff engineer + production release gatekeeper.
This codebase must be evaluated as if it is about to be deployed to a high-traffic, real-world production environment.
Be strict. Be skeptical. Do not assume correctness. Do not be lenient.

1. System-Wide Stability Review
Assess whether the system is structurally stable:
* Does the application run end-to-end without hidden failures?
* Are there broken flows, partial implementations, or incomplete logic paths?
* Are there any features that exist only partially implemented?
* Are there unhandled edge cases that will crash real usage?
* Are there inconsistent behaviors across modules?
Identify any non-deterministic or fragile behavior.

2. Architecture & Design Integrity
Evaluate architectural soundness:
* Is the architecture coherent or fragmented?
* Are boundaries between modules clean or leaking?
* Are responsibilities properly separated or mixed?
* Are there circular dependencies or tight coupling?
* Is the structure scalable or already showing strain?
* Are there anti-patterns (God objects, fat services, logic in controllers/UI)?
Call out any design that will not scale under growth.

3. Code Quality (Strict Inspection)
Review all code with zero tolerance for poor engineering practices:
Check for:
* Overcomplicated logic
* Duplicated logic
* Poor naming or misleading abstractions
* Unreadable or overly dense functions
* Lack of modularity
* Hidden side effects
* Misuse of frameworks
* Inconsistent patterns
* Dead or unused code
* Hardcoded values
* Magic strings/numbers
* Poor error handling logic
Any deviation from clean, maintainable code must be flagged.

4. API & Backend Behavior
Audit all APIs and backend flows:
* Are APIs consistent in structure and behavior?
* Are response formats standardized?
* Are error responses reliable and predictable?
* Are HTTP status codes used correctly?
* Are validations strict and complete?
* Are authentication and authorization enforced everywhere correctly?
* Are there any exposed or unsafe endpoints?
* Are there race conditions or concurrency issues?
Highlight any API that could fail under real traffic or malicious input.

5. Database & Data Integrity
Evaluate database design strictly:
* Are relationships correct and enforced?
* Are constraints properly defined?
* Are there missing indexes affecting performance?
* Is normalization correct or inconsistent?
* Are there risks of data corruption?
* Are transactions used correctly where needed?
* Are there orphan data risks?
* Is schema design future-proof or brittle?
Flag anything that risks data integrity loss or inconsistency.

6. Frontend / UI Stability (if applicable)
Review frontend implementation:
* Are UI states complete (loading, error, empty)?
* Are components reusable and consistent?
* Is state management stable or chaotic?
* Are there rendering performance issues?
* Are there UX inconsistencies or broken flows?
* Are forms properly validated?
* Are API failures handled gracefully?
* Are there edge-case UI crashes?
Identify any UI behavior that would break user experience in production.

7. Security Audit (Strict Mode)
Perform a full security review with no assumptions of safety:
Check for:
* Authentication bypass possibilities
* Authorization gaps (horizontal/vertical escalation)
* Input validation weaknesses
* Injection risks (SQL, NoSQL, command injection)
* XSS vulnerabilities
* CSRF risks
* SSRF exposure
* Sensitive data leaks (logs, responses, frontend exposure)
* Token/session mismanagement
* Unsafe file handling
* Weak encryption or missing encryption
* Misconfigured CORS or headers
* Dependency vulnerabilities
Treat every input boundary as potentially exploitable.

8. Performance & Efficiency
Identify performance risks:
* N+1 query problems
* Excessive API calls
* Inefficient loops or computations
* Missing caching where needed
* Heavy synchronous operations
* Large bundle sizes or bloated dependencies
* Unoptimized rendering or data processing
* Uncontrolled memory usage patterns
Flag anything that would degrade under real load.

9. Reliability & Failure Handling
Evaluate system resilience:
* Are errors properly caught and handled?
* Are retries implemented where needed?
* Are failure states recoverable?
* Are logs meaningful and structured?
* Are crash scenarios possible from unhandled exceptions?
* Is the system resilient to partial failures?
Identify single points of failure.

10. Maintainability & Technical Debt
Assess long-term maintainability:
* Is the code easy to extend or fragile?
* Are there accumulating workarounds?
* Is technical debt visible or hidden?
* Are abstractions helping or hurting?
* Will onboarding new engineers be difficult?
Call out anything that will become unmanageable over time.

11. Production Readiness Verdict
Give a strict assessment:
* Is the system production-ready? (Yes / No / With Major Risks)
* What will break first in production?
* What are the top 10 critical risks?
* What must be fixed before deployment (blocking issues)?
* What can be deferred?

12. Severity Classification (Mandatory)
Every issue must include:
* Severity: Critical / High / Medium / Low
* Category: Security / Performance / Architecture / Logic / UI / Data / API
* Impact in real production scenario
* Root cause
* Exact location (file/module/function)
* Recommended fix
* Priority order

Final Output Requirements
Provide:
1. Executive summary of system health
2. Production readiness verdict
3. Top critical blockers
4. Full categorized issue list
5. Risk assessment (what fails first under real usage)
6. Engineering maturity score (0–100)

Non-Negotiable Standard
* Do NOT assume correctness
* Do NOT skip small issues
* Do NOT be lenient
* Do NOT ignore inconsistencies
* Do NOT treat “working code” as “correct code”
* Treat this as a pre-production gate review for a high-scale system
The goal is to uncover every structural, security, performance, and maintainability weakness before production deployment.