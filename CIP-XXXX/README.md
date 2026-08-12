---
CIP: XXXX
Title: Plutus Blueprint Assurance Documents
Category: Tools
Status: Proposed
Authors:
    - Jean-Frédéric Etienne <jean-frederic.etienne@iohk.io>
    - Mark Petruska <mark.petruska@iohk.io>
    - Romain Soulat <romain.soulat@iohk.io>
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/CIPs/pull/?
Created: 2026-06-16
License: CC-BY-4.0
---

## Abstract

A [CIP-0057](../CIP-0057) blueprint describes the binary interface of a Plutus validator — its datum, redeemer and parameter schemas — alongside its compiled code. It says nothing about the validator's expected behaviour or the guarantees it provides.

This CIP defines the **assurance document**: a standalone, machine-readable JSON document through which any party — the contract's developers, an external auditor, or an independent community member — can publish claims about the validators described in a blueprint. Each claim states a property in natural language, optionally accompanied by a formal statement in a declared specification language, and is backed by zero or more evidence records stating who verified it, by what method (formal proof, property test, unit test, audit, manual review), with which tool, with what outcome, against which script hash, and where a reproducible verification artifact can be retrieved.

A third party can take an assurance document, confirm that it concerns the exact compiled code shipped in the blueprint, fetch the referenced artifacts, re-run the verification, and obtain the same verdicts. Trust rests on the reproducibility of the evidence and the soundness of the verification tools — not on the author's word.

## Motivation: why is this CIP necessary?

[CIP-0057](../CIP-0057) makes a validator's interface legible; it does not expose anything about the validator's behaviour. Today, "this validator is safe" is established by an off-chain, largely manual process whose end product is typically a PDF report from a trusted auditor. There is no standard, machine-readable way for developers or auditors to state which properties a validator satisfies, how those properties were verified, and how anyone can check the verification themselves.

The blueprint ecosystem is the natural anchor for such claims: a blueprint already carries the compiled validator, the datum and redeemer shapes, the required parameters, and a hash digest linking the validator to on-chain addresses. What is missing is a standard companion format for behavioural claims that binds to those elements.

This proposal enables, building directly on the use cases CIP-0057 already lists:

- **Reproducible verification**: anyone can re-derive a validator's verdicts from the referenced artifacts, rather than trusting an assertion.
- **Communicating guarantees**: wallets, explorers, and registries can display which properties are claimed for a validator, by whom, with what method and outcome — and whether the evidence still matches the deployed code.
- **Third-party assurance**: auditors and community members can publish assurance documents about a deployed contract *without any cooperation from its developers*, and several independent assurance documents about the same blueprint can coexist.
- **Health monitoring of live DApps**: the assumptions under which properties were verified are stated explicitly, so they can be monitored live to check that a running system stays within its verified envelope.
- **Specification-first development**: a property with no evidence records is a stated, unverified claim — a machine-readable specification target that implementations and verification efforts can work towards.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Overview

This specification introduces the notion of an **assurance document**: a standalone JSON document that publishes claims about the on-chain validators described by a [CIP-0057](../CIP-0057) blueprint.

An assurance document is *detached*: it is a separate document from the blueprint, and this CIP requires no change to `plutus.json`. Anyone — the contract's developers, an external auditor, an independent community member — can author and publish an assurance document about any blueprint. Multiple assurance documents about the same blueprint can coexist independently.

By convention, an assurance document authored by the contract developers is named `assurance.json` and located next to `plutus.json` at the root of the project's repository, to facilitate discoverability. Third-party documents can be published anywhere.

The meta-schema for assurance documents (i.e. the schema used for validating assurance documents themselves) is given [in annexe](./schemas/assurance.json). An assurance document identifies the format version it complies with through its `$schema` field.

A **producer** is any tool or person authoring assurance documents. A **consumer** is any tool interpreting them (e.g. a wallet, an explorer, a verification checker).

### Document structure

The document itself is a JSON object with the following fields:

| Fields      | Description                                                        |
| ---         | ---                                                                |
| `$schema`   | URI identifying the assurance document format version (required)   |
| preamble    | An object with meta-information about the assurance document       |
| blueprint   | A reference to the blueprint the claims are about                  |
| ?languages  | A registry of specification languages used by formal statements    |
| ?tools      | A registry of verification tools referenced by evidence records    |
| properties  | The list of claimed properties                                     |

