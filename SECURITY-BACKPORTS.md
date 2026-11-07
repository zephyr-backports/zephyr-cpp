# Security backports

Base: `ZephyrProtocol/zephyr-cpp` commit `4a99272` ("Updates for audit period", 2025-04-08; the state the
Zephyr v2.3.0 wallets were built from), itself on `woodser/monero-cpp` `686fbdc` (2023-05-25). Policy:
https://github.com/zephyr-backports/zephyr-backports (docs/POLICY.md). Each entry is one cherry-picked
monero-cpp commit; the commit in this repository carries `(cherry picked from commit <sha>)`.

| # | Tag | monero-cpp commit | Class | Component | Summary |
|---|---|---|---|---|---|
| SB-011 | v2.3.0-sec.1 | `6d6d203` | dos | monero_wallet_full | `scan_txs` takes the sync lock before scanning |
| SB-012 | v2.3.0-sec.1 | `83832b8` | memory | monero_wallet_model | tx query without transactions no longer reads `m_txs[0]` of an empty vector |
| SB-013 | v2.3.0-sec.1 | `48664eb` | memory | monero_wallet_model | output query no longer dereferences an unset subaddress index |
| SB-014 | v2.3.0-sec.1 | `44c81a8` | memory | monero_wallet_full | key image import checks that hex and signature are present |

## Advisories

<!-- One section per backport; template: docs/ADVISORY-TEMPLATE.md in zephyr-backports/zephyr-backports -->

### SB-011 — `scan_txs` takes the sync lock before scanning

- **Tag:** v2.3.0-sec.1
- **Origin:** woodser/monero-cpp `6d6d203f9df968c8dd652e4e667c1b05c0c52704` (2024-02-09, "get lock before scanning txs"), released in monero-cpp v0.8.7
- **Class:** dos (data race)
- **Component:** `src/wallet/monero_wallet_full.cpp`, `monero_wallet_full::scan_txs`
- **Affects:** callers of `scan_txs` (the JS `scanTxs` call through the WASM bridge) while the wallet is syncing. `scan_txs` called `wallet2::scan_tx` without holding `m_sync_mutex`, the mutex the sync loop and `lock_and_sync` hold, so a scan and a sync could modify wallet2's transfer state at the same time.
- **Impact:** concurrent modification of the wallet's transfer containers: corrupted in-memory state or a crash. The WASM build of zephyr-javascript is compiled without pthreads (`USE_PTHREADS` is commented out in its `CMakeLists.txt`), so there the two cannot run at the same moment; the race is real for native builds of the library. The fix is one line: take the lock.
- **Compatibility statement:** no consensus, relay, wire or file-format surface: monero-cpp is wallet-side C++ compiled into the zephyr-javascript WASM; it does not validate blocks, speak p2p or define RPC schemas, and the change touches neither the wallet cache nor the keys file format. POLICY §4 rules 1–5: not applicable beyond that.
- **Conflict resolution:** none
- **Test evidence:** done 2026-11-05 (local run; files in evidence/wasm/2026-11-05/): webwallet WASM build from zephyr 54aa43b12 + zephyr-cpp ac3bb4de3 + zephyr-javascript 9d022ded, guards 1–4 OK; harness smoke test (keys from seed, 40-block sync to tip, reserve info, serialisation, reopen) OK (`sec1-build-manifest.json`, `harness-results.md`); done 2026-11-05 (fork CI on GitHub; evidence/ci/2026-11-07/): provenance workflow run 42090724866 on security/v2.3.0 56abaf948: provenance OK (`fork-ci.md`); done 2026-11-07 (local run; files in evidence/wasm/2026-11-07/): the WASM built from the published commits (zephyr e9cf10f79 + zephyr-cpp 56abaf948 + zephyr-javascript 9d022ded) is byte-identical to the 2026-11-05 build when built at the same path; harness smoke test and cross-open both ways with upstream's prebuilt dist (9d022ded, keys `a541c83c`) OK (`harness-results.md`).
- **Notes:** none

### SB-012 — Tx query without transactions no longer reads past an empty vector

