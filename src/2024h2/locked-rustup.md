# Locked Rustup

| Metadata |                               |
| -------- | ----------------------------- |
| Owner(s) | [@rami3l] (co-owners desired) |
| Teams    | [t-rustup]                    |
| Status   | WIP                           |

[@rami3l]: https://github.com/rami3l
[t-rustup]: https://www.rust-lang.org/governance/teams/dev-tools#team-rustup

## Summary

Explore different lock-based mechanisms to make it safe for multiple rustup instances
to execute simultaneously.

## Motivation

<!-- _Begin with a few sentences summarizing the problem you are attacking and why it is important._ -->

In rustup's list of open issues, [rustup#988] seems to a longstanding one.
The problem it describes is very simple: rustup as it currently stands has no defense mechanisms
against concurrent runs, and this might cause broken installations when concurrent writes do happen,
or when some reader accidentally gets a view of the transient state of a Rust installation.

[rustup#988]: https://github.com/rust-lang/rustup/issues/988

### The status quo

<!-- _Elaborate in more detail about the problem you are trying to solve. This section is making the case for why this particular problem is worth prioritizing with project bandwidth. A strong status quo section will (a) identify the target audience and (b) give specifics about the problems they are facing today. Sometimes it may be useful to start sketching out how you think those problems will be addressed by your change, as well, though it's not necessary._ -->

Rustup may be invoked in different "mode"s below:

- As `rustup-init`, to install rustup, and by default a toolchain;
- As `rustup`, to explicitly query or modify an installation, including rustup itself and one or more toolchains;
- As a proxy of `rustc`, `cargo`, etc., which can implicitly trigger installation, upgrade of modification of a toolchain (e.g. through toolchain files) [^implicit-install].
  This is also the "secret sauce" that makes rustup what it is today, but without much public awareness.

[^implicit-install]:
    The removal of implicit installation is being tracked in [rustup#3635],
    however the risks of [rustup#988] is still present nonetheless.

[rustup#3635]: https://github.com/rust-lang/rustup/issues/3635

Due to this polyvalent nature, [rustup#988] has turned out to be a recurring issue in multiple cases,
and it is particularly harmful for Rust's IDE experience:
if an IDE is currently running, it might implicitly hold a long-running `rust-analyzer` instance
(possibly via a rustup proxy) while being capable to automatically run `rustup` commands
(proxies included) in order to simplify the programmer's workflow.
However, if modifications are performed to the current Rust installation at this time,
the abstractions rustup provides might suddenly break down with a race condition.

Once this has caused a faulty Rust installation, one will have to reinstall the affected component
or even the entire toolchain.

This problem has been identified in many different cases with various error messages, such as:

- `error: "C:\Users\runneradmin\.cargo\bin\rustup.exe" is not a valid subcommand[..]` ([rustup#3709])
- `error: failed to install component: 'rust-src', detected conflict: 'lib/rustlib/src/rust/Cargo.lock'` ([rustup#3716])
- `error: the 'cargo' binary, normally provided by the 'cargo' component, is not applicable to the '1.78.0-x86_64-unknown-linux-gnu' toolchain` ([rust-clippy#12763])

[rustup#3709]: https://github.com/rust-lang/rustup/issues/3709
[rustup#3716]: https://github.com/rust-lang/rustup/issues/3716
[rust-clippy#12763]: https://github.com/rust-lang/rust-clippy/issues/12763

... and this often leaves the user confused.

### The next 6 months

<!-- _Sketch out the specific things you are trying to achieve in this goal period. This should be short and high-level -- we don't want to see the design!_ -->

The plan for the next 6 months would be the following:

- Investigate prior art regarding this topic, including:
  - [`cargo`'s file locking mechanism](https://github.com/rust-lang/cargo/blob/0d67af02c9b0e7b14fc7b24ce54dcec1d5ebead7/src/cargo/util/flock.rs)
  - [`gitoxide`'s file locking mechanism](https://github.com/Byron/gitoxide/tree/caae9260ef3d66998d6826c493631f3d7296c73f/gix-lock)
- Design the appropriate locking mechanism.
- Make an MVP-level implementation.

### The "shiny future" we are working towards

<!-- _If this goal is part of a larger plan that will extend beyond this goal period, sketch out the goal you are working towards. It may be worth adding some text about why these particular goals were chosen as the next logical step to focus on._ -->

<!-- _This text is NORMATIVE, in the sense that teams should review this and make sure they are aligned. If not, then the shiny future should be moved to frequently asked questions with a title like "what might we do next"._ -->

The possible outcome of this project would be the following:

- Ship the new locking mechanism after a certain period of testing,
  first internally and then externally, preferably within the v1.29 or v1.30 release cycle.
- Further optimize the mechanism in order to simplify the workflow for our users.

## Design axioms

<!-- _This section is optional, but including [design axioms][da] can help you signal how you intend to balance constraints and tradeoffs (e.g., "prefer ease of use over performance" or vice versa). Teams should review the axioms and make sure they agree. [Read more about design axioms][da]._ -->

- The new locking mechanism should play well with rustup's
  potentially recursive execution pattern [^rec], meaning it will be re-entrant if necessary.
- Prioritize the most common use cases first.
  For example, NFS should not be a blocker to our lock-based approach even when
  NFS is known for not playing with locks very well on bad connections.
- Prioritize correctness over convenience if possible.
  For example, the critical section in our solution might be considerably large to begin with,
  and to be refined over time.

[^rec]:
    For example, a `cargo` proxy might call `cargo +toolchain` deep down, and not handling this
    properly might cause the error described in
    [rustup#3031 (comment)](https://github.com/rust-lang/rustup/issues/3031#issuecomment-1182253988).

[da]: ../about/design_axioms.md

## Ownership and other resources

**Owner:** [@rami3l] (co-owners desired)

<!-- This section defines the specific work items that are planned and who is expected to do them. It should also include what will be needed from Rust teams. -->

<!-- - Subgoal: -->
<!--   - Describe the work to be done and use `↳` to mark "subitems". -->
<!-- - Owner(s) or team(s): -->
<!--   - List the owner for this item (who will do the work) or ![Help wanted][] if an owner is needed. -->
<!--   - If the item is a "team ask" (i.e., approve an RFC), put ![Team][] and the team name(s). -->
<!-- - Status: -->
<!--   - List ![Help wanted][] if there is an owner but they need support, for example funding. -->
<!--   - Other needs (e.g., complete, in FCP, etc) are also fine. -->

_Adjust the table below; some common examples are shown below._

| Subgoal                                 | Owner(s) or team(s)  | Status           |
| --------------------------------------- | -------------------- | ---------------- |
| Locked Rustup                           |                      |                  |
| ↳ investigation                         | [@rami3l]            | ![Help wanted][] |
| ↳ implementation                        | [@rami3l]            | ![Help wanted][] |
| ↳ standard reviews                      | ![Team][] [t-rustup] |                  |
| Inside Rust blog post inviting feedback | ![Team][] [t-rustup] |                  |

[Help wanted]: https://img.shields.io/badge/Help%20wanted-yellow
[Complete]: https://img.shields.io/badge/Complete-green
[TBD]: https://img.shields.io/badge/TBD-red
[Team]: https://img.shields.io/badge/Team%20ask-red
[Compiler]: https://www.rust-lang.org/governance/teams/compiler
[Lang]: https://www.rust-lang.org/governance/teams/lang
[LC]: https://www.rust-lang.org/governance/teams/leadership-council
[Libs-API]: https://www.rust-lang.org/governance/teams/library#team-libs-api
[Infra]: https://www.rust-lang.org/governance/teams/infra
[Cargo]: https://www.rust-lang.org/governance/teams/dev-tools#team-cargo
[Types]: https://www.rust-lang.org/governance/teams/compiler#team-types

<!-- ## Frequently asked questions -->

<!-- ### What do I do with this space? -->

<!-- _This is a good place to elaborate on your reasoning above -- for example, why did you put the design axioms in the order that you did? It's also a good place to put the answers to any questions that come up during discussion. The expectation is that this FAQ section will grow as the goal is discussed and eventually should contain a complete summary of the points raised along the way._ -->
