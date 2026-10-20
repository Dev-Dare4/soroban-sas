# @soroban-sas/sdk

TypeScript SDK for the Soroban Solidity Attestation Service (SAS). This is a
companion to the Rust SDK at `packages/soroban-sas-sdk` for JS/TS
applications (wallets, indexers, relayers, frontends) that need to compute
the same content-addressed UIDs and typed-data signatures the Rust contracts
and Rust SDK use, and to submit transactions against a deployed SAS /
Schema Registry contract.

## Install

```bash
npm install @soroban-sas/sdk
```

## Hashing (byte-identical with the Rust contracts)

`computeSchemaUid`, `computeAttestationUid`, and `hashTypedData` are ports of
`soroban_sas_common::schema_uid`, `soroban_sas_common::attestation_uid`, and
`soroban_sas_common::typed_data::hash_offchain_attestation` respectively.
Every field the Rust side hashes via `.to_xdr(env)` — a `String`, `Address`,
`Bytes`, or the `UID` newtype — is encoded here as the identical `ScVal` XDR
shape using `@stellar/stellar-sdk`; fields Rust hashes as raw bytes (`u64`
big-endian, `BytesN<32>` arrays) are encoded the same way here, never
XDR-wrapped. See `src/hashing.ts`'s doc comments for the exact preimage
layout of each function, and `test/hashing.test.ts` for golden vectors
copied verbatim from passing Rust tests in
`packages/soroban-sas-common/src/typed_data.rs` and
`packages/soroban-sas-common/src/test.rs`.

```ts
import { computeSchemaUid, computeAttestationUid, hashTypedData } from "@soroban-sas/sdk";

const schemaUid = computeSchemaUid("bool verified", resolverAddress, true);

const attestationUid = computeAttestationUid(
  schemaUid,
  recipientAddress,
  attesterAddress,
  new TextEncoder().encode("payload"),
);

const digest = hashTypedData(attestation, domain);
```

## Schema type generation

The SDK exports `parseSchema`, `generateTypeScript`, `encodeSchemaValue`, and
`decodeSchemaValue`. Generated interfaces preserve field order and include a
comment with each source schema type. Payload values encode as a Soroban
`ScVal` vector in schema field order, matching a Soroban struct's XDR shape.

```ts
import { encodeSchemaValue, generateTypeScript } from "@soroban-sas/sdk";

const schema = "name string, age uint32, verified bool, account address";
const source = generateTypeScript(schema, "Profile");
const xdr = encodeSchemaValue(schema, {
  name: "Ada",
  age: 37,
  verified: true,
  account: "G...",
});
```

Install the package to use the `sas-typegen` CLI:

```sh
sas-typegen "name string, age uint32, verified bool, account address" --name Profile
sas-typegen --schema-file schema.txt --name Profile --out profile.ts
```

Supported schema forms include primitives, `Option<T>` / `T?`, `Vec<T>`,
`[T]`, `Map<K:V>`, and `Tuple(T1 T2 ...)`. Numeric values wider than 32 bits
use JavaScript `bigint`; byte values use `Uint8Array`.

## Delegated attestations

`signDelegatedAttestation` signs `hashTypedData(attestation, domain)` with
the attester's ed25519 secret key (mirroring
`packages/soroban-sas-cli/src/offchain.rs::sign_offchain_attestation`);
`verifyDelegatedAttestation` recomputes the digest and checks the signature
without needing network access.

```ts
import { signDelegatedAttestation, verifyDelegatedAttestation } from "@soroban-sas/sdk";

const delegated = signDelegatedAttestation(attestation, domain, nonce, attesterSecret);
verifyDelegatedAttestation(delegated, domain); // true

// Any funded relayer account can now submit it without holding attesterSecret:
await client.attestByDelegation(delegated);
```

## Client

`SASClient` wraps `@stellar/stellar-sdk/contract`'s `Client`, which fetches
the deployed contract's on-chain XDR spec and uses it to build the
`InvokeHostFunction` operation for each call, rather than this package
hand-encoding argument layouts.

```ts
import { SASClient } from "@soroban-sas/sdk";

const client = new SASClient({
  contractId: "C...", // SAS contract id
  registryContractId: "C...", // Schema Registry contract id
  rpcUrl: "https://soroban-testnet.stellar.org",
  networkPassphrase: "Test SDF Network ; September 2015",
  secret: "S...", // pays for and authorizes submitted transactions
});

const schemaUid = await client.registerSchema("bool verified", resolverAddress, true);
const { hash } = await client.attest(attestation);
const stored = await client.getAttestation(attestation.uid);
await client.revoke(attestation.uid);
```

## Scope and limitations

- `hashing.ts` and `delegation.ts` are cross-checked byte-for-byte against
  Rust golden vectors (see `test/hashing.test.ts` and
  `test/delegation.test.ts`) and can be relied on for offline UID
  computation and delegated-attestation signing without a network call.
- `client.ts`'s contract-call argument encoding — in particular, the `UID`
  newtype's one-element-array representation — was derived from reading the
  Rust XDR codegen (see `packages/soroban-sas-sdk/src/events.rs`'s
  `decode_uid` comment) and has not been exercised against a live Soroban RPC
  node in this environment. Before relying on `attest`/`attestByDelegation`/
  `revoke`/`registerSchema` for a real submission, run one end-to-end call
  against a local or test network and adjust the argument shape if the
  simulated transaction reports a type mismatch.
