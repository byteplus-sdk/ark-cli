# Contributing to the BytePlus doctor Skill

Keep this Skill aligned with the BytePlus build, command tree, and official
product contracts.

## Rules

- Use English ASCII text only.
- Do not add `doctor report`; it is unsupported on BytePlus.
- Do not copy another product's console URLs, balance semantics, regions, or
  credentials.
- Keep every BytePlus doctor JSON object a subset of the corresponding Volc
  object. Do not add BytePlus-only output fields, even when a BytePlus API
  returns additional data.
- Keep `compliance.realname` limited to the `GetVerifyInfo` real-name result.
  Do not combine account-opening or payment-qualification status into it.
- Prefer live command discovery for catalogs that can grow, such as
  `doctor metrics list`.
- Keep commands in `SKILL.md` concise and move detailed behavior to one
  scope-specific reference.
- Preserve the product-neutral doctor decision skeleton: command family,
  identifier extraction, resource-first routing, request-id limitation,
  structured schema consumption, and write confirmation boundaries. Product
  differences belong in the facts and available paths, not in missing routing
  guidance.
- Compare the BytePlus source with another product's current doctor only to
  detect missing product-neutral concepts and schema drift. Never copy
  product-specific report workflows, model names, console URLs, error codes,
  or account semantics.
- Verify every documented command and flag against a freshly built BytePlus
  binary. A static grader score alone is not release evidence.
- Add an evaluation prompt for every routing or safety boundary.

## Code and Skill update sequence

1. Update the product-isolated doctor implementation and tests.
2. Update `skills-byteplus/arkcli-doctor/`.
3. Run the Skill validator and static grader from the repository
   `skill-creator/` toolchain.
4. Confirm the BytePlus tree contains no Han/CJK characters, invalid links,
   product markers, or references to unsupported `doctor report` behavior.
5. Run BytePlus Skill generation and checks.
6. Build the BytePlus product and verify every doctor `--help`, local catalog
   command, and error lookup documented by the Skill. Confirm read-only Doctor
   scopes do not register `--dry-run`.
7. Install the embedded bundle into an isolated directory and compare it with
   `dist/skillbundle-byteplus/generated/skills` by relative path and SHA-256.
8. Run BytePlus-tagged doctor and Skill tests, then the repository-required
   cross-product checks to prove isolation.

Official BytePlus documentation should be the source for public error codes,
hosts, IAM behavior, account readiness, VMP, and ModelArk capabilities. When an
API contract is uncertain, preserve an explicit unavailable or unknown result
instead of claiming unverified parity.
