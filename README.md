# Abacross operations journal

Every production deploy of abacross.com since 2026-09-20, as a hash-chained, Merkle-blocked, externally anchored record.
This directory is published so that anyone can check it, not so that anyone has to trust it.

Three commands, from nothing to a verdict:

```
git clone https://github.com/abacross/agent-journal && cargo build --release --manifest-path agent-journal/Cargo.toml
git clone https://github.com/abacross/ops-journal
agent-journal/target/release/ajv verify ops-journal --require rfc3161,opentimestamps && echo "every record, block and anchor holds"
```

The same check runs in a browser at [abacross.com/journal](https://abacross.com/journal/), and [ninety seconds on why](https://abacross.com/why/) it is kept this way.

## What is here

| path | what |
| --- | --- |
| `records.jsonl` | one JSON record per line; each carries its SHA-256 (`hash`) and its predecessor's (`prev`) |
| `blocks/NNNNNN.json` | one header per block: the Merkle root over that block's record hashes, and `prev_root` linking to the block before |
| `anchors/NNNNNN.json` | receipts for that block's root: `kind`, the authority, when |
| `anchors/tokens/*.tsr` | the RFC 3161 timestamp tokens, signed by the authority |
| `anchors/proofs/*.ots` | the OpenTimestamps proofs, which commit the root to the Bitcoin blockchain |

## How to check it

You need the verifier, `ajv`, from [agent-journal](https://github.com/abacross/agent-journal) (Rust, `cargo build --release`), and optionally `openssl` and the `ots` client to check the anchors against their authorities yourself.

```
ajv verify . --require rfc3161,opentimestamps
```

That recomputes every record hash, every chain link, every block root and every `prev_root` from scratch, and reports each block's receipts.
Exit 0 means all of it holds and every block carries both kinds of receipt; 1 means the journal is internally inconsistent; 3 means it is consistent but some block is not anchored as required.

To check a receipt against its authority rather than taking the file's word for it:

```
# RFC 3161: the token's signed timestamp and the digest it covers
openssl ts -reply -in anchors/tokens/<root>.tsr -text

# OpenTimestamps: which Bitcoin block the root is committed to (needs a confirmed proof)
ots verify anchors/proofs/<root>.root.ots
```

## What this proves, and what it does not

A verified journal proves that each record existed in exactly this form no later than the time its block was anchored, and that nothing has been inserted, removed or edited since.
Two unrelated authorities are used so that distrusting one does not dismiss the record.

It does not prove that a record was true when it was written.
A journal is an honest diary, not a witness; what anchoring removes is the possibility that the diary was rewritten afterwards.

A fresh OpenTimestamps proof is a pending commitment until the Bitcoin transaction confirms, usually within a day; the weekly maintenance run upgrades them and marks the receipt `confirmed`.
