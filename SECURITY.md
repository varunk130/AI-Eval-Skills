# Security Policy

## Reporting a Vulnerability

Please report security issues privately rather than opening a public issue.

- **Preferred:** open a [private security advisory](https://github.com/varunk130/AI-Eval-Skills/security/advisories/new).
- Or contact the maintainer via their [GitHub profile](https://github.com/varunk130).

Include:

- A description of the issue and potential impact.
- Steps to reproduce or a proof-of-concept.
- Any suggested mitigations.

I aim to acknowledge reports within 7 days and provide a remediation update within 30 days.

## Handling Eval Data

These skills generate and consume eval datasets that may contain sensitive
prompts, completions, or user inputs. When reporting an issue, please:

- **Do not** attach raw datasets that contain PII or confidential prompts.
- Redact or hash sensitive content before sharing reproductions.
- If a vulnerability involves leaked eval data, describe the leak shape
  rather than including the data itself.

## Supported Versions

Only the latest commit on `main` receives fixes.

## Out of Scope

Issues in third-party model providers, judge models, or evaluation harnesses
should be reported to those projects directly.
