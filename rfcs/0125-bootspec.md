---
feature: bootspec
start-date: 2022-05-09
author: Graham Christensen
co-authors: Cole Helbling
shepherd-team: "@lheckemann, @06kellyjac, @GovanifY, @RaitoBezarius"
shepherd-leader: "@lheckemann"
related-issues: https://github.com/NixOS/nixpkgs/pull/172237
---

# Summary
[summary]: #summary

Bootspec is a set of memoized facts about a system's closure.
These facts are used as the primary input for bootloader backends like systemd-boot and grub, for creating files in `/boot/loader/entries/` and `grub.cfg`.

In this proposal, we create a stable, comprehensive, and machine-parsable definition of a NixOS Generation as an intermediate representation (IR) between the NixOS system definition and the bootloader management tools.

This document describes Bootspec v1.

# Motivation
[motivation]: #motivation

NixOS's bootloader backends don't support a uniform collection of features and design decisions, partially due to the complexity of implementing the features.
Using a statically parsable bootspec definition reduces the work involved in implementing bootloader support.

If we survey the current set of bootloader and feature matrix, we see a bit of a smattering:

| Bootloader                  | Generation limits | Initrd Secrets | Multiple Profiles | Specialisation | Custom Labels |
| --------------------------- | ----------------- | -------------- | ----------------- | -------------- | ------------- |
| systemd-boot                | YES               | YES            | YES               | YES            | YES           |
| grub                        | YES               | YES            | YES               | YES            | YES           |
| generic-extlinux-compatible | YES               |                |                   |                | YES           |
| raspberrypi                 | YES               |                |                   |                |               |
| init-script                 |                   |                |                   | YES            |               |
| generations-dir             |                   |                |                   |                |               |

One reason the matrix is not filled out is the technical difficulty of implementing these various features.
The current API for detecting most of this information is by globbing directories looking for specific files.

By forcing implementations to spelunk the filesystem, we create a complicated problem in multiple ways:

- Introducing or improving bootloader features is complicated and requires significant effort to implement it for the major bootloaders.
- The relatively complicated filesystem-based behavior involves reimplementing similar logic across Perl, Python, and Bash.
- Our current tools can only support limited workflows, and are difficult to extend.

## Supporting Externalized Bootloader Backends
[externalized-bootloaders]: #externalized-bootloaders

Our NixOS-provided tooling is sufficient for most use cases; however, there are more complicated cases that are not adequately covered by our tools and would not be appropriate to include in NixOS directly.
Let's use SecureBoot as an example.

A naive, single-user implementation of SecureBoot may have signing keys on the local filesystem, and every `nixos-rebuild boot` call signs the relevant files.
This is a valid method of implementation, but is insufficient for larger deployments.

An enterprise deployment of SecureBoot may have a centralized signing service with careful auditing about what is signed.
It is more likely that the signed bootloader files come prebuilt and presigned from upstream and an unsigned file means a violation of policy.
This is also a valid implementation, but is too heavy for a single user.

There are infinitely many policy choices and implementations possible, and a good solution here is to allow deployers of NixOS to implement their own bootloader management tools.
By creating a well defined specification of generations and the boot-relevant data we enable this external development.

## What about systemd's Bootloader Specification?
[systemd-bootloader-specification]: #systemd-bootloader-specification

Systemd's bootloader specification is a good format for a different problem.
A single NixOS generation can contain multiple bootable systems and options, with additional features unique to NixOS built on top.
Most Linux distributions don't deal with many unique and ever-changing bootables.
This proposal is specifically to deal with the collection of bootables and improve our ability to treat the collection consistently.

# Goals
[goals]: #goals

- Enable a more uniform bootloader feature support across our packaged bootloaders.
  Concretely, making it relatively straightforward to convert most of the empty spaces in the feature matrix to YES's.
- Enable users of NixOS to implement custom bootloader tools and policy without needing to dive through the system profiles, and without patching Nixpkgs / NixOS.
- Define a stable specification of a generation's boot data which internal and external users can rely on.
  Changes to the specification should go through an RFC.

