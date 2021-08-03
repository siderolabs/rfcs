# PGP and Git Key Management

## Summary

This extends the ideas presented in RFC 001 with specific and opinionated recommendations for key generation, git commit signing, and git merging workflow.
Other related topics will be discussed in future RFCs.

## Motivation

Left to their own devices, wide liberties can be taken with private key handling which can lead to substantial and non-obvious risks: private keys could be stolen, git commit authors can be impersonated, etc.

For this reason, we wish to clearly define a strategy for best practices.

## Assumptions

1. Any private key exposed to unreproducible binaries is compromised.
2. Any private key exposed to internet-connected memory is compromised.
3. Any private key that can be used without physical consent is compromised.
4. Any key pair will become irrecoverable at any time.

## Requirements

1. All private keys must be generated and securely stored on YubiKeys.
  * Our assumptions 1 and 2 make it almost impossible to generate a private key on a normal computer.
    For example, even if a network-isolated virtual machine is used for that, the host most likely was connected to the internet at one point; therefore host OS and the hypervisor may already be compromised.
  * Even if a totally secure air-gapped computer was used for private key generation, the fact that the private key is known to the user means that we should maintain _and enforce_ the strict endpoint security.
    Enforcing it typically implies some kind of "spy" software installed on computers used for work.
    I don't think we want that. Using a YubiKey for key generation means that we have it without knowing it; the endpoint security can rely on trust.
  * Due to assumption 3, signing should require a physical button tap, in addition to entering a password.
  * Due to assumption 4, everyone should have an empty spare YubiKey to avoid long job disruptions.
2. All public keys must be known to several parties (GitHub, PGP key servers, colleagues).
  * For example, suppose one's GitHub account is compromised.
    In that case, an attacker can replace the PGP key in account settings with their own and make a signed commit. `ci-bot` should check that not only commit is verified by GitHub, but also that signature can be verified with one of the keys set during bot's build process.
    That would require an attacked to compromise both GitHub account and build system.
3. All git commits must be signed by our keys.
  * That prevents the trivial impersonation of the commit's author and committer.

## Scope

* Key generation and basic distribution.
* Git commit signing.
* GitHub flow.

## Design

### Required hardware and software

