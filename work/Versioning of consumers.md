# Versioning of consumers

Within OEAPI it was agreed that consumer versioning would follow the major
version of the specification. At the same time, there is a clear principle
that minor version increments must remain non-breaking. In practice, these
two agreements are in tension.

Consider a consumer built against OEAPI 6. If that consumer introduces a
breaking change, there are two logical versioning paths, both with
drawbacks.

If the consumer moves from version 6.1 to 6.2, the breaking change would
violate the rule that minor versions are non-breaking. This undermines
predictability for integrators, who rely on minor upgrades being safe.

If instead the consumer moves from 6.1 to 7.0, the version number no longer
aligns with the OEAPI major version it is built upon. This breaks the
original intention that the consumer version reflects compatibility with a
specific OEAPI major release.

This reveals a structural conflict between alignment with OEAPI major
versions and strict semantic versioning for consumers. A consumer is not
purely a reflection of the OEAPI version, but an independently evolving
artefact with its own lifecycle and potential for breaking changes.

Therefore, either:

- the coupling between consumer major versions and OEAPI major versions must
  be relaxed,
- or the rule that minor versions are always non-breaking must be weakened
  for consumers.

## Option 1: Keep alignment with OEAPI major versions

**Pros**

- clear and simple mapping between consumer and OEAPI versions
- easy to understand for integrators which OEAPI major version is used
- consistent with the original design intention

**Cons**

- breaking changes may appear in minor versions
- reduces trust in version semantics
- increases risk for integrators when upgrading
- makes automated compatibility assumptions unreliable
- consumers cannot be used across different OEAPI versions

## Option 2: Use strict semantic versioning for consumers

**Pros**

- breaking changes are always clearly signalled via major versions
- minor versions remain safe to adopt
- aligns with widely accepted versioning practices
- improves predictability and tooling support
- consumers can be used across different OEAPI versions

**Cons**

- consumer version no longer directly reflects OEAPI major version
- requires additional mechanism to express OEAPI compatibility
- slightly more complex to interpret for integrators
