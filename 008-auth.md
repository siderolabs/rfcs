# Authentication and Authorization

## Summary

Right now, the generation of client authentication keys and RBACs thereof is a
very manual process which is not easy to manipulate for the average user.
Moreover, it does not encourage best practices such as having protection on
private keys (via hardware tokens or passwords), shorter-duration certificates,
user-oriented keys, or even the use of privilege-reduced operational certs.

We should build some tooling to facilitate all of these needs and encourage best
practices generally.

Design goals:
- simplify processes
   - provide easy generation of client keypairs
     - provide options for protecting client private keys:
       - password-protection
       - hardware keys
   - provide easy generation of user certificates
     - generate CSR from `talosctl`
     - generate certificate from CSR using `talosctl`
     - apply RBAC policies to certificates
   - provide easy way to generate and update CRLs
     - figure out how and where to house CRLs
       - public service? (what if network is not available?)
       - in etcd? (what if etcd is down?)
       - on each node? (what if nodes are down during publish?)
- greater authentication flexibility
  - support external sources of authentication, such as OIDC servers (dex,
    Google, etc)
  - allow key pairs and certificates to be easily bound to managers instead of
    roles
- greater authorization flexibility
  - allow decoupling roles from key sets
  - allow dynamic and arbitrary mappings of authentication data to roles

Related issues:
 - https://github.com/talos-systems/talos/issues/3306

## Case Studies

### Any larger organisation

In general, passing authentication credentials around on the network is an
anti-pattern.
Once the set of managers of a system grows beyond one, greater flexibility in
the distribution and management of key and assignment material is necessary.

### Organisations with an existing SSO system

When an organisation already has in place a common authentication system,
we should be able to externalise the sourcing of our authentication to their
system.
However, we still need to be able to control mappings of those identities to
Talos roles.

## Motivation


Why are we doing this? What use cases does it support? What is the expected
outcome?

# Assumptions

What assumptions do we make about the elements this RFC touches?

If this is a feature with significant memory overhead, do we assume end-users
can all meet it?

If this RFC is to make a security improvement, what threat model is it
operating under? List any assumptions we make about the capabilities we feel
we can reasonably make about potential adversaries.

# Requirements

What exactly is the outcome we want and want to enforce moving forward?

List some MUST, MUST NOT, and SHOULD statements here that people will be
expected to use as a quick reference to consider when doing engineering work.

# Design

This should be a simple bulleted explanation of the core process or outcome
that we expect when this RFC is implemented.

This could be a developer workflow, an automation workflow, a specific sequence
of events, etc.

Include any diagrams as appropriate, favoring text-based sources others can
easily edit like mermaid, UML, etc. where possible, with external files sharing
the same filename prefix as this document.

# Examples

Detail one or more hypothetical but plausible real-world scenarios where this
RFC would offer a tangible benefit.

# Drawbacks

Provide a few bullets honestly detailing the downsides of implementing this.

Will it slow down development? Cost more resources? Etc.

# Alternatives

List any alternatives others have tried, including the current solution if
there is one, to help others quickly catch up on how we got here.

# Scope

List a few items that are clearly in scope, anything that we want to do that is
explicitly deferred to future work.

## Questions

What open questions still exist that we can't answer right now?






# Client authentication


## Phase 1

First, we need to break apart the `talosconfig` from the authentication PKI.
This allows us to, among other things: 
 - encourage the distribution of `talosconfig` without secret material included
 - encourage the disuse of root admin keys for normal operation
 - encourage the use of user-specific secret material which is never sent over
   the wire
 - use a common user keypair across multiple Talos clusters

Some minor additions to `talosconfig` will be useful:
 - top-level `defaultAuthIdentity` reference
 - cluster-level `authIdentity` reference
 - cluster-level `clusterId` reference
 - cluster-level `certs` list

Perhaps add a `-i` option to `talosctl` which tells Talos to use the
given auth identity.

Construct and use `~/.config/talos/auth` (or ~/.talos/auth, for non-XDG-standard
machines) to house the Talos identities on the client machine.

Modify `talosctl` to use the search the auth directory to resolve authentication
material when connecting to Talos.

Proposed structure of `~/.config/talos/auth`:
 - `id_<name>.key` : private key of the given id
 - `id_<name>.pub` : public key of the given id

Proposed CLI subcommand:  `auth`:
 - `talosctl auth approve` : approve a user's certificate request
 - `talosctl auth request` : create a request to access a cluster
 - `talosctl auth gen[erate] id` : generate a user keypair

 - `talosctl config strip-auth` : strip the secret auth material from `talosconfig`
 - `talosctl config import-cert` : import a signed certificate


## Phase 2

Focusing on the most basic operational requirements, we need to let `talosctl`:
 - Generate a client keypair
 - Generate a CSR from a client keypair
 - Generate a signed certificate from a CSR

In order to make things work a little better for user-oriented

## Addendum

`talosctl kubeconfig` should offer RBAC-restricted versions:  namespace
restrictions, for instance.
Require `--admin` to generate an admin cert
Maybe enforce one of `--namespaces` or `--admin`


## Auth service

We should have an auth credentials service (essentially, `trustd`) which:
 - can use an external reference for authentication
 - generate client certificates based on this external reference

Each external authentication delegation will be opt-in, requiring it to be
explicitly added to the machine config before it is allowed to run.

Base configuration resource:  `cluster.auth.delegations`.

### Kubernetes external reference

One such external reference can be Kubernetes itself.
The idea would be to have the machine config contain a list of mappings of
Kubernetes Service Accounts to Talos roles.
Each mapping should also allow an optional lifetime for the credentials, which
should default to some reasonable time (e.g. 1 Hour).

```yaml
cluster.auth.delegations:
 - name: kubernetes 
   kind: kubernetes
   kubernetesSpec:
     serviceAccountMap:
      - name: upgrade
        namespace: talos-upgrade
        ttl: 1h 
        role: administrator
```


### OIDC external reference

OpenID Connect is a common external authentication protocol which allows validation of
memberships.
Identity is validated by a trusted external service, using public key
cryptography, and claims are mapped to Talos client roles.

```
cluster.auth.delegations:
 - name: google
   kind: oidc 
   oidcSpec:
     isserUrl: https://accounts.google.com
     clientId: lkasd89714
     issuerCA: bXktZW5jb2RlZC1DQS1jZXJ0Cg==
     authorizationMap:
       - claims: email
           - name: email_verified
             value: "true"
           - name: email
             values:
               - "george@siderolabs.com"
               - "albert@siderolabs.com"
         talosRole: readonly
       - claims: admin_group
           - name: groups
             values:
               - "admin"
         talosRole: administrator
```

For any given authorizationMap, _all_ claims specified must match to qualify.

The general-purpose OIDC plugin is limited in its ability to parse complex
claims.
For more advanced parsing of claims, it is recommended that an
intermediate programmable OIDC system is used, such as [dex](https://dexidp.io).

Otherwise, some common OIDC providers may be supported by their own plugins
inside Talos.

