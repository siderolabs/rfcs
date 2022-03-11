# Container Signing

## Summary

Establish patterns for easily signing and verifying all containers used in the
talos build and release pipeline to have strong protection against tampering.

## Motivation

It is valuable to be able to cache various stages of the build process as
container images, however a compromised CDN or a MITM attack could potentiall
undo all the hard work done elsewhere to ensure supply chain integrity.

It should be easy to sign images such that the Talos team or third parties can
easily pull and verify images to rapidly build Talos release artifacts with
high trust without having to fully rebuild the entire stack each time.

Containers will also be distributed for use within booted Talos OS environments
and it will be useful to have a generic means of verifying containers signed
by Talos, or for users to verify containers signed by themselves or third
parties they trust.

## Assumptions

* Any CDN can be compromised and attempt to swap in malicious assets
* Any pipeline that consumes container images could be subject to a MITM
* An adversary may attempt "downgrade" attacks to vulnerable past images
* Release engineers are distributed and any one key can be lost or compromised

## Requirements

* All private keys MUST be generated according to specifications in RFC 003
* It SHOULD generally be possible to rotate keys within one business day
* It MUST always be possible to certify new keys with old keys.

## Considerations

### Notary

#### Advantages

* Extremely well documented implementation of the TUF specification
* Container agnostic. Can be used to sign anything you can hash
* Already the foundation of Docker Content Trust
* Wide tooling support in the Docker Ecosystem
* Use of Yubikeys is expected and documented as a recommended path
* Can have complete control of own key supply chain
* Has built-in concept of a CA key and delegated easily rotated repo keys
  * Can designate contributors with limited power to sign releases

#### Disadvantages

* No concept of multi-sig and will require complex shared key logistics
* Requires maintaining dedicated web services for signing and verification
* Kubernetes does not have native support for DCT or Notary
  * Use case is just build images, so may not be a problem

### Cosign

#### Advantages

* Native Kubernetes support
* Wide collaboration with many major containerization tooling providers

#### Disadvantages

  * No concept of multi-sig or delegated keys yet
  * Tooling developed rapidly in advance of formal spec or threat model
  * Requires central trust in 5 individuals
  * Global root keys are maintained on online workstations
  * Global root key repo itself is solo maintained with no signing of any kind
  * Hardcoded root key lock-in on Yubico and Github
  * Fulcio "keyless signing" CA system is currently experimental and immature
    * Not actually keyless. Uses remotely certified local keys
      * Still requires the signing system be fully trusted
    * And by design letting other people sign for you is a major SPOF

### Skopeo

#### Advantages

  * Native K8s support via CRI-O
  * Works with well understood and recently revitalized OpenPGP standard
  * Container agnostic. Can be used to sign anything you can hash
  * Requires no hosted infra other than flat CDN to store detached signatures
  * Already well documented as default in OpenShift
  * Wide tooling support for many use cases
  * Wide hardware support for both remote and personal hardware security modules
  * Already used for code and tag signing

#### Disadvantages
  * No concept of multi-sig or delegated keys yet

## Scope

  * Signing containers used in Talos OS build system
  * Sign containers distributed for use within booted Talos OS systems
  * Verify all Talos created containers before running them

## Design

### Requirements

  * MUST have a practical path to work in a distributed team
  * MUST maintain all keys in hardware security modules
  * MUST have a path where published binaries
  * MUST be able to have key duplicated on multiple devices
  * SHOULD be modular enough to support alternative signing schemes

### Threat Model

  * Any one engineer can be compromised and sign a malicious artifact
  * Any key exposed in plain text to a human will be compromised
  * Any key exposed to an internet connected system will be compromised
  * Vulnerbilities will be discovered that warrant key rotation in the future

### Approach

Skopeo seems to be the most flexible and easy to reason about while making
use of OpenPGP signing which is already used elsewhere at Talos for things
such as commit signing.

The signing used internally within the Talos build system can be very
opinionated here for integrity and this need not be something users will
need to be familiar with.

For containers within the booted Talos OS system, we want verification to be
modular as other environments might depend on containers signed with other
popular signing schemes.

### Generation

These steps will help generate a long lived (but still rotatable) PGP keychain
for use with a Security Token with an offline backup. This backup will allow
the creation of duplicate keys and the ability to recover in the case of a
lost/damaged key.

The Master Key will be kept offline and used only for enrolling new subkeys
which will be used for most operations. It will also be possible to revoke and
rotate subkeys easily. Rotating the master key is more difficult and requires
redistribution, so it is recommended this be kept offline and rarely used.

You should perform these steps from an "airgapped" system, such as from a Linux
Live-CD not connected to the internet.

