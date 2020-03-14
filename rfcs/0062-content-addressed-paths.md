---
feature: Simple content-adressed store paths
start-date: 2019-08-14
author: Théophane Hufschmitt
co-authors: (find a buddy later to help our with the RFC)
shepherd-team: @layus, @edolstra and @Ericson2314
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary

[summary]: #summary

Add some basic but simple support for content-adressed store paths to Nix.

We plan here to give the possibility to mark certain store paths as
content-adressed (ca), while keeping the other dependency-adressed as they are
now (modulo some mandatory drv rewriting before the build, see below)

By making this opt-in, we can impose arbitrary limitations to the paths that
are allowed to be ca to avoid some tricky issues that can arise with
content-adressability.

In particular, we restrict ourselves to paths that are:

- without any non-textual self-reference (_i.e_ a self-reference hidden inside a zip file)
- known to be deterministic (for caching reasons, see [caching]).

That way we don't have to worry about the fact that hash-rewriting is only an
approximation nor by the semantics of the distribution of non-deterministic
paths.

We also leave the option to lift these restrictions later.

This RFC already has a (somewhat working) POC at
<https://github.com/NixOS/nix/pull/3262>.

# Motivation

[motivation]: #motivation

Having a content-adressed store with Nix (aka the "Intensional store") is a
long-time dream of the community − a design for that was already taking a whole
chapter in [Eelco's PHD thesis][nixphd].

This was never done because it represents a quite big change in Nix's model,
with some non-totally-solved implications (regarding the trust model in
particular).
Even without going all the way down to a fully intensional model, we can
make specific paths content-adressed, which can give some important benefits of
the intensional store at a much lower price. In particular, setting some
critical derivations as content-adressed can lead to some substancial build
cutoffs.

# Detailed design

[design]: #detailed-design

The gist of the design is that:

- Derivations can be marked as content-adressed (ca), in which case each
  of their outputs will be moved to a content-addressed `ca` store path.

- This definitions includes today's fixed-output (ca) derivations.
  In addition, we will have new "floating-output"" ca derivations.

- We extended a little-known existing notion of normalizing fixed-output ca derivations to
  account `ca` derivations in general, and give it a name: "resolving" a derivation.

## Content-addressed derivations derivation

### Derivation model, and file format

- A **ca derivation** is one with a hash algorithm for every output.

- A fixed-output ca derivation has a single output, and a hash algorithm and (fixed) hash for that output.
  This is unchanged from today.

- A floating-output ca derivation has one or more outputs, each of which has the same hash algorithm but none of which have (fixed) hashes.

### Nix language

A **ca derivation** is a derivation with the `__contentAddressed` argument set
to `true` and the `outputHashAlgo` set to a value that is a valid hash name
recognized by Nix (see the description for `outputHashAlgo` at
<https://nixos.org/nix/manual/#sec-advanced-attributes> for the current allowed
values**.
It can specify its outputs normally with `outputs`, otherwise a single output, `out`, is assumed.

## Derivation resolution

Derivation resolution today replaces fixed-output ca derivations with their single content-addressed store output.
\[For reference, it is implemented by `hashDerivationModulo`.]
It does this by resolving input derivations and rewriting those to account for transitive dependencies with fixed-output ca derivations.

A **resolved** derivation is one which only references:

 - outputs of other (non content-addresed) resolved derivations
 - Existing store paths

Derivation resolution thus produces resolved derivations, and is idempotent, mapping resolved derivations to themselves.

resolution is (intentionally) not injective: If `drv` and `drv'` only differ because one depends on `dep` and the other on `dep'`, but `dep` and `dep'` are content-addressed and have the same output hash, then `resolve(drv)` and `resolve(drv')` will be equal.
This already shows up today when we change a `fetch*` function's definition and *don't* have a mass rebuild.

We generalize it to replace all ca derivations with the hashes of their in-use outputs, not just fixed-output ca derivations.

Derivation resolution is conceptually mutually recursive.
One function maps a pair of a derivation reference (currently a path, but in no way implying the derivation is serialized at that path) and an output to a content address (dependent pair of an algo and a hash in the form of that algo).
The other maps derivations to derivations, substitutes store path for input derivation.
The mutual recursion is important because if we want ultimately want a path/reference we will start with the first, but if we ultimately want a derivation (e.g. to write to a `.drv` file or build), we will start with the second.
The current implementation[^current-implementation] somewhat obscures this, but this proposal doesn't change when we want paths vs when we want derivations.[^mrec-uses]

Derivations written to disk are always resolved, but not including the root.
This is because we are writing derivations, not hashes/paths, to derivation.
If we replace a derivation with its single output path, there is no derivation!
Derivations being built are also resolved, not including the root.
This is because we need a derivation to do a build.
Hashes or paths to not say how to compute what they refer to!

### Remembering floating output hashes

Normalizing fixed outputs is easier than floating outputs, because the derivation contains the hash with which to create the ca store path.
By contrast, the information is not present for floating outputs, and must be gotten out of band.

For each output `output` of a floating-output ca derivation `drv`, we define

- its `outputId` **DrvOutputId(drv, output)** as the tuple `(hash(drv), output, truster)`, where `truster` is a reserved field for future use and currently always set to `"world"`.
  \[See trust model in the [future work section](#future-work)].
  This id uniquely identifies the output.
  We textually represent this as `hash(drv)!output[@truster]`.

- its concrete path **PathOf(outputId)** as the path on which the output will be stored on disk.

> Unresolved: should we already include the `truster` field in `DrvOutputId`
> even if it's not used atm? What would be the cost of adding it later?

At a minimum, we need to store output paths for resolved floating-output ca derivations, which, along with the fixed-output ca derivations' output hashes, generates the resolving function.
We can do that with a table in the SQLite database:

```sql
create table if not exists PathOf (
    drv integer not null,
    output text not null,
    truster integer not null,
    hash text not null,
)
```

However, we may also wish to memoize the function over other derivations which merely refer to ca derivations.
In this case, we can use this table:

```sql
create table if not exists PathOf (
    drv integer not null,
    output text not null,
    truster integer not null,
    path integer not null,
)
```

Note that we must replace the hash column with a path column (which is an integer because its a surrogate foreign key).
This is because while non-ca derivations which refer to ca-derivations do use a "modified" store path from the resolved derivation, it not a content-address store path as the resolved root derivation is still a regular legacy derivation.

## Nix-build process

Just like today, we only build resolved derivations.
This is so we take advantage of as much normalization as possible to avoid rebuilds.
When there were only fixed-output ca derivations, this was easy, as we always have the information to resolve any derivation.
But when there are also floating-output ca derivations, we don't necessary have the information we need in the table.
Thus we need to *interleave* building and resolving, until there is no work left.

The algorithm is presented in the following mutually recursive steps.

### Building a normal derivation

Same today, except in the case of the larger memo table (second option) there is one extra step:
We will add a row if the normal derivation refers to floating-output ca derivations.

1. Compute `resolve(drv)`

2. If already built, return, else build the derivation (and continue to next step)

3. *Assuming the memo table:* Add a new mapping `pathOf(drv!${output}) == ${output}(resolve(drv))` for each output `output` of `drv`.

### Resolution

Keep a set of "stuck" floating ca-derivations; i.e. those without a table entry.

1. Resolve as much as possible.

   - When encountering a floating ca derivation with no table entry, just move on but add the stuck derivation to the set.
     With the first table option, we can only look up resolved derivations, so we work bottom up (like call by value).
     With the second table option, we may have an memo entry for an intermediate (non-leaf) derivation, so we work up until we find an entry or reach a leaf node, and then work down (like call by need).

   - If never stuck, return with resolved derivation, we have finally resolved the derivation.

2. Build all stuck derivations

3. Repeat making more progress by using the new table entries from the newly-unstuck derivations.

### Building a ca derivation

These steps cover both fixed- and floating- output CA derivations, but the algorithm for fixed-output ca

1. Compute `resolve(drv)`

2. If already built, return, else build the derivation using temporary location for each  (and continue to next step)

3. Move each output to the content-addressed store path corresponding to its hash.
   For each output `$outputId` of the derivation, this gives us a (temporary) output path `$out`.
    - We compute a cryptographic hash `$chash` of `$out`[^modulo-hashing]
    - We move `$out` to `/nix/store/$chash-$name`
    - We store the mapping `PathOf($outputId) == "/nix/store/$chash-$name"`

4. Depends on whether outputs are fixed or floating:

    - *If fixed-output:* Check if the output matches the fixed hash in the derivation, if so, we move to the next stop, if not the build fails but the moved output is kept in place.

    - *If floating output:* Add a new mapping to the table for each output.

       - *If small generating table:* `hashOf(drv!${output}) == ${output}(resolve(drv))` for each output `output` of `drv`.

       - *If large memo table:* `pathOf(drv!${output}) == ${output}(resolve(drv))` for each output `output` of `drv`.

[^current-implementation]:

  The function `hashDerivationModulo`

[^mrec-uses]:

  Here are some examples of when each is used:

   - Derivations written to disk are always resolved, but not including the root.
     This is because we are writing derivations, not hashes/paths, to derivation.
     If we replace a derivation with its single output path, there is no derivation!

   - Derivations being built are also resolved, not including the root.
     This is because we need a derivation to do a build.
     Hashes or paths to not say how to compute what they refer to!

[^modulo-hashing]:

  We can possibly normalize all the self-references before
  computing the hash and rewrite them when moving the path to handle paths with
  self-references, but this isn't strictly required for a first iteration

# Examples and Interactions

[examples-and-interactions]: #examples-and-interactions

First to note is that this "retconning" of fixed-output derivations is crucial to keep the model of how Nix works concise.

## Full example

In this example, we have the following Nix expression:

```nix
rec {
  contentAddressed = mkDerivation {
    name = "contentAddressed";
    __contentAddressed = true;
    … # Some extra arguments
  };
  dependent = mkDerivation {
    name = "dependent";
    buildInputs = [ contentAddressed ];
    … # Some extra arguments
  };
  transitivelyDependent = mkDerivation {
    name = "transitivelyDependent";
    buildInputs = [ dependent ];
    … # Some extra arguments
  };
}
```

Suppose that we want to build `transitivelyDependent`.
What will happen is the following

1. We instantiate the Nix expression, this gives us three drv files:
   `contentAddressed.drv`, `dependent.drv` and `transitivelyDependent.drv`
2. We build `contentAddressed.drv`.
   - We first compute `resolve(contentAddressed.drv)`.
   - We realise `resolve(contentAddressed.drv)`. This gives us an output path
     `out(resolve(contentAddressed.drv))`
   - We move `out(resolve(contentAddressed.drv))` to its content-adressed path
     `ca(contentAddressed.drv)` which derives from
     `sha256(out(resolve(contentAddressed.drv)))`
   - We register in the db that `pathOf(contentAddressed.drv!out) == ca(contentAddressed.drv)`
3. We build `dependent.drv`
   - We first compute `resolve(dependent.drv)`.
     This gives us a new derivation identical to `dependent.drv`, except that `contentAddressed.drv!out` is replaced by `pathOf(contentAddressed.drv!out) == ca(contentAddressed.drv)`
   - We realise `resolve(dependent.drv)`. This gives us an output path
     `out(resolve(dependent.drv))`
   - We register in the db that `pathOf(dependent.drv!out) == out(resolve(dependent.drv))` We build `transitivelyDependent.drv`
4. We build `transitivelyDependent.drv`
   - We first compute `resolve(transitivelyDependent.drv)`
     This gives us a new derivation identical to `transitivelyDependent.drv`, except that `dependent.drv!out` is replaced by `pathOf(dependent.drv!out) == out(resolve(dependent.drv))`
   - We realise `resolve(transitivelyDependent.drv)`. This gives us an output path `out(resolve(transitivelyDependent.drv))`
   - We register in the db that `pathOf(transitivelyDependent.drv!out) == out(resolve(transitivelyDependent.drv))`

Now suppose that we replace `contentAddressed` by `contentAddressed'`, which evaluates to a new derivation `contentAddressed'.drv` such that the output of `contentAddressed'.drv` is the same as the output of `contentAddressed.drv` (say we change a comment in a source file of `contentAddressed`).
We try to rebuild the new `transitivelyDependent`. What happens is the following:

1. We instantiate the Nix expression, this gives us three new drv files:
   `contentAddressed'.drv`, `dependent'.drv` and `transitivelyDependent'.drv`
2. We build `contentAddressed'.drv`.
   - We first compute `resolve(contentAddressed'.drv)`
   - We realise `resolve(contentAddressed'.drv)`. This gives us an output path `out(resolve(contentAddressed'.drv))`
   - We compute `ca(contentAddressed'.drv)` and notice that the path already exists (since it's the same as the one we built previously), so we discard the result.
   - We register in the db that `pathOf(contentAddressed.drv'!out) == ca(contentAddressed'.drv)` ( also equals to `ca(contentAddressed.drv)`)
3. We build `dependent'.drv`
   - We first compute `resolve(dependent'.drv)`.
     This gives us a new derivation identical to `dependent'.drv`, except that `contentAddressed'.drv!out` is replaced by `pathOf(contentAddressed'.drv!out) == ca(contentAddressed'.drv)`
   - We notice that `resolve(dependent'.drv) == resolve(dependent.drv)` (since `ca(contentAddressed'.drv) == ca(contentAddressed.drv)`), so we just return the already existing path
4. We build `transitivelyDependent'.drv`
   - We first compute `resolve(transitivelyDependent'.drv)`
   - Here again, we notice that `resolve(transitivelyDependent'.drv)` is the same as `resolve(transitivelyDependent.drv)`, so we don't build anything

# Drawbacks

[drawbacks]: #drawbacks

- Obviously, this makes the Nix model more complicated than what it is now. In
  particular, the caching model needs some modifications (see [caching]);

- We specify that only a sub-category of derivations can safely be marked as
  `contentAddressed`, but there's no way to enforce these restricitions;

- This will probably be a breaking-change for some tooling since the output path
  that's stored in the `.drv` files doesn't correspond to an actual on-disk
  path.

# Alternatives

[alternatives]: #alternatives

[RFC 0017][] is another proposal with the
same end-goal. The big difference between these two is in the scope they cover:
RFC 0017 is about fundamentally changing the base model of Nix, while this
proposal suggests to make only the minimal amount of changes to the current
model to allow the content-adressed model to live in parallel (which would open
the way to a fully content-adressed store as RFC0017, but in a much more
incremental way).

Eventually this RFC should be subsumed by RFC0017.

# Unresolved questions

[unresolved]: #unresolved-questions

## Caching

[caching]: #caching

The big unresolved question is about the caching of content-adressed paths.
As [Eelco's phd thesis][nixphd] states it, caching ca paths raises a number of
questions when building that path is non-deterministic (because two different
stores can have two different outputs for the same path, which might lead to
some dependencies being duplicated in the closure of a dependency).
There exist some solutions to this problem (including one presented in Eelco's
thesis), but for the sake of simplicity, this RFC simply forbids to mark a
derivation as ca if its build is not deterministic (although there's no real
way to check that so it's up to the author of the derivation to ensure that it
is the case).

## Client support

The bulk of the job here is done by the Nix daemon.

Depending on the details of the current Nix implementation, there might or
might not be a need for the client to also support it (which would require the
daemon and the client to be updated in synchronously)

## Old Nix versions and caching

What happens (and should happen) if a Nix not supporting the cas model queries
a cache with cas paths in it is not clear yet.

## Garbage collection

Another major open issue is garbage collection of the aliases table. It's not
clear when entries should be deleted. The paths in the domain are "fake" so we
can't use them for expiration. The paths in the codomain could be used (i.e. if
a path is GC'ed, we delete the alias entries that map to it) but it's not clear
whether that's desirable since you may want to bring back the path via
substitution in the future.

## Ensuring that no temporary output path leaks in the result

One possible issue with the ca model is that the output paths get moved after being built, which breaks self-references. Hash rewriting solves this in most cases, but it is only heuristic and there is no way to truly ensure that we don't leak a self-reference (for example if a self-reference appears in a zipped file − like it's often the case for man pages or java jars, the hash-rewriting machinery won't detect it).
Having leaking self-references is annoying since

- These self-references change each time the inputs of the derivation change, making ca useless (because the output will _always_ change when the input change)
- More annoyingly, these references become dangling and can cause runtime failures

We however have a way to dectect these: If we have leaking self-references then the output will change if we artificially change its output path. This could be integrated in the `--check` option of `nix-store`.

# Future work

[future]: #future-work

This RFC tries as much as possible to provide a solid foundation for building
ca paths with Nix, leaving as much room as possible for future extensions.
In particular:

- Add some path-rewriting to allow derivations with self-references to be built
  as ca
- Consolidate the caching model to allow non-deterministic derivations to be
  built as ca
- (hopefully, one day) make the CA model the default one in Nix
- Investigate the consequences in term of privileges requirements
- Build a trust model on top of the content-adressed model to share store paths

[rfc 0017]: https://github.com/NixOS/rfcs/pull/17
[nixphd]: https://nixos.org/~eelco/pubs/phd-thesis.pdf
