# How are acceptance criteria related to requirements?

## Answer

Requirements and acceptance criteria are related but distinct artifacts that sit at different levels of the same pipeline. Requirements describe **what** the system must do; acceptance criteria describe **how we know** it does it.

### The relationship

- **Requirements capture the need.** They state the functional and non-functional behavior the system must provide (problem space). A requirement answers "what should be true of the system?"
- **Acceptance criteria operationalize the requirement into testable checks.** Each criterion is a concrete, pass/fail condition that, when satisfied, proves the requirement is met. This is the seam between problem space and solution space (the bridge from spec to test).
- **Granularity differs.** A single requirement usually decomposes into multiple acceptance criteria. Requirements are coarser; acceptance criteria are finer and more specific.
- **Perspective differs.** Requirements are often written in system or stakeholder terms. Acceptance criteria are frequently written from the user's perspective and in an executable form (e.g., Given/When/Then).
- **Ownership.** Product/analysis owns the requirements (the "what/why"). Engineering owns "how," but the acceptance criteria themselves are jointly negotiated so both sides agree on the contract.

### Example

Requirement:
> As a returning customer, I can log in with my email and password so that I can access my account.

Acceptance criteria (one per checkable condition):
1. Given a registered user, when valid credentials are submitted, then the user is authenticated and redirected to their dashboard.
2. Given an unregistered email, when login is attempted, then an error is shown and no session is created.
3. Given five consecutive failed attempts, then the account is temporarily locked for 15 minutes.

### In short

Requirements are the input (the obligation). Acceptance criteria are the contract that confirms the obligation has been discharged. Without acceptance criteria, a requirement is untestable; without requirements, acceptance criteria have no reason to exist.