One such Live-CD that has the needed tools is [Airgap](https://github.com/distrust-foundation/airgap)

1. Prepare Offline Storage Directory

  This directory needs to be kept on an offline device (USB drive, etc) in a
  stable location (Fireproof box, safe, under your mattress, etc)

  This will contain:
   * Backup plaintext secret keys, useful for emergencies
   * Revocation certificate
   * Master key, used to sign other keys

  Insert the USB drive and observe the path it mounts to.

  Here we will assume /Media/UsbStick

  In the terminal, create a new directory on this device:

  ```
  $ mkdir /Media/UsbStick/pgp
  ```

2. Generate a Master Key

  This important key will be used to sign other keys and for generation of the
  subkeys that are needed for the Security Token.  This tutorial will start by
  generating a new key in "Expert mode" to allow the needed customization.

  ```
  $ gpg --full-gen-key --expert
  ```

  Be sure to enable the following options:

  * RSA (set your own capabilities)
  * Toggle off "Encrypt" and "Sign" capabilities leaving only "Certify"
  * Set the keysize to 4096 bits
  * Set expiration at 1y (will require yearly access to bump)
  * Set dedicated signing email address
  * Set strong passphrase.
    * Consider a 12 word (128 bit) BIP39 passphase as it is easy to transcribe
    * Save passphrase to a text file
    * Encrypt passphrase to personal PGP keys of at least two administrators
    * Save encrypted passphrase files to Backup directory

3. Generate Revocation Certificate

  If your keys are compromised for any reason you will need a revocation
  certificate to revoke (untrust) your keys.  Generate the revocation certificate
  and save it in the backup location.

  ```
  $ gpg --output /Media/UsbStick/pgp/revocation-certificate.txt --gen-revoke B44C2E4AA
  ```

  * Reason: 0 (No reason specified)

4. Generate Signing Subkey

  Now we will generate the key used for day to day release signing operations.

  ```
  $ gpg --edit-key --expert B44C2E4A
  ```

  * "addkey" to add a subkey
  * Select RSA (sign only)
    * ECC is reasonably widly supported now but RSA 4096 is universal
  * Set keysize to 4096 bits
  * Set expiration for one year


5. Backup Keys

Now backup your keys to your previously created offline storage directory:

```
$ gpg -a --export-secret-keys B44C2E4A > /Media/UsbStick/pgp/B44C2E4A.master.asc
$ gpg -a --export-secret-subkeys B44C2E4A > /Media/UsbStick/pgp/B44C2E4A.subkeys.asc
$ gpg -a --export B44C2E4A > /Media/UsbStick/pgp/B44C2E4A.public.asc
```

6. Write Signing Subkey to multiple Security Tokens

  ```
  gpg --edit-key B44C2E4A
  ```

  * "toggle" to unselect the master key
  * "key 1" to select the first subkey
  * "keytocard" to write signing key to a yubikey
    * Repeat for the desired number of signers

  Note that "keytocard" removes the key from the keychain once it is moved to
  a card. You will need to exit and re-import the subkeys after each write to
  a Yubikey:

  ```
  $ gpg --import /Media/UsbSgick/pgp/B44C2E4A.subkeys.asc
  ```

7. Verify

  Shut down the system and boot it again.

  Verify you can load the backups and sign a file with the master key.

  This is a bit cumbersome but it is vital to prove your backups work.

  ```
  $ gpg --import /Media/UsbSgick/pgp/B44C2E4A.*
  ```

  You will need a personal Yubikey of an administrator to decrypt the
  passphrase in order to complete the import.


### Distribution

1. Distribute a copy of the backup drive to all release engineers
2. Set physical touch settings on all Yubikeys

  ```
  ykman openpgp set-touch sig fixed
  ```

3. Distribute signing subkey Yubikeys to all release engineers
4. Distribute public key to keyservers

    ```
    gpg --keyserver hkps://keys.openpgp.org --send-key YOUR_KEY_ID
    gpg --keyserver hkps.pool.sks-keyservers.net  --send-key YOUR_KEY_ID
    gpg --keyserver pgpkeys.urown.net --send-key YOUR_KEY_ID
    gpg --keyserver keyserver.ubuntu.com --send-key YOUR_KEY_ID
    ```

5. Publish public key via WKD on your domain

  See: https://wiki.gnupg.org/WKDHosting

6. Link to public key on Website

### Signing


Containers are signed on upload with Skopeo as follows:

```
skopeo copy \
  --sign-by team@siderolabs.com \
  registry:image-name:v1.0.0 \
  remote.registry/namespace/image-name:v1.0.0
```

### Verification

If using docker you can force verification on pull.

To do so, edit /etc/containers/policy.json to contain:

```
{
  "default":[
    {
      "type":"reject"
    }
  ],
  "transports":{
    "docker":{
      "docker.io/<USER_NAME>":[
        {
          "type":"signedBy",
          "keyType":"GPGKeys",
          "keyPath":"/<path-to-public-key>/public.gpg"
        }
      ]
    }
  }
}
```

Skopeo can also manually verify a detached signature against a manifest:

```
skopeo standalone-verify \
  busybox-manifest.json \
  registry.example.com/example/busybox \
  1D8230F6CDB6A06716E414C1DB72F2188BB46CC8 \
  busybox.signature
```

In the wild containers may use other signing methods like Cosign or Notary.

For maximum flexibility it is advised that if possible pulling and verification
of containers be abstracted to a dedicated and locally trusted container.

## Glossary

## References

* [#! Key generation guide](https://github.com/hashbang/book/blob/master/content/docs/security/SSH.md)
* [gpg.wtf guide](https://gpg.wtf)
