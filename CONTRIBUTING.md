# Contributing to draft-ratnawat-httpapi-async-problem-details

Thank you for your interest in this Internet-Draft! Contributions are welcome in several forms.

## Ways to Contribute

### 1. Review the Draft

Read [`draft/draft-ratnawat-httpapi-async-problem-details-00.md`](draft/draft-ratnawat-httpapi-async-problem-details-00.md) and open an issue with feedback on:

- Technical correctness
- Clarity of normative language (RFC 2119 keywords)
- Missing use cases or edge cases
- Security or privacy concerns
- Overlap with existing or emerging IETF work

### 2. Validate Examples

The [`examples/`](examples/) directory contains example Problem Details objects. Validate them against the [JSON Schema](schema/async-job-problem-details.schema.json) and report any inconsistencies.

### 3. Build Implementations

The IETF values running code. Reference implementations in any language/framework are welcome:

- **Server-side**: Produce Problem Details objects with the extension members
- **Client-side**: Parse and handle the extension members
- **Tooling**: Validation, monitoring dashboards, SDK generators

If you build an implementation, please open an issue or PR to add it to the README.

### 4. Propose Adoption in API Guidelines

See the [Adoption Playbook](adoption/RFC_ADOPTION_PLAYBOOK.md) for strategies on getting major API design guidelines (Zalando, Microsoft, gov.uk) to reference these extension members.

## Issue Guidelines

- **Bug in the draft**: Use the label `draft-bug`
- **Feature request / new extension member**: Use the label `extension-proposal`
- **Editorial / formatting**: Use the label `editorial`
- **Question / discussion**: Use the label `question`

## Pull Request Guidelines

1. Fork the repository
2. Create a feature branch (`git checkout -b fix/timestamp-constraint`)
3. Make your changes
4. Ensure JSON examples validate against the schema
5. Open a PR with a clear description of the change and its motivation

## Code of Conduct

This project follows the [IETF Guidelines for Conduct](https://www.rfc-editor.org/rfc/rfc7154) (RFC 7154). Be respectful, constructive, and focused on technical merit.

## IETF Process

This Internet-Draft is subject to the provisions of [BCP 78](https://www.rfc-editor.org/info/bcp78) and [BCP 79](https://www.rfc-editor.org/info/bcp79). By contributing, you agree that your contributions may be incorporated into an IETF document.

## Contact

- **Author**: Gaurav Ratnawat (gaurav.ratnawat@imtf.com)
- **IETF HTTPAPI WG Mailing List**: https://www.ietf.org/mailman/listinfo/httpapi
