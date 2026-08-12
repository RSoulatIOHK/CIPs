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
