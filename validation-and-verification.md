# What question does validation and verification ask in software engineering?

## Answer

Validation and verification are the two complementary halves of evaluating software quality. Each answers a different question.

### The two questions

- **Verification: "Are we building the product right?"** Checks that the software conforms to its specification. It inspects the output against the documented requirements, design, and standards. Verification is internal and objective: reviews, static analysis, unit tests, and integration tests all verify that the artifact matches what it was supposed to be.
- **Validation: "Are we building the right product?"** Checks that the software actually satisfies the customer's real needs and intended use. Validation is external and empirical: user acceptance testing, demos, beta programs, and production telemetry all validate that the product solves the problem it was meant to solve.

### The relationship

- **Direction of comparison differs.** Verification compares the product *against the spec*. Validation compares the product *against the need*.
- **A spec can be wrong.** A system can pass every verification (it meets the spec perfectly) yet fail validation (the spec itself did not capture what the user actually wanted). This is the gap validation exists to close.
- **Failure modes differ.** A verification failure means the implementation is defective. A validation failure means the requirements or assumptions were defective.
- **Order in the pipeline.** Verification typically precedes validation: confirm the software does what was specified before checking whether the specification was the right one. But validation findings feed back into requirements, restarting the loop.

### Example

Requirement: "The login form rejects credentials after five failed attempts."

- Verification: a test asserts that the sixth attempt is blocked. It passes, so the implementation conforms to the spec.
- Validation: users complain they get locked out by routine typos and abandon the product. The feature meets the spec but does not serve the user's need, so it fails validation.

### In short

Verification confirms you implemented the specification correctly; validation confirms the specification was worth implementing. Both are required, and neither substitutes for the other.
