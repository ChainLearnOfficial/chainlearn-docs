# ADR-{NUMBER}: {TITLE}

- **Status:** {Proposed | Accepted | Deprecated | Superseded by ADR-XXX}
- **Date:** {YYYY-MM-DD}
- **Deciders:** {List of people involved in the decision}

## Context

Describe the forces at play, including technological, political, social, and project local. These forces are likely in tension, and should be called out as such. The language in this section is value-neutral. It is simply describing facts.

### Requirements

- Requirement 1
- Requirement 2
- Requirement 3

## Decision

State the decision clearly and concisely. Use active voice: "We will..."

## Consequences

### Positive

- Benefit 1
- Benefit 2
- Benefit 3

### Negative

- Drawback 1
- Drawback 2

### Risks

- Risk 1 and mitigation
- Risk 2 and mitigation

## Alternatives Considered

### Alternative 1

Description of the alternative and why it was rejected.

### Alternative 2

Description of the alternative and why it was rejected.

## Implementation Notes

Optional section for technical details about how the decision will be implemented.

## References

- Link to relevant documentation
- Link to related ADRs
- Link to external resources

---

## How to Use This Template

1. Copy this file and rename it: `adr-NNN-short-title.md`
2. Replace all `{PLACEHOLDER}` values
3. Fill in each section with relevant information
4. Submit a PR to the `chainlearn-docs` repository
5. The ADR is in "Proposed" status until approved by the team
6. Once approved, update status to "Accepted"

### ADR Numbering

ADRs are numbered sequentially starting from 001. Use zero-padded three-digit numbers:
- `adr-001-soroban-over-ethereum.md`
- `adr-002-sep41-token-standard.md`
- `adr-003-ai-provider-selection.md`

### Status Lifecycle

```
Proposed -> Accepted -> (optionally) Deprecated or Superseded
```

- **Proposed** -- Under discussion, not yet decided
- **Accepted** -- Decision made and implemented
- **Deprecated** -- No longer relevant (e.g., technology changed)
- **Superseded** -- Replaced by a newer ADR (link to the replacement)

### Tips for Writing Good ADRs

- **Be specific.** "We chose X because it's better" is not useful. "We chose X because it reduces latency by 50ms at our expected load" is useful.
- **Include trade-offs.** Every decision has downsides. Document them honestly.
- **Capture context, not just conclusions.** Future readers need to understand why a decision was made, not just what was decided.
- **Keep it focused.** One decision per ADR. If a decision is complex, break it into multiple ADRs.
- **Write for someone joining the team in 6 months.** They should be able to read the ADR and understand the decision without additional context.