#### `$schema`

The value of this field MUST be the URI of the meta-schema this document complies with. For this version of the specification, it is:

```
https://cips.cardano.org/cips/cipXXXX/schemas/assurance.json
```

The meta-schema pins this value with a `const`: a document claiming compliance with this version of the format carries exactly this URI.

Any breaking change to this specification MUST be published under a new URI. Additions of OPTIONAL fields MAY occur under the same URI. To make such additions possible, objects in an assurance document are deliberately open: consumers MUST ignore fields they do not recognize.

#### preamble

| Fields       | Description                                                          |
| ---          | ---                                                                  |
| title        | A short and descriptive title of the assurance document              |
| ?description | A more elaborate description                                         |
| ?version     | A version number for the assurance document itself                   |
| authors      | A non-empty list of authors of the document (free-form strings)      |
| created      | The creation date, as `YYYY-MM-DD`                                   |
| ?license     | A license under which the assurance document is distributed          |

#### blueprint

A reference to the blueprint the claims are about.

| Fields | Description                                                                              |
| ---    | ---                                                                                      |
| uri    | A URI from which the blueprint document can be retrieved                                 |
| ?hash  | A content digest of the blueprint document, as a [digest object](#digest-objects)        |

The `hash`, when present, MUST be computed over the raw bytes of the blueprint document exactly as retrieved from `uri`. Including it is RECOMMENDED: it makes the binding tamper-evident. Consumers MUST verify it when present and MUST treat the assurance document as *not applicable* to a blueprint whose bytes do not match.

#### digest objects

Everywhere a content digest appears, it is an object:

| Fields | Description                                                     |
| ---    | ---                                                             |
| alg    | The hash algorithm, e.g. `sha256`, `blake2b-256`                |
| digest | The lowercase, hex-encoded digest value                         |

#### languages and tools

Both fields are objects mapping a document-local identifier (used by `formal.language` and `evidence.tool` references) to a registry entry:

| Fields       | Description                                            |
| ---          | ---                                                    |
| name         | The name of the language or tool                       |
| version      | A version number, in any format                        |
| ?uri         | A URI to the language specification or tool homepage   |
| ?description | An informative description                             |

This CIP deliberately does **not** restrict which specification languages or verification tools can be referenced; see the Rationale. A reference to a key absent from the corresponding registry makes the document invalid; consumers MUST reject such documents.

#### properties

The essence of the assurance document: a list of claimed properties. Each property is an object:

| Fields       | Description                                                                                       |
| ---          | ---                                                                                               |
| id           | An identifier unique within the document, matching `^[A-Za-z0-9_-]+$`                             |
| ?title       | A short and descriptive name for the property                                                     |
| scope        | An object with a field `validators`: a non-empty list of [validator references](#referencing-validators) |
| statement    | A [statement object](#statements): what is being claimed                                          |
| ?assumptions | A list of statement objects: hypotheses under which the claim holds                               |
| ?tags        | A list of free-form classification strings (e.g. `safety`, `authorization`)                       |
| ?evidence    | A list of [evidence records](#evidence-records); when absent or empty, the property is a stated, unverified claim |

#### Statements

A statement (used for both the claimed property and its assumptions) is an object:

| Fields  | Description                                                                       |
| ---     | ---                                                                               |
| text    | A natural-language statement of the property. REQUIRED.                           |
| ?formal | A formal counterpart of `text`, in a declared specification language              |

The natural-language `text` is deliberately mandatory: every reader of an assurance document can understand every claim, whatever tooling they have. The `formal` object, when present, has the fields:

| Fields   | Description                                                                        |
| ---      | ---                                                                                |
| language | A key of the document's `languages` registry                                       |
| ?source  | The formal statement itself, inline                                                |
| ?uri     | A URI from which the formal statement can be retrieved                             |

At least one of `source` or `uri` MUST be present.

#### Referencing validators

A validator reference is a string that resolves against the blueprint's `validators` list as follows:

1. If exactly one validator has an `id` field equal to the reference, it resolves to that validator.
2. Otherwise, if exactly one validator has a `title` equal to the reference, it resolves to that validator.
3. Otherwise (zero or several matches), the reference is unresolvable and consumers MUST treat it as an error.

To make references robust, this CIP RECOMMENDS that blueprint producers include in each validator entry an OPTIONAL field `id`: a string unique within the blueprint, stable across renames of the validator's `title`. The CIP-0057 meta-schema permits additional fields on validator objects, so blueprints carrying `id` fields remain valid CIP-0057 blueprints, and tools unaware of this CIP ignore them.

```json
{
  "validators": [
    {
      "id": "escrow-spend",
      "title": "escrow.spend",
      "redeemer": { "...": "..." },
      "compiledCode": "...",
      "hash": "7b3c9d1e5f2a8b4c6d0e9f317a5b8c2d4e6f0a1b3c5d7e9f2a4b6c8d"
    }
  ]
}
```

#### Evidence records

Each evidence record documents one verification act performed against the property:

| Fields      | Description                                                                                     |
| ---         | ---                                                                                             |
| method      | How the verification was performed; see [methods](#methods)                                     |
| verifier    | A free-form string identifying who performed the verification                                   |
| ?tool       | A key of the document's `tools` registry. REQUIRED for machine-checked methods.                 |
| outcome     | One of `verified`, `falsified`, `partial`, `inconclusive`; see [outcomes](#outcomes)            |
| date        | The date the verification was performed, as `YYYY-MM-DD`                                        |
| ?scriptHash | The blake2b-224 hash digest (56 hex characters) of the script the verification ran against. REQUIRED for machine-checked methods. |
| ?artifact   | A reference to the reproducible verification artifact. REQUIRED for machine-checked methods.    |
| ?notes      | Free-form remarks (e.g. number of test cases, proof effort)                                     |

For parameterized validators, `scriptHash` is the hash of the *unapplied* validator template — the same value as the blueprint's own `hash` field for that validator.

The `artifact` object has the fields:

| Fields     | Description                                                                    |
| ---        | ---                                                                            |
| uri        | A URI from which the artifact can be retrieved                                 |
| hash       | A content digest of the artifact, as a [digest object](#digest-objects)        |
| ?mediaType | An informative media type, e.g. `application/gzip`                             |

Everything needed to re-run the verification (environment, commands, expected results) lives *inside* the artifact and is tool-specific; this CIP does not standardize it.

#### Methods

The `method` field is an **open** enumeration. This CIP defines the meaning of five values:

| Method          | Description                                                              | Machine-checked |
| ---             | ---                                                                      | ---             |
| `formal-proof`  | A machine-checked mathematical proof of the property                     | yes             |
| `property-test` | Randomized/property-based testing against an executable statement       | yes             |
| `unit-test`     | A fixed suite of concrete test cases                                     | yes             |
| `audit`         | A structured review by an auditor (see [CIP-0052](../CIP-0052))          | no              |
| `manual-review` | An informal human review                                                 | no              |

Producers MAY use other method strings. Consumers MUST accept unknown methods and treat them as opaque: display them, but attach no semantics. Method values are case-sensitive: `Formal-Proof` is an unknown method, not a formal proof. The three *machine-checked* methods carry stricter requirements (`tool`, `scriptHash` and `artifact` are REQUIRED) because their whole point is reproducibility.

#### Outcomes

The `outcome` field is a **closed** enumeration:

| Outcome        | Description                                                                                  |
| ---            | ---                                                                                          |
| `verified`     | The method completed and supports the property (proof succeeded, tests passed, audit found no violation) |
| `falsified`    | A counterexample or violation was found                                                      |
| `partial`      | The property was established for a subset of the cases or configurations covered by the statement |
| `inconclusive` | Verification was attempted but reached no conclusion (e.g. timeout, tool limitation)         |

The `method` conveys the strength of an outcome: a `verified` property-test and a `verified` formal-proof are very different levels of assurance. Consumers SHOULD always present the method alongside the outcome and SHOULD NOT collapse evidence into a single unqualified "verified" badge.

#### Consumer obligations

A consumer of assurance documents:

1. MUST validate the document against the meta-schema referenced by its `$schema` and reject invalid documents, including documents whose `formal.language` or `evidence.tool` references do not resolve in the corresponding registry.
2. MUST, when `blueprint.hash` is present, compare it against the actual blueprint bytes, and treat the assurance document as not applicable on mismatch.
3. MUST, when `scriptHash` is present on an evidence record, compare it against the resolved validator's `hash` in the blueprint, and flag the evidence as **stale** on mismatch.
4. MUST verify an artifact's content digest before relying on the artifact's content.
5. MUST treat unresolvable or ambiguous validator references as errors.
