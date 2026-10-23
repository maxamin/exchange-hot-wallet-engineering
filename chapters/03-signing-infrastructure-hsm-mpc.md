# Signing Infrastructure: HSMs, MPC and Multisig

The signer is the last line of defence: by the time a request reaches it, upstream checks may already be compromised. This chapter compares three signing architectures and what each protects against.

## Three architectures

| Property | HSM | MPC / threshold signing | On-chain multisig |
|---|---|---|---|
| Key exists in one place | yes, inside the HSM | no, shares on several nodes | no, separate full keys |
| Visible on-chain | no, looks like a single-key wallet | no, looks like a single-key wallet | yes, script or contract is visible |
| Chain support | any curve the HSM supports | any chain the protocol supports | depends on chain features |
| Policy enforcement | firmware/app inside HSM, or wrapper service | policy engine across parties | the contract or script itself |
| Main failure mode | whoever can call the HSM API can sign | compromise of a quorum of nodes or the coordinator | approvers signing the wrong payload |
| Key rotation | new key, move funds | refresh shares without moving funds | change owners (contract) or move funds (script) |

None of these is "the secure one"; they move risk around. HSMs stop key extraction, not misuse. MPC removes the single key but adds protocol complexity. Multisig is transparent, but on EVM chains it is a contract with upgrade and module surfaces, and its human approval step was the target of the largest recent thefts.

## HSMs

An HSM generates, stores and uses keys inside a tamper-resistant boundary, reached via PKCS#11, a vendor SDK or a cloud KMS API.

- **Curve support.** Bitcoin and Ethereum use secp256k1. Not every HSM or cloud KMS supports it; check before designing around one.
- **Authorisation is the real boundary.** If the withdrawal service holds the HSM PIN or KMS credentials, compromising the service is equivalent to holding the key. Put a policy layer in front that the service cannot bypass, ideally running inside the HSM (many vendors support custom firmware or "secure execution" apps) or on a separately administered host.
- **Quorum administration.** Use M-of-N smart-card or operator quorums for administrative actions (key export, policy change, firmware update).
- **Normalisation.** ECDSA signatures for Bitcoin and Ethereum must use low-S form. HSMs often return arbitrary S; the wrapper must normalise.

## MPC and threshold signatures

In threshold ECDSA or Schnorr, several parties each hold a share and jointly produce a signature without reconstructing the key. The security assumptions to check:

- Which protocol (for example GG18, GG20, CGGMP21 family, or FROST for Schnorr) and which implementation and version.
- Whether the implementation has been audited after the publicly disclosed threshold-ECDSA attacks of 2021–2023 (see **mpc-wallet-security** for the details and references).
- Who runs the coordinator, and whether the coordinator can choose what is signed.
- How backups of shares work. A backup scheme that reconstructs the full key in one place undoes the point of MPC.
- How policy is enforced. A share holder that signs whatever the coordinator asks offers redundancy, not security.

## Multisig

Multisig is a script on Bitcoin and usually a Safe contract on EVM chains. Multisig shines for cold and warm tiers because each signer can verify independently on their own hardware device. It fails when signers do not actually verify. Approving a hash shown on a laptop screen, or "blind signing" an EVM transaction whose calldata the device cannot decode, means the approver's device adds no independent check.

For EVM multisigs specifically, watch for:

- `delegatecall` operations, which let the signed transaction execute arbitrary code in the wallet's context, including replacing the implementation (the mechanism reported in the 2025 Bybit theft; see chapter 5).
- Module and guard changes, which can grant a single address the power to move funds.
- Owner or threshold changes.

A policy that rejects any `delegatecall` except to a short allowlist, and any owner/module/guard change outside a scheduled ceremony, closes much of this surface.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Create a 2-of-3 multisig on Bitcoin regtest using Bitcoin Core descriptor wallets and PSBTs, so each signer works independently.

1. Create three signer wallets and collect an extended public key descriptor from each:

```bash
for s in s1 s2 s3; do bitcoin-cli -regtest createwallet $s; done
bitcoin-cli -regtest -rpcwallet=s1 listdescriptors | jq -r '.descriptors[] | select(.desc|startswith("wpkh(")) | select(.internal==false) | .desc'
```

2. Extract the `[fingerprint/path]xpub` part from each and build `wsh(sortedmulti(2,XPUB1/0/*,XPUB2/0/*,XPUB3/0/*))` (and an internal `/1/*` version for change). Add checksums with `getdescriptorinfo`, then import them into a watch-only wallet created with `createwallet ms true true "" false true`. Bitcoin Core's `doc/multisig-tutorial.md` walks through the exact commands.
3. Fund the multisig address from the miner wallet and mine a block.
4. Create a PSBT from the watch-only wallet with `walletcreatefundedpsbt`, then have `s1` and `s2` each run `walletprocesspsbt` on their own copy.
5. Combine with `combinepsbt`, inspect with `analyzepsbt` and `decodepsbt`, then `finalizepsbt` and `sendrawtransaction`.
6. Now act as a careful approver: before step 4's signing, each signer decodes the PSBT and checks outputs against the approved withdrawal. Modify one output amount in a copy and confirm that the decoded view shows the change. That is the independent check a signer must actually perform.

Optional: install SoftHSM2 and use `pkcs11-tool --keypairgen --key-type EC:secp256k1` to see how an HSM-held key is created and referenced by label, and how the application sees only a handle, never the key.

## Common pitfalls

- HSM credentials on the same host as the withdrawal service.
- MPC deployments where the coordinator alone decides what to sign.
- Share backups that reassemble the full key on one machine.
- Multisig approvers who sign hashes or blind payloads instead of decoded transactions.
- Allowing `delegatecall` and module changes through the normal approval flow.
- Not normalising ECDSA S values when wrapping HSMs.

## Exercises

1. Complete the 2-of-3 regtest multisig and write a script that rejects a PSBT if any output is not in an allowlist or if the fee exceeds a cap.
2. Using SoftHSM2, generate a secp256k1 key and sign a 32-byte hash; verify the signature with OpenSSL or Python `ecdsa` and normalise S.
3. For an EVM Safe on a local Anvil chain, write a pre-signing checker that decodes `execTransaction` parameters and rejects `operation == 1` (delegatecall) unless the target is allowlisted.

## Further reading

- Bitcoin Core multisig tutorial: <https://github.com/bitcoin/bitcoin/blob/master/doc/multisig-tutorial.md>
- OASIS PKCS #11 specification; SoftHSM: <https://github.com/softhsm/SoftHSMv2>
- Canetti, Gennaro, Goldfeder, Makriyannis, Peled, "UC Non-Interactive, Proactive, Threshold ECDSA with Identifiable Aborts" (CCS 2020)
- Safe smart account contracts: <https://github.com/safe-global/safe-smart-account>
- See **mpc-wallet-security** and **account-abstraction-security**
