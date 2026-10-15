# Security policy

Plugins run on routers people trust with their traffic, so security problems
in this template, which other people's plugins start from, matter a lot to
us.

## Reporting a problem

**Please don't open a public issue for security problems.** Instead, use
GitHub's private vulnerability reporting: on the repository page, go to
*Security → Report a vulnerability*. Include:

- what you found and how to reproduce it,
- which file or workflow it is in,
- what an attacker could do with it.

We'll answer within a week, keep you updated, and credit you in the release
notes if you want.

Problems in RasBeacon's handling of plugins (a plugin doing more than its
permissions allow, signatures not being checked, guests or the internet
reaching a plugin) belong to
[RasBeacon](https://github.com/aazam476/RasBeacon/security) instead.
Problems in a plugin made from this template belong to that plugin's
authors.

## What we consider security problems

For example:

- the build or release workflows exposing the signing key, or publishing
  files that weren't built from the repository;
- the tools producing a `.rba` that holds something it shouldn't (a key, a
  file from outside the plugin's folder);
- the guide or AGENT.md teaching something unsafe (asking for permissions a
  plugin doesn't need, unpinned images, secrets in settings that aren't
  passwords, keys in the repository).

## Supported versions

Only the newest version of the template gets fixes. Repositories made from it
don't receive them automatically: compare your `tools/` and
`.github/workflows/` with this repository's now and then.
