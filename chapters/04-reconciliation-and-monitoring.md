# Reconciliation and Monitoring

Controls fail. Reconciliation and monitoring are how you find out quickly, while the loss is still small or still recoverable. This chapter covers balance reconciliation, on-chain watchers for wallet configuration, and alert design that operators will not learn to ignore.

## Three-way reconciliation

An exchange has three views of every asset:

1. **The internal ledger**: what users are owed plus the exchange's own funds.
2. **The wallet system**: what the custody software believes each wallet holds.
3. **The chain**: what full nodes you run (and ideally an independent provider) report.

| Comparison | A mismatch usually means |
|---|---|
| Ledger vs wallet system | credited a deposit that never arrived, missed a withdrawal debit, accounting bug |
| Wallet system vs chain | unknown outflow (theft), stuck or replaced transaction, reorg, node out of sync |
| Your node vs independent source | node eclipsed or lagging, wrong chain or fork |

Run this per asset, per wallet, on a schedule measured in minutes for hot wallets and hours for cold. The output is not just "match / no match" but a signed difference with a list of unexplained transactions. A difference that slowly grows is often a deposit-crediting bug; a sudden drop on the chain side with no matching withdrawal record is theft until proven otherwise.

Deposit crediting deserves its own checks. Credit only after a confirmation depth chosen per chain, re-check after reorgs, and treat unusual inputs carefully: tokens sent by an unexpected contract, internal transactions in EVM chains, or fake tokens with the same symbol. Several smaller exchanges have credited users for transfers of worthless look-alike tokens.

## Watching wallet configuration

For on-chain multisig and smart contract wallets, balances are not the only thing to watch. The configuration itself is an asset:

- owners and threshold;
- enabled modules and guards;
- the implementation (singleton) address for proxy wallets;
- allowances granted from the wallet to other contracts.

Do not rely only on events. The proxy used by Safe stores its implementation address in storage slot 0, and a `delegatecall` to attacker code can overwrite that slot without emitting any of the usual configuration events. Poll the storage directly and compare against a known-good value.

```bash
# expected implementation for your Safe version, recorded at deployment
EXPECTED=0x...   # placeholder
cast storage "$SAFE" 0 --rpc-url "$RPC"
cast call "$SAFE" "getOwners()(address[])" --rpc-url "$RPC"
cast call "$SAFE" "getThreshold()(uint256)" --rpc-url "$RPC"
```

Any change outside a scheduled ceremony is a page-the-on-call event. For Bitcoin, the equivalent is watching every UTXO of the cold descriptor: any spend not matching an approved ceremony is an incident.

## Alert design

Good alerts are few, specific and actionable.

| Signal | Threshold idea | Action |
|---|---|---|
| Hot outflow per asset | above rolling 99th percentile for the window | auto-pause asset withdrawals, page on-call |
| Cold or warm spend | any spend not linked to a ceremony ID | page security lead, freeze dependent flows |
| Wallet configuration change | any change | page, start incident |
| Reconciliation difference | above dust threshold and unexplained for N minutes | page treasury on-call |
| Signer error rate | spike in rejected policy checks | investigate, could be an attack being blocked |
| Node health | block height lag, peer count drop | fail over, suspect eclipse |

Two principles make this work. First, alerts must have an automatic first response, such as pausing withdrawals for an asset, because a human woken at 3 a.m. needs minutes. Second, the monitoring system must run on infrastructure separate from the signer and withdrawal pipeline, so an attacker who controls one does not silence the other.

## Proof of reserves

After the 2022 collapse of FTX, many exchanges began publishing proof-of-reserves reports. A typical design commits to user liabilities in a Merkle sum tree and shows control of addresses holding at least that much. Know the limits:

- It shows assets at a point in time, not liabilities outside the tree.
- Users must check their own inclusion for it to mean anything.
- Assets can be borrowed for the snapshot.

It is useful as an external reconciliation signal, not a replacement for internal controls.

## Where AI fits

Anomaly models can rank unusual flows for analysts. Keep them advisory: the model proposes a ranked list, while deterministic thresholds and the three-way reconciliation decide pages and pauses. Every flagged item should link to raw transactions so an analyst can confirm or dismiss it.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Build a minimal reconciler on Bitcoin regtest.

1. Using the wallets from chapter 1, keep a toy ledger as a JSON file:

```json
{"hot": 1.0, "warm": 5.0, "cold": 30.0}
```

2. Write a script that, for each wallet, calls `bitcoin-cli -regtest -rpcwallet=$w getbalances` and compares `mine.trusted` with the ledger:

```bash
for w in hot warm cold; do
  chain=$(bitcoin-cli -regtest -rpcwallet=$w getbalances | jq '.mine.trusted')
  ledger=$(jq ".$w" ledger.json)
  echo "$w ledger=$ledger chain=$chain diff=$(echo "$chain - $ledger" | bc)"
done
```

3. Simulate theft: send 0.3 BTC from `hot` to the `miner` wallet without updating the ledger, mine a block, and rerun. The difference shows up, roughly 0.3 plus fees.
4. Simulate a legitimate withdrawal: send and update the ledger, then rerun; the only difference is the fee. Teach the script to account for fees by reading `listtransactions` and matching each outgoing transaction to a ledger entry ID stored in the transaction `comment` field.
5. For EVM, start `anvil --fork-url` against a public RPC, read slot 0 and the owners of any public Safe, then use `anvil_setStorageAt` to change slot 0 on the fork and confirm your watcher alerts. This is a local fork; nothing touches the live chain.

## Common pitfalls

- Reconciling only ledger vs wallet software, never against an independent chain view.
- Monitoring Safe configuration via events alone and missing direct storage writes.
- Running monitors on the same hosts and credentials as the signers.
- Alerts that only notify, with no automatic pause.
- Crediting deposits from look-alike token contracts.

## Exercises

1. Extend the regtest reconciler to explain every difference with a transaction ID or flag it as unexplained.
2. Write a watcher that polls slot 0, owners and threshold of a Safe on an Anvil fork every block and exits non-zero on change.
3. Build a Merkle sum tree over 1,000 synthetic user balances and a verifier that checks one user's inclusion and that no node has a negative sum.

## Further reading

- Foundry `cast` reference: <https://book.getfoundry.sh/reference/cast/>
- Vitalik Buterin, "Having a safe CEX: proof of solvency and beyond" (2022): <https://vitalik.eth.limo/general/2022/11/19/proof_of_solvency.html>
- Bitcoin Core RPC `getbalances` and `listtransactions`
- See **onchain-malware-analysis** for tracing outflows after a theft
