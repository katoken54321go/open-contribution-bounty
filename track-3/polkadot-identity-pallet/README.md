# Using Polkadot's Identity Pallet

Submission for [OpenGuild bounty #31](https://github.com/openguild-labs/open-contribution-bounty/issues/31).

Polkadot identity is an on-chain naming and attestation system. It lets an
account publish structured public information, connect sub-accounts to a primary
identity, and request judgement from registrars who can attest to the quality of
that information.

This guide explains how the identity pallet works, how a user interacts with it
through the Polkadot UI or a basic script, and what runtime builders should
consider when integrating the pallet into a Substrate-based chain.

## Short Version

The identity flow has four main actors:

1. an account that stores public identity fields on chain.
2. the identity pallet, which reserves deposits and manages identity state.
3. registrars, who can provide judgements after reviewing the account.
4. applications, wallets, explorers, and UIs that display identity information.

The most common path is:

1. the user calls `identity.setIdentity` with fields such as display name,
   website, email, Matrix, or other custom data.
2. the chain reserves the required deposit for storing those fields.
3. the user optionally calls `identity.requestJudgement` for a registrar.
4. the registrar reviews the information and calls `identity.provideJudgement`.
5. wallets and dapps can show the identity and judgement beside the account.

On Polkadot and Kusama, identity data is managed on the People system chains.
For developers this matters because identity calls must be sent to the correct
chain context, not blindly to the relay chain.

## Why Identity Exists

Raw account addresses are difficult for humans to verify. The identity pallet
adds a structured, chain-native way to say:

- this account claims to be a person, team, validator, treasury, or application.
- these are the public fields the account wants to expose.
- one or more registrars have reviewed those claims.
- these sub-accounts belong under the same primary identity.

The pallet does not turn an account into a trusted account by itself. It gives
the network a common data model for public claims and registrar attestations.
Wallets and users still need to decide which registrar judgements they trust.

## Core Data Model

### Identity Fields

An identity can include standard fields such as:

- display name.
- legal name.
- website.
- email.
- Matrix or other chat handle.
- X / Twitter handle.
- additional custom fields.

In the pallet, field values are represented by bounded data. Short values can be
stored directly, while larger values are represented by a hash. This keeps state
bounded and prevents arbitrary large data from being stored on chain.

### Deposits

Identity fields consume chain state, so the pallet reserves a deposit. The bond
is not spent during normal use. It is locked while the identity exists and is
returned when the user clears the identity.

Sub-identities also require deposits. This makes the hierarchy useful without
making it free to create unbounded state.

### Registrars

A registrar is an account authorized to judge identity information. Registrars
can configure:

- the fee they charge for judgement.
- the fields they care about.
- the judgement they provide after review.

Users select a registrar and provide a maximum fee they are willing to pay. A
registrar whose fee is below that maximum can review the identity and submit a
judgement.

### Judgement Levels

The pallet supports several judgement states:

| Judgement | Meaning |
| --- | --- |
| `Unknown` | No judgement has been made. |
| `FeePaid` | The user paid or reserved the judgement fee, but no judgement has been provided yet. |
| `Reasonable` | The information appears reasonable, but without deep verification. |
| `KnownGood` | The registrar can attest strongly that the information is correct. |
| `OutOfDate` | The information was once valid but is no longer current. |
| `LowQuality` | The information is imprecise or too low quality to be useful. |
| `Erroneous` | The information is wrong and may indicate malicious intent. |

Some judgements are sticky. For example, `FeePaid` and `Erroneous` cannot be
removed just by changing identity fields. They require a full identity removal
or registrar action, depending on the case.

## User Flow In The Polkadot UI

The UI flow is useful for learners because it maps directly to pallet calls.

### Set An Identity

1. Switch to the People chain for the network you are using.
2. Open the account identity flow in the UI or use the Developer Extrinsics tab.
3. Select `identity.setIdentity`.
4. Fill the fields you want to publish.
5. Sign and submit the transaction.

The identity becomes part of public chain state. Users should avoid publishing
private information or fields that they do not want indexed by third-party
services.

### Request Judgement

1. Check available registrars with `identity.registrars`.
2. Pick the registrar index and review the registrar's instructions.
3. Call `identity.requestJudgement`.
4. Set `reg_index` to the registrar index.
5. Set `max_fee` to the maximum fee you are willing to pay.
6. Follow any off-chain verification steps requested by the registrar.

After review, the registrar calls `identity.provideJudgement` and the judgement
is displayed alongside the account.

### Clear An Identity

If the user no longer wants the identity stored, the account can call
`identity.clearIdentity`. Clearing returns the reserved deposit, but it also
removes the public identity data and associated sub-account relationships.

## Sub-Identities

Sub-identities let one primary account group related accounts under a shared
identity. This is useful for:

- validators with multiple stash or controller accounts.
- teams operating treasury, operations, and deployment accounts.
- applications that want one canonical organization identity with role-specific
  accounts underneath it.

The parent account can call:

- `identity.setSubs` to replace the full sub-account set.
- `identity.addSub` to add one sub-account.
- `identity.removeSub` to remove one sub-account.
- `identity.renameSub` to change the displayed sub-account name.

A sub-account can also call `identity.quitSub` to leave the hierarchy.

Sub-identities should be used for relationship clarity, not for security by
assumption. A wallet may display the parent relationship, but users should still
verify transaction intent and account permissions.

## Basic Script Example

The following TypeScript-style example shows the shape of a simple client flow
with `@polkadot/api`. Exact endpoint names and types can vary by network and API
version, so treat it as a starting point rather than copy-paste production code.
Replace the endpoint with the People chain RPC provider you use in production.

```ts
import { ApiPromise, WsProvider } from "@polkadot/api";

const endpoint = "wss://api-people-polkadot.n.dwellir.com/YOUR_API_KEY";
const provider = new WsProvider(endpoint);
const api = await ApiPromise.create({ provider });

const identityInfo = {
  display: { Raw: "OpenGuild Builder" },
  legal: { None: null },
  web: { Raw: "https://openguild.wtf" },
  riot: { None: null },
  email: { Raw: "builder@example.com" },
  pgpFingerprint: null,
  image: { None: null },
  twitter: { Raw: "@openguildwtf" },
};

const tx = api.tx.identity.setIdentity(identityInfo);

// Sign with the account that owns the identity.
// await tx.signAndSend(account, ({ status }) => {
//   console.log(status.toString());
// });
```

For judgement, the client would build:

```ts
const registrarIndex = 3;
const maxFee = 500_000_000_000n;

const tx = api.tx.identity.requestJudgement(registrarIndex, maxFee);
```

Before submitting a judgement request, a production client should read
`api.query.identity.registrars()` and show the registrar metadata, fee, and
instructions to the user.

## Runtime Integration Notes

For a custom Substrate chain, adding the identity pallet is a runtime design
choice. At a high level, runtime builders need to:

1. include `pallet_identity` in the runtime dependencies.
2. configure pallet constants such as basic deposit, field deposit, sub-account
   deposit, maximum registrars, maximum additional fields, and maximum subs.
3. choose the origin that can add registrars.
4. benchmark and include weights for production use.
5. expose the pallet metadata so wallets and apps can build calls.

The important design tradeoff is state growth. If deposits are too low, identity
storage can become cheap spam. If registrar limits are too loose, governance may
end up with many low-quality registrars. If the UI hides deposit costs, users may
not understand why their balance is reserved.

## Application Integration Pattern

Applications that display identity data should:

- fetch the raw identity fields from chain state.
- show the exact account that owns the identity.
- show judgement state and registrar context.
- distinguish parent identities from sub-identities.
- avoid treating identity text as fully trusted HTML.
- give users a way to inspect the source account.

An explorer or dapp can use identity as a trust hint, but not as sole
authorization. For example, a treasury frontend may show a verified display name,
but it should still validate call data, origin, and on-chain permissions.

## Security And UX Pitfalls

### Identity Is Public

Identity fields are stored on chain. Users should assume the data can be copied,
indexed, and kept by third parties.

### Registrars Are Trust Anchors

A judgement is only as trustworthy as the registrar. Applications should avoid a
single generic "verified" badge unless they also show which registrar issued the
judgement and what level was provided.

### Homograph And Impersonation Risk

Display names and websites can be misleading. A malicious account may use a name
or URL that looks similar to a legitimate organization. Users should verify
accounts through multiple channels before sending funds or granting authority.

### Sub-Accounts Are Relationships, Not Permissions

Sub-identities show that one account is listed under another identity. They do
not automatically mean the parent account controls the sub-account in every
possible context, and they do not replace proxy, multisig, or governance checks.

### Chain Context Matters

Because Polkadot and Kusama identity moved to People system chains, clients must
connect to the correct network and route calls to the correct chain. A stale
client that assumes identity lives on the relay chain can show outdated data or
fail to build transactions.

## Practical Checklist For Builders

When building an identity feature, ask:

- Which chain stores the identity data?
- Does the user understand the deposit that will be reserved?
- Are identity fields sanitized before display?
- Does the UI show registrar judgement levels clearly?
- Does the UI explain sticky judgements and how to clear identity data?
- Can users inspect sub-account relationships?
- Does the dapp still verify authorization independently of identity labels?
- Are registrar lists and fees read from chain state rather than hard-coded?

## References

- [pallet_identity crate documentation](https://docs.rs/pallet-identity/latest/pallet_identity/)
- [Polkadot Wiki: Account Identity](https://wiki.polkadot.com/learn/learn-identity/)
- [Polkadot Wiki: Polkadot-JS Identity Guides](https://wiki.polkadot.com/docs/learn-guides-identity)
- [Polkadot Developer Docs: People Chain](https://docs.polkadot.com/reference/polkadot-hub/people-and-identity/)
- [Polkadot Support: set and clear an identity](https://support.polkadot.network/support/solutions/articles/65000181981-how-to-set-and-clear-an-identity)
- [Polkadot Support: request and cancel identity judgement](https://support.polkadot.network/support/solutions/articles/65000181990-how-to-request-and-cancel-identity-judgement)
