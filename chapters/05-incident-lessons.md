# Lessons from Exchange Incidents

Exchange thefts repeat a few patterns. This chapter summarises public incidents, separating observation from attribution, with one engineering lesson each. Figures are approximate and come from public reports at the time; check the cited sources for the latest findings.

## A pattern table

| Incident (year) | Reported loss | Tier hit | Pattern |
|---|---|---|---|
| Mt. Gox (2014) | ~850,000 BTC reported missing at the time; ~200,000 later found | hot and operational wallets over years | long-running undetected leakage, weak reconciliation |
| Bitfinex (2016) | ~120,000 BTC | multisig hot wallets with a co-signing provider | co-signer limits bypassed once the exchange side was compromised |
| Coincheck (2018) | ~US$530M in NEM | hot wallet | large balance in a single-key hot wallet |
| DMM Bitcoin (2024) | ~4,500 BTC (~US$300M) | signing process via a wallet-software vendor | vendor employee social engineering, manipulated transaction request |
| WazirX (2024) | ~US$230M | Safe multisig with a custody-provider interface | approvers signed a transaction different from what they saw |
| Bybit (2025) | ~US$1.5B in ETH and staked ETH | Ethereum cold multisig (Safe) | compromised wallet-interface supply chain, delegatecall implementation swap |

## Mt. Gox: reconciliation is a security control

Reports after the 2014 collapse described coins leaking over a long period unnoticed. Continuous three-way reconciliation (chapter 4) turns slow theft into an early alert.

## Bitfinex: co-signers must hold their own policy

Bitfinex used a multisig arrangement with a co-signing provider. Public accounts of the 2016 theft indicate that the attacker, once inside the exchange's systems, was able to get the co-signer to approve the transfers, so the per-user limits did not stop the drain. A co-signer that applies limits configured by the party it is meant to check offers little independence. Policy for a second signer should be set and changed only through that signer's own out-of-band process.

## Coincheck: hot-wallet sizing

Coincheck reported in January 2018 that NEM tokens worth around US$530M at the time were taken from a hot wallet. Public reporting noted that the funds were held in a single-signature hot wallet rather than in cold or multisig storage. The lesson is chapter 1: the hot-wallet balance is your maximum loss from remote compromise, so it must be sized from withdrawal needs.

## DMM Bitcoin: vendors are inside the boundary

In May 2024 DMM Bitcoin reported the loss of about 4,500 BTC. In December 2024, the FBI, the US Department of Defense Cyber Crime Center and Japan's National Police Agency jointly attributed the theft to the DPRK-linked cluster tracked as TraderTraitor. Their statement described a fake recruiter approaching an employee of Ginco, a Japanese wallet-software company, with a malicious "pre-employment test" script, followed by access to Ginco's systems and manipulation of a legitimate DMM transaction request.

Lesson: vendor staff and systems are your attack surface. Ask vendors whether a staff compromise could alter transaction contents, and verify destinations on hardware you control.

## WazirX: what you see is not what you sign

In July 2024 WazirX lost roughly US$230M from an Ethereum Safe multisig that used a custody provider's interface for approvals. Public post-incident statements from both companies disagreed on root cause, but agreed that signers approved a transaction whose real effect (a contract upgrade giving the attacker control) differed from what the interface showed. Several analytics firms linked the theft to DPRK-associated actors.

Lesson: approvals must be verified on an independent device that decodes the actual calldata, and implementation or module changes must be blocked by policy outside scheduled ceremonies.

## Bybit: the interface supply chain

On 21 February 2025 Bybit reported that about US$1.5B in ETH and staked-ETH tokens was taken from an Ethereum cold wallet. Bybit and Safe published forensic findings (with Sygnia and Verichains) indicating that a Safe{Wallet} developer machine was compromised and that malicious JavaScript, served through Safe{Wallet}'s web infrastructure, targeted Bybit's wallet specifically. Signers saw what looked like a routine transfer; the transaction actually performed a `delegatecall` that replaced the wallet's implementation with attacker code, which then drained it. The FBI publicly attributed the theft to TraderTraitor (DPRK) within days.

Lessons:

- The signing UI is a supply-chain dependency. Pin and verify front-end assets, or run your own build.
- Hardware wallets must clearly decode Safe transactions; blind signing removes the independent check.
- A pre-signing policy checker that rejects `delegatecall` to unknown targets would have flagged the payload regardless of the UI.

See **dprk-crypto-threat-operations** for the actor's broader playbook and **web3-supply-chain-security** for front-end integrity.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Rebuild the Bybit-style failure mode safely on a local Anvil chain to test a pre-signing checker. Do not test against live wallets.

1. Start `anvil` and deploy a minimal proxy whose slot 0 holds an implementation address, plus a "wallet" implementation with an `execute(address to, bytes data, uint8 op)` that performs `call` when `op == 0` and `delegatecall` when `op == 1` (a simplified Safe).
2. Deploy an "upgrade" contract whose function writes a new address into slot 0 via `sstore(0, x)`.
3. Build the transaction a signer would approve: `execute(upgrade, abi.encodeWithSignature("sweep(address)", attacker), 1)`.
4. Write a checker that decodes the call before signing:

```python
from eth_abi import decode
def check(calldata: bytes, allow_delegate: set[str]) -> str:
    to, data, op = decode(["address", "bytes", "uint8"], calldata[4:])
    if op == 1 and to.lower() not in allow_delegate:
        return f"REJECT delegatecall to {to}"
    return "OK"
```

5. Confirm the checker rejects step 3 and accepts a plain `call` transfer. Then confirm with `cast storage` that executing step 3 on Anvil changes slot 0. The checker decides; the UI is not trusted.

## Common pitfalls

- Quoting loss figures without dates and sources.
- Assuming that a reputable custody vendor removes the need for your own verification.
- Studying only the hack, not the detection delay.
- Treating attribution as proven technical fact rather than as an assessment by a named agency.

## Exercises

1. For two incidents above, write a one-page timeline from primary sources and mark the earliest point where a control from chapters 2–4 could have stopped it.
2. Extend the Anvil checker to reject owner, threshold, module and guard changes.
3. Run a tabletop: the on-call gets a "cold wallet configuration changed" page at 03:00. Write the first 30 minutes of actions.

## Further reading

- FBI, DC3 and NPA joint statement on the DMM Bitcoin theft (December 2024)
- FBI public service announcement on the Bybit theft (February 2025): <https://www.ic3.gov>
- Bybit and Safe incident updates and the Sygnia/Verichains forensic reports (February–March 2025)
- WazirX and Liminal post-incident statements (July 2024)
- See **dprk-crypto-threat-operations** and **web3-supply-chain-security**
