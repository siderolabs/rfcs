# Software Supply Chain Integrity

## Overview

This process is recommended for any binaries, that if compromised, could result
in a loss of funds exceeding $1,000,000 which is the rough level where
incidents of violent coercion or deployments of expensive ($200k+ per Zerodium)
"zero-day" exploits have been seen in the financial technology sector.

This could be an operating system image, firmware for a high value
cryptographic signing system, medical equipment control software, flight
navigation systemes, etc.

Notably this document seeks to largely mitigate all classes of supply chain
attack we have seen deployed in the wild with high accountability build and
release processes.

## Case Studies

* [CI/CD system compromised by covert and sophisticated rootkit](https://igor-blue.github.io/2021/03/24/apt1.html)
* [Trader tortured for $1,000,000 in bitcoin](https://www.ccn.com/dutch-bitcoin-trader-suffers-brutal-torture-with-a-heavy-drill-in-violent-robbery/)
* [NPM supply chain attack on CoPay wallets](https://medium.com/@hkparker/analysis-of-a-supply-chain-attack-2bd8fa8286ac)
* [0day in Firefox used to attack Coinbase](https://www.zdnet.com/article/firefox-zero-day-was-used-in-attack-against-coinbase-employees-not-its-users/)
* [Ruby Gem Supply chain attack to facilitate cryptoasset theft via clipboards](https://www.bleepingcomputer.com/news/security/malicious-rubygems-packages-used-in-cryptocurrency-supply-chain-attack/)
* [Web analytics supply chain attack deployed on Gate.io exchange](https://www.welivesecurity.com/2018/11/06/supply-chain-attack-cryptocurrency-exchange-gate-io/)

## Threat Model

* Any one human or computer involved in the supply chain is compromised
* All systems managed by a single party (IT, third parties) are compromised
* Any code or binaries controlled by one system or party are compromised
* All memory of all internet connected computers is visible to adversary
* Any logging or backups that are not signed are tampered with
* Adversary wields a "zero-click" "Zero-Day" exploit for any system

## Requirements

The following only applies if code is bound for production, and these can be
met in any order or cadence desired.

* Third party code:
  * MUST have extensive and frequent review.
    * Example: The Linux kernel has well funded distributed review.
  * MUST be hash-pinned at known reviewed versions
  * MUST be at version with all known related security patches
  * SHOULD be latest versions if security disclosures lag behind releases
    * Example: The Linux kernel
* First party code:
  * MUST be signed in version control systems by well known author keys
  * MUST be signed by separate subject matter expert after security review
* All code MUST build deterministically
* All binaries:
  * MUST be built and signed by multiple parties with no management overlay
    * Example: One build by IT, another by Infrastructure team managed CI/CD
  * MUST be signed by well known keys signed by a common CA
    * Example: PGP Smartcards signed under OpenPGP-CA.
* All signing keys:
  * MUST be stored in approved PERSONAL HSMs
  * MUST NOT ever come in contact with network accessible memory
* All execution environments SHOULD be able to verify m-of-n binary signatures
  * Example: Custom Secure Boot verifies minimum signatures against CA
* Only code required for sensitive operation SHOULD be present
  * Example: A server has no need for sound card drivers

## Implementation

Many implementations are possible under the above requirements and assumptions
however the following opinionated workflow can be used as a reference to
consider.

### Developer Workflow

1. Software Engineer makes source code change
    * Changes that bump third party dependencies should make them easy to review
      * Link to Changelog indicating this is the latest appropriate release
      * Links to evidence of third party review
      * Link to source of truth for hashes or keys as appropriate
      * Hash of commit or archive that is verified at build time
      * Public key hashes of any keys that can be used to verify sources
    * Automated verification of hash and signature in build logic
2. Software Engineer builds and tests changes locally
    * Local build should put output hashes of generated binaries for committing
3. Software Engineer makes commit signed with PERSONAL HSM and pushes

### Continuous Integration Workflow

1. Pulls code
2. Verifies commit signature by key signed with hardcoded CA key
3. Builds binaries
4. Verifies binary hashes match hashes in signed commit
5. Runs test suite
6. Signs binary with well known key in attached HSM
7. Publishes binary and signature to artifact storage
8. Continuous Integration notifies project team of success/error of above steps

### Security Reviewer Workflow

1. Reviews all code changes between release candidate tag and last commit
2. Reviews all third party changes and evidence they are appropriate
3. Appends tag signed with PERSONAL HSM to release candidate commit reviewed

### Release Engineer Workflow

1. Release engineer verifies:
    * Commit signature and security reviewer signature on commit are valid
    * The artifact corresponding to this commit is signed by the CI key
    * The artifact hash is in the release candidate commit
2. Release engineer generates detached artifact signature with PERSONAL HSM
3. Release engineer publishes detached signature to artifact store

### Continuous Deployment Workflow

1. CD daemon for each environment monitors for new signed artifacts
    * Development environments can use artifacts that are signed by any key
    * Staging environments can use artifacts with any key, but with a RC tag
    * Production environments will use artifacts signed both a RE and CI key.
2. CD daemon triggers rotation to new artifacts
3. New systems SHOULD boot to low level bootloader image where possible
    * Should be anchored with TPM as root of trust
      * Example: Heads, safeboot, etc.
    * This image should rarely ever change.
4. Bootloader image will:
    * Pull latest OS artifact and detached signatures
    * Verify detached signatures by two REs or one RE and one from CI
    * Use kexec to pivot to new multi-sig verified image
5. OS image will:
    * Pull down any new container artifacts and detached signatures
    * Verify signatures by two REs or one from an RE and one from CI
    * Execute desired container artifact

## Glossary

### MUST, MUST NOT, SHOULD, SHOULD NOT, MAY

These key words correspond to their IETF definitions per RFC2119

### PERSONAL HSM

Small form factor HSM capable of doing common required cryptographic operations
such as GnuPG smartcard emulation, TOTP, challenge/response, etc.

The following devices are recommended for each respective use case.

#### WebAuthN / U2F
  * Yubikey 5+
  * u2f-zero
  * Nitrokey
  * OnlyKey
  * ChromeOS Fingerprint ID
  * Password Management
  * Trezor Model T
  * Leger Nano X
  * Mooltipass

#### Encryption/Decryption/SSH
  * Yubikey 5+
  * Trezor Model T
  * Leger Nano X

#### Firmware Attestation
  * Librem Key
  * Nitrokey

## Disclaimers

Nothing in this document is proprietary in nature and subsets or variations of
these tactics are in use at several organizations.
