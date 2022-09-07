---
layout: post
caption: Towards Stable Rust UEFI Firmware
categories: [fedora]
tags: [efi, firmware, rust, stable, tier-2, tier-3, uefi]
hidden: true
---
While _Tianocore EDKII_ still dominates the UEFI development world, there has
been continuous effort to enable Rust for firmware development. But so far the
tools involved have not been stabilised. We now started an effort to remedy
this and get stable Rust support for UEFI targets.

The rust compiler has gained support for multiple UEFI targets in the past,
namely:

 * `aarch64-unknown-uefi` [@**Tier-3**](https://doc.rust-lang.org/nightly/rustc/platform-support/unknown-uefi.html)
 * `i686-unknown-uefi` [@**Tier-3**](https://doc.rust-lang.org/nightly/rustc/platform-support/unknown-uefi.html)
 * `x86_64-unknown-uefi` [@**Tier-3**](https://doc.rust-lang.org/nightly/rustc/platform-support/unknown-uefi.html)

This allows building Rust UEFI Applications with a standard compiler by simply
passing `--target <arch>-unknown-uefi` to _cargo_ or _rustc_. Unfortunately,
[_Tier-3_](https://doc.rust-lang.org/nightly/rustc/target-tier-policy.html#tier-3-target-policy)
support means no compiler builds are distributed via the Rust release
channels, nor does the Rust-CI guarantee the targets build successfully.
Moreover, this implies that a nightly/unstable compiler is required to build
for those targets, even though no nightly Rust Language features are required.

Raising support of these targets to
[_Tier-2_](https://doc.rust-lang.org/nightly/rustc/target-tier-policy.html#tier-2-target-policy)
will include automatic toolchain builds distributed via Rust release channels.
Hence, no nightly/unstable compiler is required, anymore. Automatic CI builds
will guarantee the targets build successfully and do not randomly break. This
will greatly improve the trust in the platform and significantly enhance the
developer experience.

Rust support for UEFI has been documented in the _rustc-book_ section
[UEFI Platform Support](https://doc.rust-lang.org/nightly/rustc/platform-support/unknown-uefi.html).
You can follow and support the _Major Change Proposal_ (MCP) to raise support
to _Tier-2_ on the
[Rust Compiler-Team Tracker](https://github.com/rust-lang/compiler-team/issues/555).
