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

Building directly on the use cases CIP-0057 already lists, this proposal enables:

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

The meta-schema for assurance documents (i.e. the schema used for validating assurance documents themselves) is given as an annex: [schemas/assurance.json](./schemas/assurance.json). An assurance document identifies the format version it complies with through its `$schema` field.

A **producer** is any tool or person authoring assurance documents. A **consumer** is any tool interpreting them (e.g. a wallet, an explorer, a verification checker).

### Document structure

The document itself is a JSON object with the following fields. Here and in all field tables below, a leading `?` marks a field as optional; all other fields are required.

| Fields      | Description                                                        |
| ---         | ---                                                                |
| `$schema`   | URI identifying the assurance document format version              |
| preamble    | An object with meta-information about the assurance document       |
| blueprint   | A reference to the blueprint the claims are about                  |
| ?languages  | A registry of specification languages used by formal statements    |
| ?tools      | A registry of verification tools referenced by evidence records    |
| ?formalFragments | Reusable formal-language definitions that formal statements are written against |
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

The `hash`, when present, MUST be computed over the raw bytes of the blueprint document exactly as retrieved from `uri`. Including it is RECOMMENDED: it makes the binding tamper-evident. Consumers MUST verify it when present and MUST treat the assurance document as *not applicable* to a blueprint whose bytes do not match. When `hash` is absent, consumers SHOULD signal that the binding to the blueprint document is unverified.

#### digest objects

Everywhere a content digest appears, it is an object:

| Fields | Description                                                     |
| ---    | ---                                                             |
| alg    | The hash algorithm, e.g. `sha256`, `blake2b-256`                |
| digest | The lowercase, hex-encoded digest value                         |

#### languages and tools

Both fields are objects mapping a document-local identifier — a key matching `^[A-Za-z0-9_-]+$`, used by `formal.language` and `evidence.tool` references — to a registry entry:

| Fields       | Description                                            |
| ---          | ---                                                    |
| name         | The name of the language or tool                       |
| version      | A version number, in any format                        |
| ?uri         | A URI to the language specification or tool homepage   |
| ?description | An informative description                             |

This CIP deliberately does **not** restrict which specification languages or verification tools can be referenced; see the Rationale. A reference to a key absent from the corresponding registry makes the document invalid; consumers MUST reject such documents.

Registry entries are open objects, like every object in an assurance document. The four fields above identify a language or a tool, but they cannot on their own describe the *environment* a formal statement is checked in: a language embedded in a proof assistant, for instance, elaborates its statements against particular versions of particular libraries, and its own `version` pins none of them. A language or tool specification MAY therefore define additional fields on its registry entry — a list of library dependencies and their versions being the obvious case — and consumers that do not recognize them MUST ignore them, as they must for any unrecognized field. This CIP does not standardize such fields: what a checking environment consists of is exactly the language-specific question this format declines to answer.

#### properties

