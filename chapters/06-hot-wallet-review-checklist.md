# Hot Wallet Security Review Checklist

This chapter turns the book into a review you can run against a real custody stack, whether you are an internal security team, an auditor, or a vendor answering due-diligence questions. Each section lists the questions, the evidence to ask for, and a quick test. Run tests only on systems you are authorised to assess, using regtest, testnets or staging environments. Report findings through the organisation's disclosure or internal process.

## How to run the review

1. Collect the architecture: tiers, signers, vendors, networks, and every flow between wallets.
2. Interview owners: treasury, security, platform, and whoever runs signing ceremonies.
3. Walk one withdrawal end to end with logs, from user click to on-chain confirmation.
4. Run the tests below in a staging environment.
5. Rate each finding by the maximum value it could move and how many independent boundaries it bypasses.

```mermaid
flowchart LR
    A[Architecture] --> B[Interviews]
    B --> C[End-to-end trace]
    C --> D[Staging tests]
    D --> E[Findings rated by max loss]
```

## Tiering and sizing

| Question | Evidence | Test |
|---|---|---|
| How is the hot ceiling set per asset? | sizing model from outflow history | recompute the 99th-percentile outflow from data |
| What happens when hot runs dry? | refill runbook | trigger a staging refill and time it |
| Are internal transfer destinations fixed? | signer allowlist | attempt a refill to an unlisted address in staging |
| What share is in each tier today? | current balances | compare with policy |

## Withdrawal pipeline

- Is the balance hold atomic with withdrawal creation? Test with concurrent requests.
- Do security changes (password, 2FA, email) trigger a withdrawal lock? Change one in staging and withdraw.
- Is there a per-asset velocity limit independent of per-user limits? Replay a synthetic spike.
- Is the address whitelist integrity-protected and delayed? Try a direct database edit in staging and confirm the signer rejects it.
- Are fees and token decimals capped at the signer?

## Signing infrastructure

- Can any single credential cause a signature above the hot limit? List every credential that can reach a signer.
- Does the signer verify transactions against approved withdrawal IDs and reject replays?
- HSM: where is the PIN? Who can change policy? Is admin quorum enforced?
- MPC: which protocol and version, audited when, who runs the coordinator, how are shares backed up and refreshed?
- Multisig: do approvers verify decoded transactions on independent hardware? Are `delegatecall`, owner, module and guard changes blocked outside ceremonies?
- Are signer deployments built reproducibly and deployed through a pipeline separate from the main application?

## Vendors and supply chain

- Which vendor software runs on approver machines or in the signing path?
- Can a vendor employee alter transaction contents? Ask for their threat model and incident history.
- Are web interfaces used for signing pinned to verified builds or self-hosted?
- Is there a process to pause use of a vendor within minutes?

See **web3-supply-chain-security** for front-end and dependency integrity checks.

## People and process

- Are engineers with signing access trained on fake-recruiter and fake-investor lures, which feature in several attributed DPRK operations?
- Are signing ceremonies scripted, with a written expected payload compared against the decoded payload by two people?
- Are approvers' devices dedicated to signing, not used for email and browsing?
- Is there a duress or pause procedure any approver can trigger?

## Monitoring and response

- Is three-way reconciliation running, and how fast does it detect a 1% discrepancy?
- Are wallet configuration watchers reading storage, not just events?
- Do alerts trigger automatic asset pauses?
- Is there a tested incident runbook covering freezing deposits, contacting issuers of stablecoins and other freezable tokens, engaging analytics firms, and notifying law enforcement?
- Are monitors isolated from signing infrastructure?

## Rating findings

| Severity | Rule of thumb |
|---|---|
| Critical | one compromise moves cold or warm funds, or more than the hot limit |
| High | one compromise moves up to the hot limit, or disables detection |
| Medium | requires two independent compromises, or causes outage or loss of funds below the hot limit |
| Low | defence-in-depth gaps, missing documentation |

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Produce an evidence-backed finding from the regtest lab built in earlier chapters.

1. Configure the `hot` wallet so a script holding only the RPC password can call `sendtoaddress` to any destination. This mirrors a withdrawal service with direct signer access.
2. From that script, send 0.5 BTC to a fresh address in the `miner` wallet. It succeeds.
3. Record the finding: "The withdrawal service credential can sign arbitrary destinations and amounts up to the full hot balance. Max loss: entire hot wallet. Boundaries bypassed: one."
4. Implement the fix: move the hot key into a separate signer process that only accepts PSBTs and runs the allowlist and approval-ID checks from chapter 2's exercise.
5. Retest: the same script now produces a PSBT that the signer rejects with a logged reason. Attach both runs' logs to the finding as evidence.
6. If you use an LLM to draft the report, verify every claim in it against the logs before sending. The model drafts; the logs decide.

## Common pitfalls

- Reviewing architecture diagrams without tracing a real withdrawal.
- Rating findings by technical novelty instead of maximum loss.
- Skipping vendors because they are "out of scope".
- Testing in production without written authorisation.
- Accepting "we have an HSM" as an answer to "who can make it sign?".

## Exercises

1. Run this checklist against the regtest lab and write three findings with severity, evidence and fix.
2. Convert the checklist into a YAML file and a script that reports which items lack evidence links.
3. Hold a 45-minute tabletop based on one chapter 5 incident and record which checklist items would have caught it.

## Further reading

- CryptoCurrency Security Standard (CCSS), C4
- NIST SP 800-61, Computer Security Incident Handling Guide
- NIST SP 800-57 Part 1, Recommendation for Key Management
- See **mpc-wallet-security**, **dprk-crypto-threat-operations** and **web3-supply-chain-security**
