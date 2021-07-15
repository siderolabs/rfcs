# Key Management

## Summary

This extends the ideas presented in RFC 001 with specific and opinionated
recommendations for key generation, trust bootstrapping, and management.

## Motivation

Left to their own devices, wide liberties can be taken with private key handling
which can lead to substantial and non-obvious risks.

For this reason, we wish to clearly define a strategy for best practices.

## Assumptions

* Any private key exposed to unreproducible binaries is compromised
* Any private key exposed to internet-connected memory is compromised
* Any private key that can be used without physical consent is compromised
* Any key pair without physical backups will become irrecoverable at any time
* Any public key not signed by peers or a well maintained CA is is an imposter
* Any shared hardware that is not used with constant supervision is compromised

## Requirements

* All private keys:
  * MUST be generated in one of the following ways:
    * Airgapped system (Advanced)
      * Secure booted with Heads or Safeboot
      * Deterministic OS image verified by at least one peer
      * Network devices physically disabled
      * External entropy source provided
        * Examples: Infinite Noise, analog sensors seeding /dev/random, etc
    * Inside PERSONAL HSM (Easy)
      * Yubikeys enable simple one-command generation but lack backup options
      * Ledger provides on-device generation and backup generation
  * MUST be stored on PERSONAL HSMs
  * SHOULD maintain Master key and Subkeys on separate PERSONAL HSMs
  * SHOULD have strictly scoped and distinct master key and subkeys:
    * Master key only has "Certify" permission
    * Subkeys for each of "Auth", "Sign", and "Encrypt"
  * SHOULD have a paper backup
    * BIP39 or raw GnuPG ASC exports possible
    * Consider safety deposit box, high quality safe, or Shamir's Secret Sharing
* All PERSONAL HSMs
  * MUST require physical interaction for each operation
  * MUST have a unique pin set for any access levels
    * Examples: Admin and User pins on Yubikey

## Scope

### Current

  * Key generation
  * Key distribution
  * Authentication
    * SSH
    * FIDO2
    * WebAuthn
    * U2F
    * TOTP (Google Authenticator)
  * Signing
    * Git Commits and Tags
    * Files
  * Encryption
    * Sensitive files
    * Email

### Future

  * Password management
  * Full disk decryption
  * System login
  * Measured Boot Attestation

## Design

The following is an opinionated set of defaults to consider that hit the MUST
requirements while being maximally low friction.

There are a number of alternative paths to hit the above MUST and SHOULD
requirements an advanced reader may consider, but they are out of scope for
this document. See the references section if you wish to pursue a highly
flexible setup that requires more upfront hardware and work.

### Requirements
  * 1+ Yubikey 5 series
    * Selected due to wide compatibility and PGP touch support
  * NFC/USB WebAuthn capable browser
    * MacOS, Linux
      * Chrome
      * Chromium
      * Firefox
    * Linux
      * Qutebrowser
    * Android
      * Android Browser
      * Chrome
      * Chromium
  * PGP smartcard capable email client
    * Android
      * Fair Email
      * K-9 Mail
    * MacOS, Linux
      * Thunderbird
  * Yubico Authenticator
    * Android, MacOS, or Linux device
  * Yubikey Manager CLI
    * MacOS or Linux device
  * GnuPG 2.0+
    * MacOS or Linux device

### Generation

1. Insert Yubikey into a workstation
2. Generate keychain

    Example:

    ```
    $ gpg2 --card-edit
    Application ID ...: D2760001240102000006037030240000
    Version ..........: 2.0
    Manufacturer .....: Yubico
    Serial number ....: 03703024
    Name of cardholder: Some Person
    Language prefs ...: en
    Sex ..............: male
    URL of public key : [not set]
    Login data .......: sperson
    Signature PIN ....: forced
    Key attributes ...: 4096R 4096R 4096R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

    gpg/card> admin
    Admin commands are allowed

    gpg/card> generate
    Make an off-card backup of encryption key? (Y/n) n
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 2y
    Key expires at Wed Dec 14 15:55:04 2018 PST
    Is this correct? (y/N) y

    GnuPG needs to construct a user ID to identify your key.

    Real name: Some Person
    Email address: person@company.com
    Comment:
    You selected this USER-ID:
        "Some Person <person@company.com>"

    Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
    (Note that this step will take a little while ~1 minute)
    gpg: key ABAEE282 marked as ultimately trusted
    public and secret key created and signed.

    gpg: checking the trustdb
    gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
    gpg: depth: 0  valid:   4  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 4u
    gpg: next trustdb check due at 2018-08-19
    pub   4096R/ABAEE282 2016-02-11 [expires: 2018-12-14]
          Key fingerprint = 5D97 0241 09AA 2A20 EC1F  AAEF 2223 BDB8 ABAE E282
    uid       [ultimate] Some Person <person@company.com>
    sub   4096R/C63542DE 2016-02-11 [expires: 2018-12-14]
    sub   4096R/051D07BD 2016-02-11 [expires: 2018-12-14]

    gpg/card> quit
    ```

