# Polkadot Identity Pallet Review Checklist

Use this checklist to review whether a learner understands the identity pallet
and can apply it safely in a Polkadot or Substrate application.

## Core Concepts

- Can you explain why identity fields require a deposit?
- Can you distinguish identity claims from registrar judgements?
- Can you name common identity fields and explain why large data is bounded?
- Can you explain why identity data is public once stored on chain?

## Judgement Flow

- How does a user find available registrars?
- What does `requestJudgement` require?
- What does the `max_fee` parameter protect against?
- Who calls `provideJudgement`?
- Which judgement states are warnings for applications?
- Why should a UI show the registrar behind a judgement?

## Sub-Identities

- What problem do sub-identities solve?
- Why does each sub-account require a deposit?
- What is the difference between `setSubs`, `addSub`, and `renameSub`?
- What should happen if a sub-account no longer wants to be linked?

## Client Integration

- Is the client connected to the correct People chain?
- Does the UI show reserved deposit costs before signing?
- Does the UI sanitize identity text before rendering it?
- Does the UI avoid treating identity as authorization?
- Does the app fetch registrar data from chain state?

## Runtime Integration

- Which origin can add registrars?
- Are maximum registrars, fields, and sub-accounts bounded?
- Are deposits high enough to discourage state-bloat attacks?
- Have pallet weights been benchmarked for production?
- Is chain metadata exposed for wallets and indexers?

## Extension Questions

1. How would you design a validator dashboard that groups stash accounts under
   one verified company identity?
2. How should a dapp display an account with `LowQuality` judgement?
3. What user warning would you show before publishing an email address on chain?
4. Why is a verified display name not enough to approve a treasury transfer?
5. How should an app behave if the identity pallet moves to another system
   chain or runtime location?
