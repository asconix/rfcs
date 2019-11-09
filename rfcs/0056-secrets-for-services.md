---
feature: secrets_for_services
start-date: 2019-10-29
author: @d-goldin
co-authors: (find a buddy later to help our with the RFC)

shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

This RFC introduces an interface, terminology and library functions to help managing
secrets for NixOS systemd services modules.

# Motivation
[motivation]: #motivation

There is currently a lack of a consistent and safe mechanism to make secrets
available to systemd services in NixOS. Various modules implement it in various
ways across the ecosystem. There have also been ideas like adjustments to the
Nix Store (like issue https://github.com/NixOS/nixpkgs/issues/8), which
would allow for non-world-readable files, but has made no progress in several years.

Especially with the introduction of Systemd's `DynamicUser` more traditional
approaches of manually managing permissions of some out-of-store files could
become cumbersome or slow the adoption of DynamicUser and other sandboxing
features throughout the nixpkgs modules.

The approach outlined in this document aims to solve only a part of the secrets
management problem, namely: How to make secrets that are already accessible on the
system (be it through a secrets folder only readable by root, or a system like
vault) to non-interactive services in a safe way.

It assumes that shipping secrets is already solved sufficiently by krops, nixops,
git-crypt, simple rsync etc, and if not, that this can be addressed as a separate
concern without needing change to the approach proposed here. Further, its outside
of scope to ensure other properties of the secret store, such as encryption at rest
and so forth.

The main idea here is to allow for flexibility in the way secrets are actually
delivered to the system while providing a consistent and unintrusive mechanism
that could be applied widely across the service modules that does not require
large big-bang code-changes and allows for a gradual transition of nixos services.

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the ecosystem to understand, and implement.  This should get
into specifics and corner-cases, and include examples of how the feature is
used.

To summarize, necessary preconditions:

* Delivery of secrets to target systems is a solved problem
* It's sufficiently secure to store the secrets or access tokens in a location
  only accessible by root on the system
* The secrets store locations is secure at rest, such as full-disk-encryption.
* Interactive unlocking scenarios are should be treated separately
* Linux namespaces are sufficiently secure
* The service can be run using `PrivateTmp`

Design goals:
* A set of secrets are made available to a set of services only for the duration of their execution
* Retrieved secrets are only accessible to the service processe and root
* Retrieved secrets are reliably cleaned up when services are stopped, killed,
  crash or the system is restarted.

Core concepts and terminology:

* *Secrets store*: a secure file-system based location, in this document
  `/etc/secrets`, only accessible to root
* A *fetcher* function: a function whose task it is to resolve the secret
  identifier, retrieve the secret and place it in the service process' private
  namespace within /tmp name
* Simple helper functions to *enrich* expressions defining systemd services
  with secrets
* "Side-cart" service: A privileged systemd service running the fetcher
  function to retrieve the fetcher function, and initially create the service
  namespace.
* Secrets scope: provides a context in swhich secrets are accessible as
  attributes resolving to path names within the private namespace

The general idea is centered around this simple process:

A privileged side-cart service is launched first, creates a namespace, executes
the fetcher function which retrieves the secrets and copies them into the private
tmpfs. The side-cart service binds to the target service to ensure it's shut
down and the namespace is destroyed when the target service disappears. This
service uses `RemainAfterExit` to keep the namespace open for other services
without relying on arbitrary timing.

The target service launches once the side-cart service has been launched,
joins it's namespace and is able to access the secrets provided in the shared
tmpfs in `/tmp`.
The service is not free to access the file in whichever way it wants -
for instance just passing the path to the software to be launches as
an argument, or load it up into an environment variable.

Example of user-facing API:

```
let
   secretsScope = mkSecretsScope {
     loadSecrets = [ "secret1" "secret2" ];
     type = "folder";
   };
in
   systemd.services = secretsScope ({ secret1, ... }: {
      foo = {
        description = "Simple test service using a secret";
        serviceConfig = {
          ExecStart = "${pkgs.coreutils}/bin/cat ${secret1}";
          DynamicUser = true;
        };
      };
    };
```

This is a minimal example of a service depending on a secret `secret1`.

More specifically, in this example a secrets scope is created - to allow for
extensibility and differentiation a store has a type. In this case "folder"
denotes a secrets store in the form of a root-only accessible locked down
directory on the local filesystem. Here we want to acquire access to
2 secrets, and 2 secrets only, which are specified in `loadSecrets`, by
their id. How a secrets identifier is resolved, should be up to the fetcher
function and here it's just trivially the file-name (this of course does not
allow for file extensions).

Those secrets are then made acessible to the target service unit definitions as
arguments passed into a lambda within the scope. Those arguments then point to
some private location within the namespace - in our case `secret1 ->
/tmp/secret1`.

The resolution and location is decided by the inner workings of
the implementation and should be rather unimportant to the user, as it could
potentially change, if other private locations besides `/tmp` become available.

It is of course still possible to just point to the file locations given the
knowledge, but is less convenient and would not result in build time errors
when wrong paths are specified - thus the arguments add a little bit of
convenience and safety, aside from the indirection they offer.

For every service defined this way in a scope, a side-cart container is generated
_per service_ and wired up with the target service. This means that the ability
to create a scope does not break isolation between multiple target services
but can add a little bit of developer convenience.

For this the target service is forced/asserted to utilize `PrivateTmp=true`.

## Rotating secrets

Right now, secrets rotation is not done automatically. When new secrets are
pushed, it is the responsibility of the user to restart the services affected.

It is assumed that once secrets are rotated, old secrets will become invalid and
no further harm is done aside from failing to access the resources (and possibly
restart on its own).

It would be possible to allow for automatic restarts using systemd path monitors.
Also see _Future work_.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

I can't really think of a serious drawback right now, but hopefully the
RFC process can surface some.

One aspect is of course the additional number of services generated.

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

* One approach that has been proposed in the past is a non-world readable store,
  in issue #8 (support private files in the nix store, from 2012). While this would
  be pretty great, it's rather complex and has not made huge progress in a while.

* "Classical" approach of just storing secrets readable only to a service user and
  utilizing string-paths to reference them. This does not work well anymore with
  DynamicUser.

Impact of not doing this:

Continued proliferation of individual per-module solutions per default.
Persistent confusion by new-comers and veterans alike, about what the a
recommended way looks like and a variety of different approaches within
nixos service modules.

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD or unknowns?

* Is it sufficient to put responsibility on restarting services after key changes
  onto the user or would an automated mechanism be better? There are also down-sides
  to an automated mechanism too.

* Right now the POC does not support joining other namespaces if needed. Also, this has
  some security implications, as the other process would be able to read the secrets
  files loaded into the private tmp. But this would be also the case in other scenarios.
  Is it sufficient to just emit a warning? Should there be a "i know what I'm doing" flag?

* One sidecart per secret? join multiple namespaces? Or one sidecart per scope?
  I think one side-cart per secret could cause problems with shared tmp's
  between different services and thus be less isolated.

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature
without being directly part of the work?

* When using a scope with multiple services, ideally only the secrets
  references in the services definition should be made available to each
  service. Right now all the secrets of the scope are blindly copied.
* Transition of most critical services to use proposed approach
* Implementation of more supported secret stores, such as vault
* Optional restarting for services affected by rolled secrets
* Merging some attributes better than in the POC - like JoinsNamespaceOf
* Provide simple shell functions for features like loading a file into an environment
  variable