- **Tag:** v2.3.0-sec.1
- **Origin:** woodser/monero-cpp `83832b89f97cfadd36690a629566584d1336cf41` (2026-09-14, "wallet model: prevent deserialization crash (#168)"), released in monero-cpp v0.9.5
- **Class:** memory
- **Component:** `src/wallet/monero_wallet_model.cpp`, `monero_tx_query::deserialize_from_block`
- **Affects:** every tx query that reaches the library as JSON (the WASM bridge's `get_txs` deserialises its query this way). When the JSON block carried no transactions, the code took `block->m_txs[0]` of an empty vector.
- **Impact:** out-of-bounds read of a `shared_ptr` followed by a cast and member access: undefined behaviour, in practice a crash of the wallet module. The fix returns an empty query in that case.
- **Compatibility statement:** no consensus, relay, wire or file-format surface: monero-cpp is wallet-side C++ compiled into the zephyr-javascript WASM; it does not validate blocks, speak p2p or define RPC schemas, and the change touches neither the wallet cache nor the keys file format. POLICY §4 rules 1–5: not applicable beyond that.
- **Conflict resolution:** none
- **Test evidence:** done 2026-11-05 (local run; files in evidence/wasm/2026-11-05/): webwallet WASM build from zephyr 54aa43b12 + zephyr-cpp ac3bb4de3 + zephyr-javascript 9d022ded, guards 1–4 OK; harness smoke test (keys from seed, 40-block sync to tip, reserve info, serialisation, reopen) OK (`sec1-build-manifest.json`, `harness-results.md`); done 2026-11-05 (fork CI on GitHub; evidence/ci/2026-11-07/): provenance workflow run 42090724866 on security/v2.3.0 56abaf948: provenance OK (`fork-ci.md`); done 2026-11-07 (local run; files in evidence/wasm/2026-11-07/): the WASM built from the published commits (zephyr e9cf10f79 + zephyr-cpp 56abaf948 + zephyr-javascript 9d022ded) is byte-identical to the 2026-11-05 build when built at the same path; harness smoke test and cross-open both ways with upstream's prebuilt dist (9d022ded, keys `a541c83c`) OK (`harness-results.md`).
- **Notes:** none

### SB-013 — Output query no longer dereferences an unset subaddress index

- **Tag:** v2.3.0-sec.1
- **Origin:** woodser/monero-cpp `48664ebaf6bc687fec511fd892b3b591e1e472aa` (2026-08-26, "wallet model: guard unset subaddress index in output query (#152)"), released in monero-cpp v0.9.3
- **Class:** memory
- **Component:** `src/wallet/monero_wallet_model.cpp`, `monero_output_query::meets_criteria`
- **Affects:** output queries filtered on subaddress indices (`get_outputs` with `subaddressIndices`) when an output has no subaddress index set. The filter dereferenced the `boost::optional` without checking it.
- **Impact:** dereferencing an empty `boost::optional` is undefined behaviour (an assertion in debug builds, an indeterminate value or a crash otherwise), so the filter could crash or give a wrong answer. With the fix such an output simply does not match.
- **Compatibility statement:** no consensus, relay, wire or file-format surface: monero-cpp is wallet-side C++ compiled into the zephyr-javascript WASM; it does not validate blocks, speak p2p or define RPC schemas, and the change touches neither the wallet cache nor the keys file format. POLICY §4 rules 1–5: not applicable beyond that.
- **Conflict resolution:** none
- **Test evidence:** done 2026-11-05 (local run; files in evidence/wasm/2026-11-05/): webwallet WASM build from zephyr 54aa43b12 + zephyr-cpp ac3bb4de3 + zephyr-javascript 9d022ded, guards 1–4 OK; harness smoke test (keys from seed, 40-block sync to tip, reserve info, serialisation, reopen) OK (`sec1-build-manifest.json`, `harness-results.md`); done 2026-11-05 (fork CI on GitHub; evidence/ci/2026-11-07/): provenance workflow run 42090724866 on security/v2.3.0 56abaf948: provenance OK (`fork-ci.md`); done 2026-11-07 (local run; files in evidence/wasm/2026-11-07/): the WASM built from the published commits (zephyr e9cf10f79 + zephyr-cpp 56abaf948 + zephyr-javascript 9d022ded) is byte-identical to the 2026-11-05 build when built at the same path; harness smoke test and cross-open both ways with upstream's prebuilt dist (9d022ded, keys `a541c83c`) OK (`harness-results.md`).
- **Notes:** none

### SB-014 — Key image import checks that hex and signature are present

- **Tag:** v2.3.0-sec.1
- **Origin:** woodser/monero-cpp `44c81a80cde4a40f7d277dec23c63fd59e183dc8` (2026-08-24, "wallet full: add guard for key images hex (#148)"), released in monero-cpp v0.9.3
- **Class:** memory
- **Component:** `src/wallet/monero_wallet_full.cpp`, `monero_wallet_full::import_key_images`
- **Affects:** key image import from JSON (the WASM bridge's `import_key_images`; key images typically come from another device or wallet file). An entry without `hex` or without `signature` was read with `.get()` on an empty `boost::optional`.
- **Impact:** undefined behaviour on malformed import data, in practice a crash of the wallet module. The fix throws a clear error ("key image hex is not defined", "key image signature is not defined") before parsing.
- **Compatibility statement:** no consensus, relay, wire or file-format surface: monero-cpp is wallet-side C++ compiled into the zephyr-javascript WASM; it does not validate blocks, speak p2p or define RPC schemas, and the change touches neither the wallet cache nor the keys file format. POLICY §4 rules 1–5: not applicable beyond that.
- **Conflict resolution:** none
- **Test evidence:** done 2026-11-05 (local run; files in evidence/wasm/2026-11-05/): webwallet WASM build from zephyr 54aa43b12 + zephyr-cpp ac3bb4de3 + zephyr-javascript 9d022ded, guards 1–4 OK; harness smoke test (keys from seed, 40-block sync to tip, reserve info, serialisation, reopen) OK (`sec1-build-manifest.json`, `harness-results.md`); done 2026-11-05 (fork CI on GitHub; evidence/ci/2026-11-07/): provenance workflow run 42090724866 on security/v2.3.0 56abaf948: provenance OK (`fork-ci.md`); done 2026-11-07 (local run; files in evidence/wasm/2026-11-07/): the WASM built from the published commits (zephyr e9cf10f79 + zephyr-cpp 56abaf948 + zephyr-javascript 9d022ded) is byte-identical to the 2026-11-05 build when built at the same path; harness smoke test and cross-open both ways with upstream's prebuilt dist (9d022ded, keys `a541c83c`) OK (`harness-results.md`).
- **Notes:** none