The essence of the assurance document: a non-empty list of claimed properties. Each property is an object:

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
| ?uses    | A list of [formal fragment](#formalfragments) ids the statement is written against  |

At least one of `source` or `uri` MUST be present.

A formal statement is a claim about the validators named in the enclosing property's `scope`, so the two have to be connected: nothing is gained by proving a theorem that never mentions the code. This CIP does not reserve an identifier for a validator — doing so would privilege one language's naming inside a format that deliberately privileges none — but it does require the connection to be specified somewhere. A `formal.language` specification MUST define how the validators named in `scope.validators` are denoted inside a formal statement written in that language, and SHOULD define the language's notion of a validator accepting or rejecting a transaction, since that is what most claims are about. A formal statement that does not refer to the scoped validators constrains nothing about the compiled code, however rigorous its proof; no consumer can detect that, which is why the obligation falls on the language.

#### formalFragments

Formal statements are frequently not self-contained. A property's `formal.source` may be stated in terms of definitions — a model of the contract's state, a helper predicate, a family of concrete scenarios — that several properties share, and that a producer extracting statements from source annotations has extracted from the same source. The OPTIONAL top-level `formalFragments` field carries those definitions once, as a list of objects:

| Fields   | Description                                                                     |
| ---      | ---                                                                             |
| id       | An identifier unique within the document, matching `^[A-Za-z0-9_.-]+$`          |
| language | A key of the document's `languages` registry                                    |
| ?imports | A list of ids of other fragments this fragment's `source` depends on            |
| source   | The fragment itself, inline                                                     |

A statement's `formal.uses` names the fragments it is written against. `imports` supplies the transitive closure, so `uses` need not spell out what the fragments it names already depend on.

The `id` pattern is deliberately wider than the property `id` pattern: fragment ids are typically the names of modules in the producer's source language, such as `My.Contract.Types`, so `.` is permitted.

`imports` is what makes the collection ordered rather than a bag: the order of the `formalFragments` array is not significant, and `imports` is. The edges MUST be acyclic, and both `imports` and `uses` MUST resolve within the document; see [Consumer obligations](#consumer-obligations).

**Fragments are opaque.** A fragment's `source` is text in the language named by its `language` field, exactly as `formal.source` is; this CIP attaches no meaning to it. A consumer that does not know that language MUST ignore the document's fragments rather than reject the document — an unrecognized language is the normal case, not an error. A consumer that does know the language, and that is asked to check a property, MUST make available to it every fragment reachable from that property's `uses` through `imports`.

**What `imports` does not express.** `imports` names fragments *within this document*. A fragment will usually also depend on things outside it — the standard library of the declared language, a library of ledger definitions — and this version of the format does not express those dependencies. What a fragment may assume to already be in scope is part of the declared language's own definition; [languages and tools](#languages-and-tools) is where a language can declare the environment it elaborates in.

#### Referencing validators

A validator reference is a string that resolves against the blueprint's `validators` list as follows:

1. If exactly one validator has an `id` field equal to the reference, it resolves to that validator.
2. Otherwise, if exactly one validator has a `title` equal to the reference, it resolves to that validator.
3. Otherwise (zero or several matches), the reference is unresolvable and consumers MUST treat it as an error.

To make references robust, this CIP RECOMMENDS that blueprint producers include in each validator entry an OPTIONAL field `id`: a string unique within the blueprint, stable across renames of the validator's `title`. The CIP-0057 meta-schema permits additional fields on validator objects, so blueprints carrying `id` fields remain valid CIP-0057 blueprints, and tools unaware of this CIP ignore them.

For a producer that generates assurance documents from annotations in the contract's own source, `id` is more than a recommendation in practice: the id is what binds an annotation to the validator it annotates, and a `title` is prose the author is free to rewrite. Such a producer emits `id` for every validator it describes, and its references never fall through to rule 2.

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

`scriptHash` is the on-chain script hash as defined by CIP-0057's validator `hash` field: a blake2b-224 digest of the serialised script, with language tag prefix. For parameterized validators, it is the hash of the *unapplied* validator template — the same value as the blueprint's own `hash` field for that validator.

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

1. MUST validate the document against the meta-schema referenced by its `$schema` and reject invalid documents.
2. MUST additionally reject documents that violate the constraints the meta-schema cannot express: `formal.language` or `evidence.tool` references that do not resolve in the corresponding registry; property `id`s that are not unique within the document; `formal.uses` or `formalFragments.imports` entries that do not resolve to a fragment of the same document; fragment `id`s that are not unique; and `imports` edges that contain a cycle. None of these checks requires knowing any formal language, so they apply to every consumer; the four concerning fragments are vacuous for a document that carries neither `formalFragments` nor `uses`, which is why they can be added without a new `$schema` URI.
3. MUST, when `blueprint.hash` is present, compare it against the actual blueprint bytes, and treat the assurance document as not applicable on mismatch.
4. MUST, when `scriptHash` is present on an evidence record, compare it against the resolved validator's `hash` in the blueprint, and flag the evidence as **stale** on mismatch. When the blueprint validator carries no `hash` (CIP-0057 makes it optional in the absence of `compiledCode`), consumers MUST treat the evidence as **unverifiable** against that validator — distinct from stale.
5. MUST verify an artifact's content digest before relying on the artifact's content; a consumer that cannot compute the declared `alg` MUST treat the artifact as unverified rather than skip the check.
6. MUST treat unresolvable or ambiguous validator references as errors.

#### Producers

Most of this specification constrains consumers, because a consumer is what a reader's trust passes through. Producers carry four obligations of their own, and they matter most for the case this format is expected to become ordinary: a document generated by the contract's own toolchain, from annotations in the contract's source, on every build.

A producer of assurance documents:

1. SHOULD produce byte-identical output for a given source tree, so that regenerating a document produces no spurious diff. A document that changes on every build is one no reviewer can read a diff of, and one no repository can usefully track.
2. SHOULD take `preamble.authors` and `preamble.created` from project metadata rather than from the invoking user and the wall clock, so that rebuilding an old commit reproduces that commit's document. `created` dates the document's content, not the file.
3. MUST compute `blueprint.hash` over the blueprint document as finally written. For a producer emitting both documents this constrains build order: write `plutus.json` first, hash the bytes as they now exist on disk, then write the assurance document. A digest taken over an intermediate representation of the blueprint is not a digest of the blueprint, and a consumer comparing it against the published file will find it does not match.
4. MUST NOT scope a property to a validator the property is not about. `scope` is what makes a claim checkable against particular compiled code, so a property attached to a validator it does not constrain is a false claim about that validator rather than an imprecise one — and no consumer can detect it. A producer holding a claim that fits no validator MUST omit the claim rather than attach it to an arbitrary one; see [Validators only](#validators-only).

A producer running at build time, before any verification has been attempted, is expected to emit properties with **no** `evidence` records. That is not an incomplete document: it is the "stated, unverified claim" case of the [Motivation](#motivation-why-is-this-cip-necessary), and a later verification run is what adds the evidence.

## Examples

The following complete examples are also available as machine-readable files under [examples](./examples).

<details>
  <summary>Formal verification with Blaster (formal-proof evidence, UAL formal statements, assumptions)</summary>

```json
{
  "$schema": "https://cips.cardano.org/cips/cipXXXX/schemas/assurance.json",
  "preamble": {
    "title": "Escrow contract — formal verification assurance",
    "description": "Machine-checked safety properties of the escrow validator, verified with Blaster.",
    "version": "1.0.0",
    "authors": [
      "Input Output — Blaster team <blaster@iohk.io>"
    ],
    "created": "2026-08-11",
    "license": "CC-BY-4.0"
  },
  "blueprint": {
    "uri": "https://raw.githubusercontent.com/example-org/escrow/v1.2.0/plutus.json",
    "hash": {
      "alg": "sha256",
      "digest": "3f8a1c2b9d4e5f60718293a4b5c6d7e8f9012a3b4c5d6e7f8091a2b3c4d5e6f7"
    }
  },
  "languages": {
    "ual": {
      "name": "Universal Annotation Language",
      "version": "0.4",
      "uri": "https://github.com/input-output-hk/ual-spec",
      "description": "Property specification language used by Blaster."
    }
  },
  "tools": {
    "blaster": {
      "name": "Blaster",
      "version": "0.3.1",
      "uri": "https://github.com/input-output-hk/blaster",
      "description": "SMT-based formal verification tool for UPLC validators."
    }
  },
  "properties": [
    {
      "id": "no-locked-funds",
      "title": "Funds can always be recovered",
      "scope": {
        "validators": ["escrow-spend"]
      },
      "statement": {
        "text": "For every reachable state of the escrow, either the buyer or the seller can construct a transaction that spends the locked UTxO.",
        "formal": {
          "language": "ual",
          "uri": "https://raw.githubusercontent.com/example-org/escrow/v1.2.0/verification/no-locked-funds.ual"
        }
      },
      "assumptions": [
        {
          "text": "The datum of the escrow UTxO conforms to the blueprint's datum schema."
        },
        {
          "text": "The transaction validity interval is finite."
        }
      ],
      "tags": ["safety", "liveness"],
      "evidence": [
        {
          "method": "formal-proof",
          "verifier": "Input Output — Blaster team",
          "tool": "blaster",
          "outcome": "verified",
          "date": "2026-08-01",
          "scriptHash": "7b3c9d1e5f2a8b4c6d0e9f317a5b8c2d4e6f0a1b3c5d7e9f2a4b6c8d",
          "artifact": {
            "uri": "https://github.com/example-org/escrow/releases/download/v1.2.0/escrow-proofs.tar.gz",
            "hash": {
              "alg": "sha256",
              "digest": "ab31c7e2f4d5968a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f708192a3b4c5d6"
            },
            "mediaType": "application/gzip"
          }
        }
      ]
    },
    {
      "id": "redeem-authorized",
      "title": "Only authorized parties can move funds",
      "scope": {
        "validators": ["escrow-spend"]
      },
      "statement": {
        "text": "Before the deadline, only a transaction signed by the buyer can spend the escrow UTxO; after the deadline, only a transaction signed by the seller can.",
        "formal": {
          "language": "ual",
          "uri": "https://raw.githubusercontent.com/example-org/escrow/v1.2.0/verification/redeem-authorized.ual"
        }
      },
      "assumptions": [
        {
          "text": "The datum of the escrow UTxO conforms to the blueprint's datum schema."
        }
      ],
      "tags": ["safety", "authorization"],
      "evidence": [
        {
          "method": "formal-proof",
          "verifier": "Input Output — Blaster team",
          "tool": "blaster",
          "outcome": "verified",
          "date": "2026-08-01",
          "scriptHash": "7b3c9d1e5f2a8b4c6d0e9f317a5b8c2d4e6f0a1b3c5d7e9f2a4b6c8d",
          "artifact": {
            "uri": "https://github.com/example-org/escrow/releases/download/v1.2.0/escrow-proofs.tar.gz",
            "hash": {
              "alg": "sha256",
              "digest": "ab31c7e2f4d5968a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f708192a3b4c5d6"
            },
            "mediaType": "application/gzip"
          }
        }
      ]
    }
  ]
}
```
</details>

<details>
  <summary>Property testing with Aiken (minimal document: no languages registry, natural-language statement only)</summary>

```json
{
  "$schema": "https://cips.cardano.org/cips/cipXXXX/schemas/assurance.json",
  "preamble": {
    "title": "hello_world — property-test assurance",
    "authors": [
      "Ada Lovelace <ada@example.com>"
    ],
    "created": "2026-08-11"
  },
  "blueprint": {
    "uri": "https://raw.githubusercontent.com/aiken-lang/aiken/v1.1.5/examples/hello_world/plutus.json"
  },
  "tools": {
    "aiken": {
      "name": "Aiken",
      "version": "1.1.5",
      "uri": "https://aiken-lang.org"
    }
  },
  "properties": [
    {
      "id": "requires-owner-signature",
      "title": "Spending requires the owner's signature and the magic message",
      "scope": {
        "validators": ["hello_world"]
      },
      "statement": {
        "text": "The validator succeeds only if the transaction is signed by the public key hash stored in the datum's 'owner' field and the redeemer's 'msg' field is exactly the UTF-8 bytes of 'Hello, World!'."
      },
      "evidence": [
        {
          "method": "property-test",
          "verifier": "Ada Lovelace",
          "tool": "aiken",
          "outcome": "verified",
          "date": "2026-07-15",
          "scriptHash": "5e1e8fa84f2b557ddc362329413caa3fd89a1be26bfd24be05ce0a02",
          "artifact": {
            "uri": "https://github.com/example-org/hello-world-tests/archive/refs/tags/v1.0.tar.gz",
            "hash": {
              "alg": "sha256",
              "digest": "d9e0f1a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f70819a2b3c4d5e6"
            }
          },
          "notes": "500 randomized scenarios generated with Aiken's property-testing framework; suite rejects missing signatures and wrong messages."
        }
      ]
    }
  ]
}
```
</details>

<details>
  <summary>Compiler-generated from source annotations (formalFragments, no evidence)</summary>

Emitted by a Plinth toolchain from annotations in the contract's own source, during the same build that writes `plutus.json`. Its single property carries no `evidence` record, and that is the point: the document is produced *before* any verification has been attempted, so every property in it is a stated, unverified claim — a specification target for a later run to fill in.

The definitions the formal statement is written against travel in `formalFragments`, keyed by the surface module they were extracted from, and the statement names the fragment through `formal.uses`. The `blueprint.uri` is relative because the two documents are written side by side at the project root.

```json
{
  "$schema": "https://cips.cardano.org/cips/cipXXXX/schemas/assurance.json",
  "preamble": {
    "title": "Ticket contract — UAL assurance",
    "description": "Generated from UAL annotations.",
    "version": "1.0.0",
    "authors": [
      "Example Author <a@example.com>"
    ],
    "created": "2026-08-24",
    "license": "CC-BY-4.0"
  },
  "blueprint": {
    "uri": "plutus.json",
    "hash": {
      "alg": "sha256",
      "digest": "6c1f0b7d3a94e582f0d6a1b2c3e4f50918273a4b5c6d7e8f90a1b2c3d4e5f607"
    }
  },
  "languages": {
    "ual": {
      "name": "Universal Annotation Language",
      "description": "Property specification language used by Blaster.",
      "version": "0.4",
      "uri": "https://github.com/input-output-hk/ual-spec"
    }
  },
  "formalFragments": [
    {
      "id": "Ual.Fixture",
      "language": "ual",
      "source": "def ticketOk (t : Ticket) : Prop := t.value > 0"
    }
  ],
  "properties": [
    {
      "id": "ticket_ok",
      "scope": {
        "validators": [
          "ticketSpend"
        ]
      },
      "statement": {
        "text": "A ticket with a positive value is accepted.",
        "formal": {
          "language": "ual",
          "uses": [
            "Ual.Fixture"
          ],
          "source": "∀ (t : Ticket), ticketOk t"
        }
      }
    }
  ]
}
```
</details>

## Rationale: how does this CIP achieve its goals?

### Why a detached document rather than a blueprint extension

An earlier draft of this proposal extended CIP-0057 blueprints in place, through an additional `$vocabulary` entry and new keywords inside `plutus.json`. The detached design was chosen instead, for three reasons:

1. **Third-party publishing.** The parties best placed to make assurance claims — auditors, independent verification teams — usually do not control the contract's repository. A detached document lets them publish claims about deployed code without any cooperation from the developers, and lets several independent assurance documents about the same blueprint coexist.
2. **Hand-maintained data does not survive a generated file.** `plutus.json` is regenerated on every build by the smart-contract framework (Aiken, OpShin, plu-ts, ...). Assurance data maintained by hand inside it would be overwritten on each compilation, or would require every framework to learn how to preserve and merge it. This argument does *not* apply to assurance content derived from source annotations, which a toolchain can regenerate as freely as the blueprint itself — see [Producers](#producers). It is the first and third reasons that make detachment right in both cases: a generated assurance document still has to be publishable by parties who cannot regenerate the blueprint, and still gains nothing from `$vocabulary` machinery.
3. **No dialect machinery needed.** An assurance document is not a JSON Schema dialect extension; it is its own document type. Identifying the format by a versioned `$schema` URI is simpler than the `$vocabulary` opt-in mechanism, and CIP-0057 remains entirely untouched.

Developers who want the "blueprint carries its claims" experience simply ship an `assurance.json` next to their `plutus.json`.

### Why the specification language is free

There is today no consensus — on Cardano or elsewhere — on a single property language for smart contracts, and mandating one would gate adoption of this CIP on the outcome of that debate. Instead, the format standardizes the *envelope*: what is claimed (in mandatory natural language), by whom, about which exact code, with what outcome, and where the evidence lives. Formal statements are optional and declare their language through the `languages` registry, so a Blaster user can reference Universal Annotation Language statements while an Aiken user references executable property tests, without either being privileged by the format. If the ecosystem later converges on a standard property language, it slots into the registry like any other.

The mandatory natural-language `text` guarantees that every claim remains legible to every reader — including the users whose funds are at stake — regardless of which formal languages and tools they know.

That freedom has a consequence the format absorbs rather than resolves. A `formal.language` implementation MAY require additional fields in the *blueprint* itself — an ordered list of the terms a program is applied to together with their encoding schemes, for instance, or an execution budget — because a formal statement about a compiled program has to say what that program is applied to, and CIP-0057 splits `parameters`, `datum` and `redeemer` without giving the application order. Such fields are legal CIP-0057 additions: validator objects do not forbid additional fields, the same argument this CIP makes for `id`. They are specified by the language that needs them, not by this CIP, and a blueprint carrying them remains a valid CIP-0057 blueprint that tools unaware of the language ignore.

### Why no signatures in v1

Evidence records name their verifier, but nothing in this version authenticates that name cryptographically. This is deliberate: the trust model of this CIP rests on *reproducibility* — anyone can fetch the artifact, check its digest, re-run the verification and compare verdicts — not on the authority of the verifier. Signing evidence records (e.g. with [CIP-0008](../CIP-0008) message signing or COSE) would add real value against lazy consumers who do not re-run verifications, but it drags key management and identity questions into the format. It can be layered on in a future version, or by wrapping assurance documents in an external attestation, without changing the format defined here.

### Open methods, closed outcomes

New verification methods keep appearing (symbolic execution, model checking, fuzzing variants, ...), so `method` is an open enumeration: unknown values are legal and treated as opaque by consumers. Outcomes, by contrast, are what consumers compare and display at a glance, so `outcome` is a small closed enumeration with fixed semantics. The strength of an outcome is conveyed by its method — which is why consumers are told to always present the two together.

### Relation to CIP-0052

[CIP-0052](../CIP-0052) defines best practices for conducting audits of Cardano smart contracts. This CIP is complementary: it gives an audit performed along CIP-0052 lines a standard, machine-readable, code-bound publication format — an evidence record with `method: audit` whose artifact is the audit report itself, hash-pinned and tied to the audited script hashes.

### Validators only

Assurance documents annotate validators — the unit of description in CIP-0057 blueprints. "Validator" is meant mechanically rather than by ledger role: a CIP-0057 `validators` entry is in substance a *named compiled program with argument schemas*, and nothing in this CIP requires that the ledger invoke it directly. A producer MAY therefore emit a `validators` entry for any named compiled program it can describe that way — a helper function compiled to a UPLC program of its own, for instance — and properties about that program are then scoped exactly like properties about a spending validator. That is what the format already describes; reading `validators` more narrowly would exclude claims it can carry perfectly well.

Finer-grained specifications *within* a validator — per-function contracts over definitions that never become a compiled program of their own — and richer blueprint descriptions remain deliberately out of scope, and are left to future work.

Making `scope` mandatory has a cost worth stating plainly. `scope.validators` is REQUIRED and non-empty, so every property in this version is a claim about at least one named program. A claim about a contract's *pure model* — an arithmetic lemma about a vesting schedule, say, which mentions no compiled code at all — has no home here, and a producer MUST NOT give it one by scoping it to a validator it is not about ([Producers](#producers)). Admitting such claims means either permitting an empty `scope.validators` or introducing a second kind of scope; both change the meaning of a REQUIRED field rather than add an OPTIONAL one, so both belong to a future version under a new `$schema` URI. Until then, a project with properties of that kind publishes the validator-scoped ones here and keeps the rest where they already live.

### Backward compatibility

This CIP requires no change to CIP-0057. The RECOMMENDED validator `id` field is already legal under the CIP-0057 meta-schema (validator objects do not forbid additional fields), and blueprints without `id` fields remain fully usable through `title`-based references. Tools unaware of this CIP are unaffected: assurance documents are separate files they never read.

## Path to Active

### Acceptance Criteria

- [ ] The meta-schema is published and has remained stable through community review.
- [ ] At least one producer toolchain emits assurance documents (Blaster, planned).
- [ ] At least one consumer tool validates assurance documents, including binding checks (blueprint hash, validator resolution, script hash comparison) and artifact digest verification.
- [ ] At least one assurance document is published for a real-world, deployed contract.

### Implementation Plan

- [ ] Blaster to emit an `assurance.json` (UAL formal statements, `formal-proof` evidence records, proof artifacts) for the validators it verifies.
- [ ] Develop a standalone checker that performs the consumer obligations of this specification: meta-schema validation, binding checks, and artifact digest verification.
- [ ] Engage explorer, registry and wallet developers on surfacing assurance claims to end users.

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