3. Set personal details

    This is optional but can really help to disambiguate multiple keys or
    perhaps get a lost key in an office returned to you.

    Example:
    ```
    $ gpg2 --card-edit
    Application ID ...: D2760001240102000006037030240000
    Version ..........: 2.0
    Manufacturer .....: Yubico
    Serial number ....: 03703024
    Name of cardholder: [not set]
    Language prefs ...: [not set]
    Sex ..............: unspecified
    URL of public key : [not set]
    Login data .......: [not set]
    Signature PIN ....: forced
    Key attributes ...: 4096R 4096R 4096R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

    gpg/card> admin
    Admin commands are allowed

    gpg/card> name
    Cardholder's surname: Person
    Cardholder's given name: Some

    gpg/card> sex
    Sex ((M)ale, (F)emale or space): M

    gpg/card> login
    Login data (account name): sperson

    gpg/card> lang
    Language preferences: en

    gpg/card> quit
    ```

4. Set random user/admin pins

    ```
    $ gpg2 --card-edit
    Application ID ...: D2760001240102000006037030240000
    Version ..........: 2.0
    Manufacturer .....: Yubico
    Serial number ....: 03703024
    Name of cardholder: [not set]
    Language prefs ...: [not set]
    Sex ..............: unspecified
    URL of public key : [not set]
    Login data .......: [not set]
    Signature PIN ....: forced
    Key attributes ...: 4096R 4096R 4096R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

    gpg/card> admin
    Admin commands are allowed

    gpg/card> passwd
    gpg: OpenPGP card no. D2760001240102000006037030240000 detected

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? 1
    (You will have to type the old PIN (123456) and enter a new pin.
    PIN changed.

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? 3
    (You will have to type the old Admin PIN (12345678) and enter a new admin pin.
    PIN changed.

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? q
    ```

5. Enforce mandatory physical user interaction for all key operations

    ```
    ykman openpgp set-touch sig fixed
    ykman openpgp set-touch aut fixed
    ykman openpgp set-touch enc fixed
    ```

6. Distribute key to public key servers

This is optional but recommended. Keys generally will propagate to other
servers if you get it on one, but you can speed up the process by
publishing to a few major ones:

    ```
    gpg --keyserver hkps://keys.openpgp.org --send-key YOUR_KEY_ID
    gpg --keyserver hkps.pool.sks-keyservers.net  --send-key YOUR_KEY_ID
    gpg --keyserver pgpkeys.urown.net --send-key YOUR_KEY_ID
    gpg --keyserver keyserver.ubuntu.com --send-key YOUR_KEY_ID
    ```

Note: If you lose your public key, you can -not- recover it from a Yubikey.

### Configuration

1. Configure shell to use yubikey for ssh

    Add the following to your shell configuration file:

    ```
    unset SSH_AGENT_PID
    if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
      export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
    fi
    ```

    Be sure to source this file or open a new shell, so this applies!

2. Configure VCS to use new ssh/gpg keys

    Get ssh public key:
    ```
    gpg --export-ssh-key YOUR_KEY_ID
    ```

    Get PGP public key:
    ```
    gpg --export -a YOUR_KEY_ID
    ```

    Configure these in your Github/Gitea/Gitlab settings as your sole keys in
    the respective ssh and pgp sections.

3. Configure Yubikey as only 2FA

   Navigate to your security settings in your VCS web interface:

   * Enroll/Re-enroll TOTP using Yubico Authenticator per UI instructions
     * Enable "touch" in options in Yubico Authenticator
   * Setup Yubikey for U2F/FIDO2/WebAuthn 2FA as well
   * Ensure all other 2FA methods such as SMS are disabled

4. Verify ssh is only offering your Yubikey key

    ```
    ssh-add -L
    ```

5. Verify you can ssh to your git provider

    ```
    ssh git@github.com
    ```

    Note: Don't forget to tap your key when it flashes!

6. Configure git to always sign by default

   Adjust ~/.gitconfig similar to contain the following:
    ```
    [user]
        signingKey = YOUR_KEY_ID_HERE
    [commit]
        gpgSign = true
    [merge]
        gpgSign = true
    [gpg]
        program = gpg2
    ```

7. Publish key to maintainers repo

    The goal is to get a signed commit that shows as "Verified"

    ```
    git clone git@github.com:talos-systems/keys.git
    cd keys
    gpg --export YOUR_KEY_ID > pgp/jdoe.pgp
    gpg --export-ssh-key YOUR_KEY_ID > ssh/jdoe.pub
    git checkout -b add-keys-jdoe
    git commit -m "Add keys"
    ```

8. Have the existing maintainer merge your key

    ```
    git merge --no-ff add-keys-jdoe master
    ```

    Note: Maintainers, ensure you confirm the key is legitimate out of band!


## Drawbacks

* Bulk key operations like ssh, file decryption, etc. will require many taps
  * If you need this, pursue an advanced setup with an air-gapped laptop
* Windows support is poor
* iOS support does not exist
  * Android is well supported if mobile use cases are required

## Alternatives

List any alternatives others have tried, including the current solution if
there is one, to help others quickly catch up on how we got here.

## Questions

* How do we want to approach multi-sig?
  * distrust-foundation/Sig and hashbang/git-signatures approaches are options
  * Should likely address after the team has comfort with the scope of this doc
* How do we want to approach release engineering?
  * Should likely solve deterministic builds first
  * Will need to decide on a single widely known release key
    * Who holds it?
    * Who should hold the revocation certificate as insurance against abuse?

## Glossary

### PERSONAL HSM

Small form factor HSM capable of doing common required cryptographic operations
such as GnuPG smartcard emulation, TOTP, challenge/response, etc.

### MAINTAINER

Someone with write access to the primary source of truth branch for one or more
projects.

## References

* [#! Key generation guide](https://github.com/hashbang/book/blob/master/content/docs/security/SSH.md)
* [gpg.wtf guide](https://gpg.wtf)