### Non-Goals
[non-goals]: #non-goals

- Rewriting the existing bootloader backends to actually fill out the feature matrix.
  The goal of this RFC is to make the feature development *easier*, not actually do it.
- Supporting SecureBoot.
  The authors of this RFC have done work in this regard, but this RFC is not about SecureBoot.
- Integrating Bootspec into an existing bootloader backend is not expected to perfectly produce the exact same result. For example, it is likely that the text describing the entries in the menu may differ.
- Specifying how to discover generations. This is desirable, but should not be tied to bootspec directly since bootspec may be useful with diverse discovery mechanisms.

# Proposed Solution
[proposed-solution]: #proposed-solution

- Each NixOS generation will have a bootspec (a JSON document) at `$out/boot.json` containing all of the boot properties for that generation.
  NixOS's bootloader backends will read these files as inputs to the bootloader installation phase.
  A v1 bootspec document must have a key `"org.nixos.bootspec.v1"` that describes a bootable NixOS configuration.
    - The bootloader installation phase is relatively unchanged from the way it is now.
      The bootloader backend will have an executable that is run against a collection of generations, and the backend is any of the currently supported backends plus an "external" backend which the user can define.
- The bootloader backends will avoid reading data from the other files and directories when possible, preferring the information in the bootspec.
- A bootspec synthesizing tool will be used to synthesize a bootspec for generations which don't already have one.
  This tool will be shared across all of the bootloader backends, helping produce more uniform behavior.
- Existing bootloader backends will be updated to read properties from the bootspec, removing most if not all of their filesystem-spelunking code.
- The file paths referred to in the bootspec document should be references to Nix store paths, so that the presence of the bootspec document in the store (assuming no breakage of the Nix store invariants through modifications outside Nix or bugs in Nix) guarantees the presence of the files required for boot.
  Bootspec documents should not be copied out of the store to avoid breaking this invariant.

### Bootspec Format v1
[format-v1]: #format-v1


Using the following JSON:

```json5
{
  // Toplevel key describing the version of the specification used in the document
  "org.nixos.bootspec.v1": {
    // (Required) System type the bootspec is intended for (e.g. `x86_64-linux`, `aarch64-linux`)
    "system": "x86_64-linux",

    // (Required) Path to the stage-2 init, executed by the initrd (if present)
    "init": "/nix/store/xxx-nixos-system-xxx/init",

    // (Optional) Path to the initrd
    "initrd": "/nix/store/xxx-initrd-linux/initrd",

    // (Optional) Path to a tool to dynamically add secrets to an initrd.
    // Consumers of a bootspec document should copy the file referenced by the "initrd" key to a writable location, ensure that the file is writable, invoke this tool with the path to the initrd as its only argument, and use the initrd as modified by the tool for booting.
    // This may be used to add files from outside the Nix store to the initrd.
    // This tool is expected to run on the system whose boot specification is being set up, and may thus fail if used on a system where the expected stateful files are not in place or whose CPU does not support the instruction set of the system to be booted.
    // If this field is present and the tool fails, no boot configuration should be generated for the system.
    "initrdSecrets": "/nix/store/xxx-append-secrets/bin/append-initrd-secrets",

    // (Required) Path to the kernel image
    "kernel": "/nix/store/xxx-linux/bzImage",

    // (Required) Kernel commandline options
    "kernelParams": [
      "amd_iommu=on",
      "amd_iommu=pt",
      "iommu=pt",
      "kvm.ignore_msrs=1",
      "kvm.report_ignored_msrs=0",
      "udev.log_priority=3",
      "systemd.unified_cgroup_hierarchy=1",
      "loglevel=4"
    ],

    // (Required) The label of the system. It should contain the operating system, kernel version,
    // and other user-relevant information to identify the system. This corresponds
    // loosely to `config.system.nixos.label`.
    "label": "NixOS 21.11.20210810.dirty (Linux 5.15.30)",

    // (Required) Top level path of the closure, in case some spelunking is required
    "toplevel": "/nix/store/xxx-nixos-system-xxx",
  },
  // The top-level object may contain arbitrary further keys ("extensions"), whose semantics may be defined by third parties.
  // The use of reverse-domain-name namespacing is recommended in order to avoid name collisions.

  // (Optional) Specialisations are an extension to the specification which allows bundling multiple variants of a NixOS configuration with a single parent.
  // These are shaped like the top level; to be precise:
  //  - Each entry in the toplevel "org.nixos.specialisation.v1" object represents a specialisation.
  //  - In order for the top-level document to be a valid v1 bootspec, each specialisation must have a valid "org.nixos.bootspec.v1" key whose value conforms to the same schema as the toplevel "org.nixos.bootspec.v1" object.
  //  - The behaviour of nested specialisations (i.e. entries in "org.nixos.specialisation.v1" which themselves contain the "org.nixos.specialisation.v1" key) is not defined.
  //  - In particular, there is no expectation that such nested specialisations will be handled by consumers of bootspec documents.
  //  - Each specialisation document may contain arbitrary further keys (extensions), like the top-level document.
  //  - The semantics of these should be the same as when these keys are used at the top level, but only apply for the given specialisation.
  "org.nixos.specialisation.v1": {
    // Each key in this object corresponds to a specialisation as defined by the `specialisation.<name>` NixOS option.
    "<name>": {
      "org.nixos.bootspec.v1": {
        // See above
      }
    }
  },
}
```

