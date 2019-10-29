---
feature: secrets_for_services
start-date: 2019-10-29
author: Dmitry Goldin
co-authors: (find a buddy later to help our with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

This RFC introduces some interfaces and library functions to help managing
secrets for NixOS systemd services modules.

# Motivation
[motivation]: #motivation

There is currently a lack of a consistent and safe mechanism to make secrets
available to systemd services in NixOS. Various modules implement it in various
ways across the ecosystem. There have also been ideas like adjustments to the
Nix Store, which would allow for non-world-readable files, but has made no
progress in several years.

Especially with the introduction of SystemD's `DynamicUser` simple approaches
manually managing permissions of some out-of-store files could become cumbersome
or slow the adoption of DynamicUser throughout the nixpkgs modules.

The approach outlined in this RFC aims to solve only a part of the secrets
management problem: How to make secrets that are already accessible on the
system (be it through a secrets folder only readable by root, or a system like
vault).

Shipping secrets is already solved sufficiently by krops, nixops, git-crypt,
simple rsync etc.

The idea here is to allow for flexibility in the way secrets are actually
delivered to the system while providing a consistent and unintrusive mechanism
that could be applied widely across the service modules that does not require
large code-changes and allows for a gradual transition.

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the ecosystem to understand, and implement.  This should get
into specifics and corner-cases, and include examples of how the feature is
used.

Fundamental assumptions:
* Delivery of secrets to target systems is a solved problem
* It's sufficiently secure to store the secrets or access tokens in a location
  only accessible by root on the system
* Interactive unlocking scenarios are not a big issue (???)
* Linux namespaces are sufficiently secure
* The service can be run using `PrivateTmp`

Invariants:
* Secrets store is only accessible to root
* A set of secrets are made available to a set of services only for the duration
of their execution
* Retrieved secrets are only accessible to the services processes and root
* Retrieved secrets are reliably cleaned up when services are stopped, killed,
crash or the system is restarted.

Core components:

* Secrets store: a secure file-system based location, in this document `/etc/secrets`
* A *fetcher* function: a function of the type SecretId -> () whose task it is
to resolve the secret identifier, retrieve the secret and place it in the
service process' private namespace within /tmp name
* Simple helper functions to *enrich* expressions defining systemd services with
  secrets.
* "Side-cart" service: A privileged systemd service running the fetcher function
to retrieve the fetcher function, and initially create the service namespace.



TODO: Maybe there is a nicer way to depend on secrets than by "path"
# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

Continued proliferation of individual per-module solutions per default. Persistent confusion by new-comers and veterans alike, about
what the a recommended way looks like.

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD or unknowns?

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature
without being directly part of the work?

* Transition of most critical services to use proposed approach
* Implementation of more supported secret stores
