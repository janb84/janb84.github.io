---
title: "Bitcoin Block Validation — Architecture Diagrams"
subtitle: "Visual companion to the three-level validation pipeline"
date: 2026-06-10
categories: ["ARTICLE", "BITCOIN"]
tags: ["BITCOIN", "ARCHITECTURE", "CORE"]
readtime: "10 MIN READ"
---

Sedited wrote a nice post on Bitcoin block validation. This blog post is meant as a visual reading guide to go with it, with ASCII diagrams of the call graph, data structures, and caching layers to follow along as you read through [the blogpost](https://thecharlatan.ch/Validation/).

<!--more-->

<span class="continue" id="continue-reading"></span>

## 1. The Three-Level Validation Pipeline

```
  Network / RPC (Receive Block)
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ProcessNewBlock()                          │
│                   block_validation.h:45                         │
│                  acquires cs_main mutex                         │
└────────────────────────┬────────────────────────────────────────┘
                         │ calls directly
                         ▼
               ┌─────────────────────────────────────────────────┐
               │  LEVEL 1  — stateless, context-free             │
               │                                                 │
               │  CheckBlock()              block_validation.h:48│
               │    ├─ CheckBlockHeader()   (PoW / nBits)        │
               │    ├─ CheckMerkleRoot()                         │
               │    ├─ size limits          (inline)             │
               │    ├─ coinbase position    (inline)             │
               │    └─ CheckTransaction()   (per tx,             │
               │                            incl. CVE-2018-17144 │
               │                            duplicate inputs)    │
               └────────┬────────────────────────────────────────┘
                        │ FAIL → silently drop, return false
                        │        AcceptBlock() is never called
                        │
                        │ OK → calls next
                        ▼
               ┌─────────────────────────────────────────────────┐
               │  LEVEL 2  — contextual, touches block tree      │
               │                                                 │
               │  AcceptBlock()             block_validation.h:44│
               │    ├─ AcceptBlockHeader()                       │
               │    │    ├─ CheckBlockHeader()  (PoW again)      │
               │    │    ├─ prev block in BlockMap?              │
               │    │    └─ ContextualCheckBlockHeader()         │
               │    │         (PoW target, timestamps,           │
               │    │          soft-fork activation state)       │
               │    ├─ DoS guards (already have / not requested  │
               │    │             / too far ahead / min work)    │
               │    ├─ CheckBlock()          (cached, near-free) │
               │    ├─ ContextualCheckBlock()                    │
               │    │    (witness rules, BIP68 seq. locks, etc.) │
               │    ├─ ◀── RELAY TO PEERS (NewPoWValidBlock)     │
               │    │      happens here, before scripts checked! │
               │    ├─ WriteBlock() to disk                      │
               │    └─ ReceivedBlockTransactions()               │
               │                                                 │
               │  block stored; invalid verdict CAN be written   │
               └────────┬────────────────────────────────────────┘
                        │
                        │ ⚠ Note: a block that passed Level 1 & 2 but is
                        │   later found invalid (e.g. bad script in a tx)
                        │   is nevertheless already persisted on disk here.
                        │   To prevent an attacker filling the disk with
                        │   cheap junk, single blocks below a minimum
                        │   cumulative proof-of-work threshold are rejected
                        │   in the DoS guards above (MinimumChainWork).
                        │
                        │ OK → releases lock, then:
                        ▼
               ┌─────────────────────────────────────────────────┐
               │  LEVEL 3  — "Chain Level"                       │
               │                                                 │
               │  ActivateBestChain()       chainstate.h         │
               │    │  uses ─────────────────────────────────────┼──► CChain
               │    │         height-indexed array, genesis→tip  │    (active chain,
               │    │         O(1) lookup by block height        │     chain.h)
               │    │                                            │
               │    └─ ActivateBestChainStep()                   │
               │         └─ ConnectTip()                         │
               │              └─ ConnectBlock()  ◄── UTXO work   │
               │                   │  uses ──────────────────────┼──► CCoinsViewCache
               │                   │         outpoint → Coin     │    (in-memory UTXO,
               │                   │         txid:vout → value   │     coins.h)
               │                   │                             │
               │                   ├─ spend coins (UTXO set)     │
               │                   ├─ CheckInputScripts()        │
               │                   │    └─ CCheckQueueControl    │
               │                   │       (parallel, 15 threads)│
               │                   ├─ check subsidy / fees       │
               │                   └─ write CBlockUndo to disk   │
               │                                                 │
               │  most expensive stage                           │
               └─────────────────────────────────────────────────┘

  ⚠ Two main performance bottlenecks, both sitting here:
    1. Script & signature evaluation — cryptographically expensive;
       each input's signature must be verified (parallelised via
       CCheckQueueControl, but still CPU-bound).
    2. UTXO set interaction — the set is ~10 GB on disk; every
       input requires a lookup, and cache misses force disk I/O.
       CCoinsViewCache absorbs most hits (~80% of UTXOs spent
       within a day of creation), but the remainder is slow.

  Note: no function is named Chain*() here. The blog post calls
  this "chain level" because these functions operate on CChain
  (the active chain), unlike Level 1/2 which only touch the
  block tree and disk. ActivateBestChainStep also handles reorgs
  by calling DisconnectTip() → DisconnectBlock() when needed.
```

> **Design intent — why `ProcessNewBlock` calls `CheckBlock` itself before `AcceptBlock`:**
>
> If `CheckBlock` fails, the block is silently dropped and **never permanently marked invalid**.
> `AcceptBlock` is what writes the "this block is invalid" verdict to disk — skipping it means a
> peer is never banned for sending a block that fails `CheckBlock`.
>
> This guards against CVE-2012-2459 and similar: if an unknown form of block malleability causes
> a technically-valid block to fail `CheckBlock`, banning the sender would be a consensus-splitting
> bug. Keeping the failure silent and cheap avoids that risk.
> See: `block_validation.cpp:1120–1124`

---

## 2. Key Data Structures

```
┌──────────────────────────────────────────────────────────────────────┐
│                         ChainstateManager                            │
│                           chainstate.h                               │
│                                                                      │
│  ┌─────────────────────────┐   ┌───────────────────────────────────┐ │
│  │  BlockMap               │   │  Chainstate                       │ │
│  │  (block tree)           │   │                                   │ │
│  │                         │   │  ┌────────────────────────────┐   │ │
│  │  hash → CBlockIndex*    │   │  │  CChain  (active chain)    │   │ │
│  │                         │   │  │                            │   │ │
│  │  every known block      │   │  │  height → CBlockIndex*     │   │ │
│  │  incl. orphans          │   │  │  genesis ──────────► tip   │   │ │
│  └─────────────────────────┘   │  └────────────────────────────┘   │ │
│                                │                                    │ │
│                                │  ┌────────────────────────────┐   │ │
│                                │  │  CCoinsViewCache (UTXO)    │   │ │
│                                │  │                            │   │ │
│                                │  │  outpoint → Coin           │   │ │
│                                │  │  (txid:vout → value+height)│   │ │
│                                │  └────────────┬───────────────┘   │ │
│                                └───────────────│───────────────────┘ │
└──────────────────────────────────────────────── │ ───────────────────┘
                                                  │ cache miss
                                                  ▼
                                    ┌─────────────────────────┐
                                    │  CCoinsViewDB           │
                                    │  (LevelDB on disk)      │
                                    │  ~10 GB UTXO set        │
                                    └─────────────────────────┘
```

---

## 3. UTXO Coin View Hierarchy (Layered Cache)

```
  ConnectBlock()
       │ reads / spends coins
       ▼
┌─────────────────────────┐   in-memory hash map
│   CCoinsViewCache       │   ~80% of UTXOs spent within a day of
│   (in-memory)           │   creation → most lookups hit here
└──────────┬──────────────┘
           │ miss → slow path
           ▼
┌─────────────────────────┐   LevelDB, ~10 GB on disk
│   CCoinsViewDB          │   key-value: COutPoint → Coin
│   (disk)                │   serialised: VARINT(height<<1|coinbase)
└──────────┬──────────────┘             + compressed CTxOut
           │
           │  coins.h:36  struct Coin { CTxOut out; height; fCoinBase }
           │
           ▼
      (disk I/O — the bottleneck; cache exists to avoid this)
```

---

## 4. Script Validation — Parallel Path

```
 ConnectBlock()
      │
      │  for each transaction in block
      ▼
 ┌──────────────────────────────────────────────────────┐
 │  CheckInputScripts()      block_validation.h:90      │
 │                                                      │
 │  1. check ValidationCache (sig + script caches)      │
 │     hit? → skip expensive crypto                     │
 │     miss? → enqueue CScriptCheck                     │
 └──────────────────────┬───────────────────────────────┘
                        │  batch of CScriptCheck jobs
                        ▼
 ┌──────────────────────────────────────────────────────┐
 │         CCheckQueueControl  (checkqueue.h)           │
 │                                                      │
 │   worker 0 │ worker 1 │ worker 2 │ … │ worker N     │
 │   ─────────┼──────────┼──────────┼───┼──────────    │
 │   sig check│ sig check│ sig check│   │ sig check    │
 │                                                      │
 │   MAX_SCRIPTCHECK_THREADS = 15   (chainstate.h:78)   │
 │   only parallel step in block validation             │
 └──────────────────────────────────────────────────────┘
                        │ all workers done
                        ▼
               ConnectBlock() continues
```

---

## 5. Caching Layers — Where Time Is Saved

```
                    ┌──────────────────────────────┐
 Mempool tx enter ─▶│  ValidationCache             │
                    │                              │
                    │  ┌──────────────────────┐    │
                    │  │  SigCache            │    │
                    │  │  (sig verified once) │    │
                    │  └──────────────────────┘    │
                    │  ┌──────────────────────┐    │
                    │  │  ScriptCache         │    │
                    │  │  (full script pass)  │    │
                    │  └──────────────────────┘    │
                    └───────────────┬──────────────┘
                                    │  tx appears in block
                                    ▼
                    ┌───────────────────────────────┐
                    │  CheckInputScripts()          │
                    │                               │
                    │  cache HIT  → skip script     │
                    │  cache MISS → verify + store  │
                    └───────────────────────────────┘

 assumevalid:  for blocks BEFORE developer-set checkpoint
               ─────────────────────────────────────────
               CheckBlock()  ✓  (PoW + merkle still checked)
               AcceptBlock() ✓
               ConnectBlock() script checks SKIPPED entirely
               → ~10× faster IBD on resource-limited nodes
```

---

## 6. Block Relay Timing

```
 ─── time ───────────────────────────────────────────────────────────▶

  CheckBlock()          AcceptBlock()              ConnectBlock()
  (PoW, merkle)  ──OK──▶ (tree, softforks)  ──OK──▶ (UTXO, scripts)
                         │
                         │ block written to disk in network
                         │ serialisation format (fast GETDATA reply)
                         │
                         └──▶  RELAY TO PEERS  ◀─── happens here!
                                                     before scripts
                                                     are validated

 Why: miners need fast relay; full script validation is the slow part.
 Risk mitigated by: scripts validated right after on ConnectBlock().
```

---

## 7. IBD — Header-First Sync Strategy

```
 Phase 1: Headers only (80 bytes each, no block data)
 ──────────────────────────────────────────────────────
  Peer                     Node
   │──── headers (2000) ────▶│  ProcessNewBlockHeaders()
   │──── headers (2000) ────▶│  builds full block tree first
   │         …               │  block_validation.h:43
   │                         │
 Phase 2: Full blocks (download in parallel, validate sequentially)
 ──────────────────────────────────────────────────────────────────
   │──── block @ height N ──▶│
   │──── block @ height M ──▶│  out-of-order OK:
   │──── block @ height K ──▶│  stored as "out of order" until
   │         …               │  ancestor reaches active chain,
   │                         │  then full UTXO+script eval from disk

 Storage modes:
 ┌──────────────┬───────────────────────────────────────────────┐
 │  Full node   │  ~800 GB (since genesis) 900k blocks          │
 │  Pruned node │  ~18 GB  (UTXO set + configurable buffer)     │
 │              │  older blocks deleted after validation        │
 └──────────────┴───────────────────────────────────────────────┘
```

---

## 8. What Is Inside a Pruned Node (~18 GB)?

```
  A pruned node keeps only what it MUST have to validate new blocks.
  Everything else is deleted after validation.

  ┌─────────────────────────────────────────────────────────────────────┐
  │  FIXED — cannot be pruned                                           │
  │                                                                     │
  │  chainstate/  (LevelDB)                          ~10 GB             │
  │  └─ CCoinsViewDB — every unspent output (UTXO)                      │
  │       key: COutPoint (txid + vout index)                            │
  │       val: Coin (value, height, coinbase flag)                      │
  │       grows as the UTXO set grows; cannot be deleted                │
  │                                                                     │
  │  blocks/index/  (LevelDB)                        ~500 MB            │
  │  └─ BlockMap — index of ALL headers ever seen,                      │
  │       incl. orphans; never pruned because headers                   │
  │       are needed to validate the chain                              │
  └─────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────┐
  │  CONFIGURABLE — the pruning buffer                                  │
  │                                                                     │
  │  blocks/blk?????.dat  (raw block data)           configurable       │
  │  blocks/rev?????.dat  (undo data / CBlockUndo)   ~15% of blk size   │
  │                                                                     │
  │  • MAX_BLOCKFILE_SIZE  = 128 MB per .dat file                       │
  │  • MIN_BLOCKS_TO_KEEP  = 288 blocks (~2 days)                       │
  │    blocks within this window of the tip are NEVER pruned            │
  │    (needed to handle reorgs)                                        │
  │  • MIN_DISK_SPACE_FOR_BLOCK_FILES = 550 MiB (hard minimum)         │
  │                                                                     │
  │  User sets e.g. prune=8000 (MB); older block files deleted          │
  │  once the active window moves past them.                            │
  └─────────────────────────────────────────────────────────────────────┘

  Approximate total for a typical pruned node:

   ~10 GB   UTXO set (fixed)
  + ~0.5 GB block index (fixed)
  + ~7-8 GB block + undo buffer (user-configured, min 550 MB)
  ─────────────────────────────
   ≈ 18 GB

  A full archival node keeps ALL blk?????.dat files since genesis
  (~800 GB and growing), but the same fixed ~10.5 GB of UTXO +
  index overhead.
```

---

## 9. The cs_main Bottleneck

```
  cs_main protects:  block tree (BlockMap)
                     active chain (CChain)
                     UTXO set (CCoinsViewCache)
                     ValidationCache

  ┌──────────────────────────────────────────────────────────────┐
  │  LOCK 1 — held continuously (block_validation.cpp:1118)      │
  │                                                              │
  │   CheckBlock()  ──OK──▶  AcceptBlock()                       │
  │                                                              │
  │   lock released after AcceptBlock returns (line 1137)        │
  └──────────────────────────────────────────────────────────────┘
                          │
                          │  lock released; ActivateBestChain() called
                          ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  LOCK 2 — re-acquired internally by ActivateBestChain()      │
  │                                                              │
  │   ActivateBestChainStep() ──▶ ConnectTip() ──▶ ConnectBlock()│
  │                                                              │
  │   ONE block at a time — still a sequential bottleneck        │
  └──────────────────────────────────────────────────────────────┘

  Proposed optimisation (PR #35295):
  pre-warm coin cache in parallel BEFORE acquiring lock
  → overlaps disk I/O with prior block's script checks
```