An *optional* field means: a field that is either missing or present, but **never `null`**.

### Risks
[risks]: #risks

- Some of the bootloader backends are quite complicated, and in many cases have inadequate tests.
  We could accidentally break corner cases.
- The bootloader backends are inherently a weak point for NixOS, as it is our last option for rolling back.
  We cannot roll back a broken bootloader.
  This and the previous point are risks, but also help demonstrate the value of reducing the amount of code and complexity in the generator.

### Milestones
[milestones]: #milestones

- Create and package the backwards compatibility synthesizer as a standalone tool.
  A version of it already exists, but it is not standalone.
- Generate the bootspec files as part of building the system closure.
- Update the bootloader backends to use bootspec as their primary source of installation data.
- Implement a NixOS module which supports external bootloader tooling.

# FAQ
[faq]: #faq

**Does this involve patching any of the bootloaders?**

No: this work is all about creating a cleaner interface for the tools we maintain which generate the files and configuration for our supported bootloaders.

**Why can't we solve this with the module system?**

Boot-menu generation is by definition cross-generation.
Bootloader backends, by definition reconfigure the boot partition and records, and are supposed to apply retroactively to old configurations.
This regeneration happens way beyond the lifecycle of the module system.

**How is this easier than `if (-f "$out/append-initrd-secrets")`?**

The main problem with that approach is there are a lot of files that you need to look for and examine in specific ways.
We end up duplicating a lot of work across bootloader backends to "learn" about the system.
By creating a source of truth for this information the bootloader backends can focus more on the work that is unique to their problem.

Doing this work now may also make it easier to refactor a NixOS system's `$out` , reducing the number of symlinks and "noise" if we wanted to do that.

**Does this interrupt the work to make stage-1 based on systemd?**

No.
Stage-1 won't use this document at all.

**What about memtest and other boot options?**

That should be left to configuration passed to the bootloader backend.
There could be any number of "extra" bootables like this, and importantly they have nothing to do with a specific NixOS system closure.
Concretely, a user who enables memtest probably wants the most recent memtest, and not one memtest from every generation.

# Open Questions
[open-questions]: #open-questions

# Future Work
[future]: #future-work

- Completing the migration from filesystem-spelunking into using the bootspec data.
- Implementing a NixOS module for supporting externalized bootloader backends.
- Implementing a base level of SecureBoot support.
- Addressing the known weaknesses of this specification (no support for multiple initrds, janky initrd secrets mechanism, lack of device tree support) in a later version of the spec.