* YubiKey 5 with firmware [5.2.3+](https://support.yubico.com/hc/en-us/articles/360016649139-YubiKey-5-2-3-Enhancements-to-OpenPGP-3-4-Support).
  Tested with firmware 5.2.7.
* Linux or macOS.
* GnuPG v2. Tested with version 2.3.1.
  * macOS: `brew install gnupg`.
* PIN entry program:
  * macOS: `brew install pinentry-mac`.
  * Linux: TODO.
* Git.
* No Yubico software like YubiKey Manager, `ykman`, etc is required.

### Key generation and basic distribution

1. Insert YubiKey into a computer.
2. Perform OpenPGP application reset:

```
$ gpg --card-edit

Reader ...........: Yubico YubiKey OTP FIDO CCID
Application ID ...: D2760001240100000006154577040000
Application type .: OpenPGP
Version ..........: 0.0
Manufacturer .....: Yubico
Serial number ....: 15457704
Name of cardholder: [not set]
Language prefs ...: [not set]
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
UIF setting ......: Sign=off Decrypt=off Auth=off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

$ gpg/card> admin
Admin commands are allowed

gpg/card> factory-reset
gpg: OpenPGP card no. D2760001240100000006154577040000 detected

gpg: Note: This command destroys all keys stored on the card!

Continue? (y/N) y
Really do a factory reset? (enter "yes") yes
```

3. Set PIN and Admin PIN:

```
gpg/card> passwd
gpg: OpenPGP card no. D2760001240100000006154577040000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection?
```

If `passwd` command output is different, `admin` mode should be enabled as described in the previous point.

PINs are actually passwords or passphrases – up to 127 7-bit ASCII symbols.
Don't use only numbers.

Set PIN via option 1 and Admin PIN via option 3.
Default PIN: `123456`.
Default Admin PIN: `12345678`.

> Despite some guides claiming that PUK should be set too, [it has nothing to do with YubiKey's OpenPGP application](https://github.com/drduh/YubiKey-Guide/issues/271).
> Reset code also should be set as [it is not useful for us](https://forum.yubico.com/viewtopicd01c.html?p=9055#p9055].

4. Enable Curve 25519 key generation as it is [the most secure option](https://xkcd.com/285/):

```
gpg/card> key-attr
Changing card key attribute for: Signature key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 2
Please select which elliptic curve you want:
   (1) Curve 25519 *default*
   (4) NIST P-384
   (6) Brainpool P-256
Your selection? 1
The card will now be re-configured to generate a key of type: ed25519
Note: There is no guarantee that the card supports the requested
      key type or size.  If the key generation does not succeed,
      please check the documentation of your card to see which
      key types and sizes are supported.
Changing card key attribute for: Encryption key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 2
Please select which elliptic curve you want:
   (1) Curve 25519 *default*
   (4) NIST P-384
   (6) Brainpool P-256
Your selection? 1
The card will now be re-configured to generate a key of type: cv25519
Changing card key attribute for: Authentication key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 2
Please select which elliptic curve you want:
   (1) Curve 25519 *default*
   (4) NIST P-384
   (6) Brainpool P-256
Your selection? 1
The card will now be re-configured to generate a key of type: ed25519
```

5. Generate a key pair: do not make a backup; use your real name and `@talos-systems.com` email;
   set key to expire in 2 years:

```
gpg/card> generate
Make off-card backup of encryption key? (Y/n) n

Please note that the factory settings of the PINs are
   PIN = '123456'     Admin PIN = '12345678'
You should change them using the command --change-pin

Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 2y
Key expires at Wed Jul 26 17:53:41 2023 MSK
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Alexey Palazhchenko
Email address: alexey.palazhchenko@talos-systems.com
Comment:
You selected this USER-ID:
    "Alexey Palazhchenko <alexey.palazhchenko@talos-systems.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
gpg: key 5F06850601C47617 marked as ultimately trusted
gpg: directory '/Users/aleksi/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/Users/aleksi/.gnupg/openpgp-revocs.d/D2356ADE54050F011923004F5F06850601C47617.rev'
public and secret key created and signed.
```

Securely backup the revocation certificate.

6. Enforce mandatory physical user interaction for all key operations:

```
gpg/card> uif 1 permanent

gpg/card> uif 2 permanent

gpg/card> uif 3 permanent
```

7. Check settings:

```
gpg/card> list

Reader ...........: Yubico YubiKey OTP FIDO CCID
Application ID ...: D2760001240100000006154577040000
Application type .: OpenPGP
Version ..........: 0.0
Manufacturer .....: Yubico
Serial number ....: 15457704
Name of cardholder: [not set]
Language prefs ...: [not set]
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: ed25519 cv25519 ed25519
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 4
KDF setting ......: off
UIF setting ......: Sign=on Decrypt=on Auth=on
Signature key ....: D235 6ADE 5405 0F01 1923  004F 5F06 8506 01C4 7617
      created ....: 2021-07-26 14:54:12
Encryption key....: 5C71 862D C402 1926 4154  D18E AB20 4201 828F 396B
      created ....: 2021-07-26 14:54:12
Authentication key: 41C9 3468 2282 E380 F25A  4BFB 5E1A EADF D129 C2A0
      created ....: 2021-07-26 14:54:12
General key info..:
pub  ed25519/5F06850601C47617 2021-07-26 Alexey Palazhchenko <alexey.palazhchenko@talos-systems.com>
sec>  ed25519/5F06850601C47617  created: 2021-07-26  expires: 2023-07-26
                                card-no: 0006 15457704
ssb>  ed25519/5E1AEADFD129C2A0  created: 2021-07-26  expires: 2023-07-26
                                card-no: 0006 15457704
ssb>  cv25519/AB204201828F396B  created: 2021-07-26  expires: 2023-07-26
                                card-no: 0006 15457704

gpg/card> quit
```

Check `Key attributes`, `UIF setting`, keys.


8. Export your public key.
   Somewhat surprisingly, YubiKey does not store it, and it is required for working on another computer.

```sh
$ gpg --armor --export D2356ADE54050F011923004F5F06850601C47617 > key.pgp
```

On other machine import it and mark as ultimately trusted:

```sh
$ gpg --armor --import key.pgp
```

```
$ gpg --edit-key D2356ADE54050F011923004F5F06850601C47617
gpg (GnuPG) 2.3.1; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  ed25519/5F06850601C47617
     created: 2021-07-26  expires: 2023-07-26  usage: SC
     card-no: 0006 15457704
     trust: unknown       validity: unknown
ssb  ed25519/5E1AEADFD129C2A0
     created: 2021-07-26  expires: 2023-07-26  usage: A
     card-no: 0006 15457704
ssb  cv25519/AB204201828F396B
     created: 2021-07-26  expires: 2023-07-26  usage: E
     card-no: 0006 15457704
[ unknown] (1). Alexey Palazhchenko <alexey.palazhchenko@talos-systems.com>

gpg> trust
sec  ed25519/5F06850601C47617
     created: 2021-07-26  expires: 2023-07-26  usage: SC
     card-no: 0006 15457704
     trust: unknown       validity: unknown
ssb  ed25519/5E1AEADFD129C2A0
     created: 2021-07-26  expires: 2023-07-26  usage: A
     card-no: 0006 15457704
ssb  cv25519/AB204201828F396B
     created: 2021-07-26  expires: 2023-07-26  usage: E
     card-no: 0006 15457704
[ unknown] (1). Alexey Palazhchenko <alexey.palazhchenko@talos-systems.com>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

sec  ed25519/5F06850601C47617
     created: 2021-07-26  expires: 2023-07-26  usage: SC
     card-no: 0006 15457704
     trust: ultimate      validity: unknown
ssb  ed25519/5E1AEADFD129C2A0
     created: 2021-07-26  expires: 2023-07-26  usage: A
     card-no: 0006 15457704
ssb  cv25519/AB204201828F396B
     created: 2021-07-26  expires: 2023-07-26  usage: E
     card-no: 0006 15457704
[ unknown] (1). Alexey Palazhchenko <alexey.palazhchenko@talos-systems.com>
Please note that the shown key validity is not necessarily correct
unless you restart the program.

gpg> quit
```

9. Configure GnuPG to use pinentry program:

On macOS:
```
$ echo 'pinentry-program /usr/local/bin/pinentry-mac' > ~/.gnupg/gpg-agent.conf
```

On Linux: TODO.

10. Configure git:

```
$ git config --global user.email alexey.palazhchenko@talos-systems.com
$ git config --global user.signingkey D2356ADE54050F011923004F5F06850601C47617
$ git config --global commit.gpgSign true
$ git config --global tag.gpgSign true
```

11. Try to sign something locally:

```
$ gpg -K
sec>  ed25519 2021-07-26 [SC] [expires: 2023-07-26]
      D2356ADE54050F011923004F5F06850601C47617
      Card serial no. = 0006 15457704
uid           [ultimate] Alexey Palazhchenko <alexey.palazhchenko@talos-systems.com>
ssb>  ed25519 2021-07-26 [A] [expires: 2023-07-26]
ssb>  cv25519 2021-07-26 [E] [expires: 2023-07-26]

$ git commit -s
```

After editing the commit message, the PIN entry program should ask for a PIN.
After that, a physical tap on the button on the YubiKey should be required.

12. Distribute keys

```sh
gpg --keyserver hkps://keys.openpgp.org --send-key D2356ADE54050F011923004F5F06850601C47617
gpg --keyserver keyserver.ubuntu.com --send-key D2356ADE54050F011923004F5F06850601C47617
```

keys.openpgp.org has an additional step for verifying an email address; do it [there](https://keys.openpgp.org/manage).

Additionally, if you already use [Keybase](https://keybase.io), you may upload your key there.
See [this issue](https://github.com/keybase/keybase-issues/issues/4025) for details.

13. Configure YubiKey to work over SSH

If you plan to use git on a remote server, you need to configure SSH.
See [this page](https://wiki.gnupg.org/AgentForwarding) for details.
If everything is configured correctly, you should see similar lines in `ssh -v <HOST>` output:

```
debug1: Remote connections from /run/user/0/gnupg/S.gpg-agent:-2 forwarded to local address /Users/aleksi/.gnupg/S.gpg-agent.extra:-2
...
debug1: remote forward success for: listen /run/user/0/gnupg/S.gpg-agent:-2, connect /Users/aleksi/.gnupg/S.gpg-agent.extra:-2
```

14.  Add your public key to GitHub account [there](https://github.com/settings/keys).

### Git commit signing

After configuration, all commits should be signed by default; there is no need to pass `-S` flag (but `-s` flag
should still be used for adding "Signed-off" lines).
You will be asked a PIN once; it is cached by `gpg-agent`.
The button on the YubiKey should be tapped on each commit.

### GitHub flow

If we want all commits to be signed by our keys, we have only one option - ensure that all PRs are merged
in fast-forward mode and contain exactly one signed commit.
Unfortunately, GitHub does not support it natively.

We should modify ci-bot and CI configuration to:

1. Check that PR's commit is verified and signed by the key that is both set in author's settings (using an URL
   like https://github.com/AlekSi.gpg) and is present in bot's build configuration (see requirement #2).
2. Check that PR can be merged with `git --ff-only`.
3. Modify `/lgtm` command to perform `git --ff-only` or equivalent
   (see [this StackOverflow question](https://stackoverflow.com/questions/55800253/how-can-i-do-a-fast-forward-merge-using-the-github-api)).

We should also update repositories settings and protected branches configuration to forbid direct pushes
to the base branches, among other things.
Probably, those settings should be automated by Kres or something else.

> There are two other options: allow GitHub to [sign merged and squashed PR with their own key](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/about-commit-signature-verification) (see [this commit](https://github.com/AlekSi/talos/commit/fe0d975391ea4d7545fc16a30fe726b5127f9261)) or push directly to the base branch (see [this commit](https://github.com/talos-systems/bldr/commit/3198175d11e21abbc1982ef4efeed45acd817f20)).
> The former option doesn't work for us as we want to use our own keys.
> The latter makes it too easier to screw up.

After that is done, we should enable "Vigilant mode" on GitHub and merge all PRs with `/lgtm` command.
