# Paris Traceroute for Linux

**Requirements Specification Document (RSD)**

**Status:**  In progress  
**Initial version:** 2026-06-11  
**Author:** Timur Friedman  
**Reviewed by:**  
**Approved by:**  

## Document History

| Version | Date | Author | Description |
| --- | --- | --- | --- |
| 0.1 | 2026-06-11 | Timur Friedman | Initial draft |
| 0.2 | 2026-06-15 | Timur Friedman | Revised draft |
| 0.3 | 2026-06-17 | Timur Friedman | Added [MPTH-07-PR](#MPTH-07-PR) and [MPTH-08-PR](#MPTH-08-PR), and revised the definition of the utility |
| 0.4 | 2026-06-25 | Timur Friedman | Minor corrections |
| 0.5 | 2026-06-25 | Timur Friedman | Markdown edition |

## Purpose

This document specifies the requirements for Paris Traceroute for Linux.

The following is the definitive statement of what the utility is. It is the single source of truth for the utility's definition: the requirements below, and the functional and design specifications that derive from them, elaborate it and must remain consistent with it.

> **Paris Traceroute for Linux** is an IP path tracing utility for Linux distributions that is capable of accurately tracing both individual paths and all concurrent paths in the presence of load balancing. It serves as a backward-compatible, drop-in replacement for the current standard, Traceroute for Linux.

The current default traceroute tool for most Linux distributions is Dmitry Butskoy's Traceroute for Linux &#91;[Butskoy 2007](#ref-butskoy-2007)&#93;, which has shipped as the standard since 2007. It fails, however, in the face of equal-cost multi-path routing (ECMP), a form of load balancing widely deployed across the internet, producing incomplete or misleading path information.

Paris Traceroute, first developed in 2006 &#91;[Augustin et al. 2006](#ref-augustin-2006)&#93;, addresses this limitation in two ways: it correctly traces an individual path through a load-balanced routing topology, and, through its Multipath Detection Algorithm (MDA) &#91;[Veitch et al. 2009](#ref-veitch-2009), building on [Augustin, Friedman and Teixeira 2007](#ref-augustin-2007)&#93;, it correctly identifies and traces multiple concurrent paths.

Despite its adoption as the standard tool in the internet measurement research community &#91;[ACM SIGCOMM 2022](#ref-sigcomm-2022)&#93;, Paris Traceroute remains largely inaccessible to network operators and the general public. Paris Traceroute for Linux is intended to close that gap.

## Scope

**In scope:**

- The features the utility is to include
- The documentation and packaging that accompany it
- The manner in which it is to be distributed

**Out of scope:**

- Functional and design specifications; these follow from the requirements specification
- Ports to operating systems other than Linux
- A catalog of desired features for possible future versions of the utility

## Background

The traceroute tool, originally written by Van Jacobson in December 1988 &#91;[Jacobson 1988](#ref-jacobson-1988)&#93;, is one of the most widely used network diagnostic utilities. On modern Linux systems, the standard implementation is Traceroute for Linux, a 2007 rewrite of Jacobson's tool by Dmitry Butskoy &#91;[Butskoy 2007](#ref-butskoy-2007)&#93;. It is now the default provider of the traceroute command across the major distribution families, including Debian and Ubuntu, Fedora and Red Hat Enterprise Linux (RHEL), openSUSE, Arch Linux, and Gentoo. It is not the only traceroute-style tool available: Fedora and Arch Linux also include tracepath as part of the core iputils suite &#91;[Kuznetsov and Yoshifuji 1999](#ref-kuznetsov-1999)&#93;, installed alongside the standard ping command. Nevertheless, any environment or automation script that requires the traceroute command by name will invoke Butskoy’s implementation, which every major package manager ships under the name traceroute.

Traceroute works by sending a series of probe packets toward a destination address, limiting how far each one can travel by initializing the IP packet header’s Time-to-Live (TTL) field to a low value. As each router forwards a probe, it decrements the TTL value by one. If a router reduces the TTL to zero, it discards the packet and replies to the sender with an ICMP Time Exceeded message. The traceroute tool collects the reply and logs the message’s source IP address. By incrementally increasing the initial TTL for subsequent probes, traceroute records an IP address from each successive router along the path to the destination. To match reply messages to the probes that elicited them, traceroute encodes identifiers in header fields that the application does not otherwise need.

Jacobson’s choice of the destination port number as an identifier for UDP probes was unproblematic in 1988, as it predated the widespread deployment of load balancing on the internet. However, modern routers that enable equal-cost multi-path (ECMP) load balancing use this field to assign packets to paths, meaning that traditional traceroute probes can be scattered across several physical paths without the utility being aware of it. Consequently, the path that traceroute reports can contain anomalies such as missed IP addresses, missed links between addresses, and falsely inferred links. The Paris Traceroute paper documented this problem and presented a technique to avoid it by manipulating alternative header fields &#91;[Augustin et al. 2006](#ref-augustin-2006)&#93;. While the scientific community has wholly embraced the Paris Traceroute approach, recognized by the ACM Internet Measurement Conference's 2022 Test of Time Award &#91;[ACM SIGCOMM 2022](#ref-sigcomm-2022)&#93;, mainstream Linux distributions have not kept pace, and Traceroute for Linux retains the flawed legacy method.

Simply changing the header fields would allow traceroute to avoid single-path errors, but it would not enable it to trace all concurrent paths between a source and a destination. For this, a stochastic algorithm called the Multipath Detection Algorithm (MDA) &#91;[Veitch et al. 2009](#ref-veitch-2009), building on [Augustin, Friedman and Teixeira 2007](#ref-augustin-2007)&#93; is required. The complexities of this algorithm make a fresh implementation difficult.

Rather than attempt an MDA implementation in C, the language used for Traceroute for Linux, the Dioptra Group at the LIP6 laboratory of Sorbonne University has written one in Rust, which is more readable and guarantees memory safety. This is embodied in Voyage &#91;[Lohrer 2024](#ref-lohrer-2024)&#93;, a traceroute tool built on modular components: the caracat probing library and the pantrace format-conversion library. This document refers to these three components together as the implementation basis.

Paris Traceroute for Linux will be built by adapting Voyage and its supporting libraries. The requirements for that tool are specified in the sections that follow.

## Motivation

No tool available today satisfies all four of the following properties. Paris Traceroute for Linux will satisfy them all. (References are to the requirements specified later in this document.)

- correct tracing of a single path through a load-balanced topology, by holding the flow identifier steady ([SPTH-01-PR](#SPTH-01-PR));
- multipath tracing with the Multipath Detection Algorithm, carrying explicit statistical guarantees of completeness ([MPTH-02-PR](#MPTH-02-PR));
- command-line compatibility with Butskoy’s Traceroute for Linux ([COMP-02-PR](#COMP-02-PR));
- output compatibility with Butskoy’s Traceroute for Linux ([COMP-03-PR](#COMP-03-PR)).

Every one of these properties is found in some existing tool, but no tool has all four at once. (See Table B-4 of Appendix B for a tool-by-tool comparison.)

Compatibility with Butskoy's Traceroute for Linux matters on two levels. The first is mechanical: countless scripts invoke traceroute and countless parsers consume its output, and a replacement must not break them. The second is human, and it is the one that is crucial for adoption. A network operator turns to traceroute out of habit, typing familiar options and reading the output at a glance. They will switch to a new tool only if it is called in the same way and prints the same thing. Paris Traceroute for Linux will meet those expectations.

Traceroute for Linux has command-line and output compatibility by definition, since it is the reference, but it offers neither correctness under load balancing nor multipath tracing. The research community’s tools, among them the original paris-traceroute command-line implementation, scamper, and Dublin Traceroute, trace correctly and enumerate the load-balanced paths, but each one has to be learned from scratch. None can be invoked as traceroute or read as traceroute, by a person at the command line or by a script or parser. The traceroute of RIPE Atlas applies the Paris Traceroute correction, but it is designed for a particular measurement platform, not as a general command-line replacement for traceroute. The implementation that is closest of all as a drop-in replacement to the classic utility is Catchpoint's Pietrasanta traceroute, which extends Traceroute for Linux compatibly and steadies the flow of its TCP probes; but it does not enumerate the multiple paths, and it has not displaced the reference implementation in the distributions.

This is the gap that Paris Traceroute for Linux is built to fill. To a person at the command line, and to any script or parser, it will look exactly like the standard Linux traceroute. But correct single-path tracing will be the default, and multipath tracing will be available for anyone who asks for it.

The choice of language brings a further advantage. Every traceroute shipped with an operating system today is written in C (see Table B-1 of Appendix B); Paris Traceroute for Linux will be written in Rust ([SOFT-03-PR](#SOFT-03-PR)). A traceroute utility parses packets from untrusted networks, and some of its probing methods need elevated privileges. In C, a bug in that parsing is a possible memory-corruption vulnerability. Rust rules out that entire class of bug. Its built-in concurrency mechanism also suits multipath tracing, which sends far more probe packets in parallel than a classic traceroute does.

## Vocabulary

While the design specification will define which components make up the utility, it is not possible to define the requirements without a vocabulary that implies the existence of some entities. The terms used in this document are:

- the **utility**, which is Paris Traceroute for Linux, whose name declares its compatibility with Butskoy’s Traceroute for Linux, and whose requirements this document defines;
- the **reference implementation**, which is the existing traceroute against which backward compatibility is defined: Butskoy’s Traceroute for Linux;
- the **implementation basis**, which is the existing software on which the utility is to be built: Voyage, together with the caracat and pantrace libraries;
- the **classic mode**, which is the utility's behavior when it is invoked without any multipath option, and the **multipath mode**, which is its behavior when a multipath option is invoked;
- a **flow identifier**, which is the set of packet header fields that per-flow load balancers use to assign a packet to one among several available paths, classically the five-tuple of source address, destination address, protocol, source port, and destination port, or fields occupying equivalent locations in ICMP packets;
- a **calling system**, which is any independent program, script, or service that invokes traceroute via its command line and consumes its output, whether by parsing it programmatically or by presenting it to a human;
- the **package**, which is the deliverable unit through which the utility is distributed and installed, together with its documentation and ancillary files.

## Requirements

### Requirement Identifiers

Each requirement is identified by a code of the form XXXX-NN-PR or XXXX-NN-NR, where XXXX is a four-letter abbreviation indicating the category of the requirement, such as COMP for "backward compatibility" or MPTH for "multipath tracing", and NN is a two digit number to distinguish the requirement from others in the same category.

This document distinguishes between positive requirements and negative requirements. Negative requirements are used to make it clear that some feature that one might think would be required is, in fact, not required. Positive requirements are identified by PR and negative requirements by NR.

### Requirement Keywords

Each requirement below is given as a single set-apart statement, followed by explanatory discussion. The normative force of a requirement lies entirely in that set-apart statement; the surrounding discussion is explanatory and binds nothing. Within the requirement statements, the words below, written in capital letters, carry a precise and consistent force. The same words in lower case — including everywhere in the discussion — carry only their ordinary English meaning and impose no requirement.

- MUST and MUST NOT express an absolute requirement: something the utility is obliged to do, or obliged never to do. A deliverable that violates a MUST or a MUST NOT does not meet this specification.
- SHOULD and SHOULD NOT express a strong recommendation. The stated behavior is expected by default, but there may be legitimate reasons to depart from it in particular circumstances; any such departure should be deliberate and its implications understood.
- MAY expresses something that is permitted but not required: an option left open, with no obligation either way.

### Requirements Table

The table below indexes the requirements by category, giving the code and name of each. The full statement of each requirement, with its discussion, follows in the sections after the table.

| Category | Code & link | Name |
| --- | --- | --- |
| Deliverables | [DELV-01-PR](#DELV-01-PR) | Command-line utility |
| Deliverables | [DELV-02-PR](#DELV-02-PR) | Man page |
| Deliverables | [DELV-03-PR](#DELV-03-PR) | Source release |
| Deliverables | [DELV-04-PR](#DELV-04-PR) | Compatibility test suite |
| Deliverables | [DELV-05-NR](#DELV-05-NR) | No graphical interface |
| Naming | [NAME-01-PR](#NAME-01-PR) | Tool name |
| Software | [SOFT-01-PR](#SOFT-01-PR) | Software hosting |
| Software | [SOFT-02-PR](#SOFT-02-PR) | Software licensing |
| Software | [SOFT-03-PR](#SOFT-03-PR) | Implementation language |
| Software | [SOFT-04-PR](#SOFT-04-PR) | Legacy repositories |
| Backward Compatibility | [COMP-01-PR](#COMP-01-PR) | Reference implementation |
| Backward Compatibility | [COMP-02-PR](#COMP-02-PR) | Command-line compatibility |
| Backward Compatibility | [COMP-03-PR](#COMP-03-PR) | Output compatibility |
| Backward Compatibility | [COMP-04-PR](#COMP-04-PR) | Calling-system compatibility |
| Backward Compatibility | [COMP-05-PR](#COMP-05-PR) | Companion commands |
| Backward Compatibility | [COMP-06-PR](#COMP-06-PR) | Privilege parity |
| Backward Compatibility | [COMP-07-NR](#COMP-07-NR) | No wire-level replication |
| Single-Path Tracing Under Load Balancing | [SPTH-01-PR](#SPTH-01-PR) | Steady flow identifier in classic mode |
| Single-Path Tracing Under Load Balancing | [SPTH-02-PR](#SPTH-02-PR) | Selectable flow identifier |
| Single-Path Tracing Under Load Balancing | [SPTH-03-PR](#SPTH-03-PR) | Single path with multipath alert |
| Multipath Tracing | [MPTH-01-PR](#MPTH-01-PR) | Multipath tracing mode |
| Multipath Tracing | [MPTH-02-PR](#MPTH-02-PR) | MDA-based path enumeration |
| Multipath Tracing | [MPTH-03-PR](#MPTH-03-PR) | Idiomatic command-line extension |
| Multipath Tracing | [MPTH-04-PR](#MPTH-04-PR) | Idiomatic output extension |
| Multipath Tracing | [MPTH-05-PR](#MPTH-05-PR) | Probing method parity |
| Multipath Tracing | [MPTH-06-NR](#MPTH-06-NR) | No parser guarantee for multipath output |
| Multipath Tracing | [MPTH-07-PR](#MPTH-07-PR) | Choice of multipath algorithm |
| Multipath Tracing | [MPTH-08-PR](#MPTH-08-PR) | Multipath with per-destination load balancing alert |
| Output Formats | [OFMT-01-PR](#OFMT-01-PR) | Machine-readable output |
| Packaging & Distribution | [PACK-01-PR](#PACK-01-PR) | Source tarball |
| Packaging & Distribution | [PACK-02-PR](#PACK-02-PR) | Standard package contents |
| Packaging & Distribution | [PACK-03-PR](#PACK-03-PR) | Binary packages |
| Packaging & Distribution | [PACK-04-PR](#PACK-04-PR) | Distribution repositories |
| Packaging & Distribution | [PACK-05-PR](#PACK-05-PR) | Drop-in installability |
| Packaging & Distribution | [PACK-06-NR](#PACK-06-NR) | No repository acceptance guarantee |
| Packaging & Distribution | [PACK-07-PR](#PACK-07-PR) | Minimal dependency footprint |
| Packaging & Distribution | [PACK-08-NR](#PACK-08-NR) | No packaging beyond Linux |
| Future Proofing | [FPRO-01-PR](#FPRO-01-PR) | Allow future features |

### Deliverables

This section describes the deliverables: the work product that is expected as a result of the development of the utility. It consists of: the command-line utility itself, its man page, a source release from which the package can be built, and an automated compatibility test suite. The sections that follow this one will detail the requirements for these deliverables.

<a id="DELV-01-PR"></a>

#### DELV-01-PR — Command-line utility

> A command-line utility MUST be provided that runs on Linux.

Linux is the utility’s only platform in this version, just as it is the platform of every compatibility guarantee in this document. The restriction is deliberate, and the name of the utility declares it ([NAME-01-PR](#NAME-01-PR)). The case this document makes is a Linux case: the vacancy in the distributions, the drop-in role of [PACK-05-PR](#PACK-05-PR), and the compatibilities of the COMP category all point to Linux, and a Linux release is where the utility will have its impact.

Ports to the other operating systems are expected to follow in future versions: the BSDs and macOS share the Unix socket model, and the precedents of scamper and Trippy show the ports to be tractable, while Windows, with its raw-socket restrictions and its dependency on the Npcap driver, is a larger undertaking.

These ports belong to the catalog of desired features that the Scope section places out of scope; [FPRO-01-PR](#FPRO-01-PR) requires that nothing in the design preclude them, and the implementation basis is itself portable, building wherever the Rust toolchain and libpcap are available. Packaging for the other operating systems is correspondingly not required ([PACK-08-NR](#PACK-08-NR)).

<a id="DELV-02-PR"></a>

#### DELV-02-PR — Man page

> A man page for the utility MUST be provided. It MUST document the classic options and the multipath options together, presenting the multipath options in the same style as the rest.

The man page is to be installed in section 8 of the manual, as traceroute(8) is, and is to follow the structure and conventions of the reference implementation's man page: name, synopsis, description, a description of every option, and examples. Documenting the classic and multipath options together, in a uniform style, ensures that the man page reads as the documentation of a single coherent tool rather than of a tool with a bolted-on extra.

<a id="DELV-03-PR"></a>

#### DELV-03-PR — Source release

> A buildable source release of the utility MUST be provided.

The requirements on the form and contents of the source release are detailed under Packaging & Distribution, in particular [PACK-01-PR](#PACK-01-PR) and [PACK-02-PR](#PACK-02-PR).

<a id="DELV-04-PR"></a>

#### DELV-04-PR — Compatibility test suite

> An automated test suite that verifies the utility’s backward compatibility with the reference implementation MUST be provided.

[COMP-04-PR](#COMP-04-PR) makes substitution the acceptance criterion for backward compatibility; the test suite is the instrument by which that criterion is applied.

It is to run a corpus of invocations against both the reference implementation and the utility, and to verify that the outputs are structurally equivalent; that is, that a parser written against the reference implementation’s output extracts the same fields, with the same meanings, from both.

The composition of the corpus, which should include the representative calling systems of [COMP-04-PR](#COMP-04-PR), is left to the functional specification. The corpus should also exercise the ancillary features that shape the reference implementation’s output and that are absent from the implementation basis as it stands: name resolution of returned addresses, AS path lookups (-A), and the display of ICMP extensions (-e), since these are part of the output interface that [COMP-03-PR](#COMP-03-PR) protects.

The test suite is to be maintained and distributed with the source (see [PACK-02-PR](#PACK-02-PR)).

<a id="DELV-05-NR"></a>

#### DELV-05-NR — No graphical interface

> A graphical user interface IS NOT REQUIRED to be provided.

The utility is a command-line utility, as traceroute is. Graphical front-ends, web interfaces, and visualizations are out of scope; should one ever be wanted, it would be the subject of a separate project that builds on this utility.

### Naming

<a id="NAME-01-PR"></a>

#### NAME-01-PR — Tool name

> The utility MUST be named Paris Traceroute for Linux.

The name attaches the utility to two lineages at once.

Paris Traceroute attaches it to the research lineage from which it descends: Paris Traceroute is the name under which the steady flow identifier correction of [SPTH-01-PR](#SPTH-01-PR) was introduced &#91;[Augustin et al., 2006](#ref-augustin-2006)&#93;, and the name carries two decades of recognition in the network measurement and operations communities; recognition that a new name would forfeit.

And “for Linux” deliberately echoes the name of the reference implementation, Traceroute for Linux ([COMP-01-PR](#COMP-01-PR)): the suffix conveys the compatibility promise and the platform on which that promise is kept.

The earlier implementations bearing the Paris Traceroute name, the original paris-traceroute and the libparistraceroute library that succeeded it, come from the same research effort; the handling of their repositories is treated in [SOFT-04-PR](#SOFT-04-PR).

The name presents no conflict in the current distributions: a paris-traceroute package built from the earlier implementation existed in Debian, but it was removed from Debian testing in 2020 and is absent from the current releases, so that name is available to be reclaimed for the package, and reclaiming it is preferable to abandoning it, since whatever recognition the earlier package retains accrues to its successor.

Note that the name of the project need not be the name of the package, which is expected to remain paris-traceroute, nor of the installed command; the question of how the installed command is invoked is treated under [PACK-05-PR](#PACK-05-PR).

### Software

<a id="SOFT-01-PR"></a>

#### SOFT-01-PR — Software hosting

> The software MUST be developed in a public Git repository in the Paris Traceroute organization on GitHub.

A Paris Traceroute organization and a repository have been set up for this purpose:

https://github.com/Paris-Traceroute/Paris-Traceroute-for-Linux

Two purposes inform the choice of GitHub. The first is credibility: the repository is the utility’s public face, and an active, visible history of development and maintenance lends the project the legitimacy that adoption requires. The second is the attraction of bug reports: a traceroute tool depends on reports from networks that its developers cannot see, so the barrier to filing an issue must be as low as possible, and GitHub is where the largest population of potential reporters already holds accounts.

Hosting platforms favored elsewhere in the free-software community, such as Codeberg, SourceForge, or an institutional GitLab, were considered and set aside: whatever their virtues, each would interpose an account-creation step between a user with a bug and the filing of that bug. The repository’s issue tracker is to serve as the project’s public channel for bug reports and feature requests, and the package’s documentation should point to it.

The same purposes argue for an organization named for the utility, rather than a place under the Dioptra group or a personal account. A major tool aspires to outlive any single research group’s roster of projects; an organization bearing the utility’s own name gives the project an institutional face of its own, and gathers in one place the repositories that belong to it: the utility itself, the legacy software of [SOFT-04-PR](#SOFT-04-PR), ancillary tools, and in time the project website.

<a id="SOFT-02-PR"></a>

#### SOFT-02-PR — Software licensing

> The software MUST be licensed under the GNU General Public License, version 2 or later.

This matches the license of the reference implementation. Matching licenses removes one obstacle to the utility being packaged by, and eventually adopted within, the Linux distributions, and permits code or documentation to be borrowed from the reference implementation where that is useful.

The implementation basis, which includes Voyage and the caracat and pantrace libraries, is MIT-licensed, and MIT-licensed code may be incorporated into a work distributed under the GPL. However, the consequences of these different licenses need to be thoroughly investigated, and perhaps new licenses for the implementation basis should be considered.

<a id="SOFT-03-PR"></a>

#### SOFT-03-PR — Implementation language

> The utility’s own code MUST be written in Rust.

This makes explicit a commitment that the choice of implementation basis has already made: Voyage and the caracat and pantrace libraries on which the utility builds are written in Rust, and the utility is to be developed as their extension.

The language addresses a concern that is particular to this tool: it parses untrusted packets arriving from the network and, for some probing methods, runs with elevated privileges (see [COMP-06-PR](#COMP-06-PR)), a scenario in which memory-corruption defects might become security vulnerabilities, and against which Rust’s memory safety is a protection.

The requirement attaches to the utility’s own code only. It neither extends to the libraries that the utility links, of which libpcap is the foremost (see [PACK-07-PR](#PACK-07-PR)), nor forbids the foreign-function interfaces through which Rust reaches them.

The build consequences of Rust are treated under [PACK-01-PR](#PACK-01-PR), and the consequences for acceptance into the distributions under [PACK-04-PR](#PACK-04-PR) and [PACK-07-PR](#PACK-07-PR).

<a id="SOFT-04-PR"></a>

#### SOFT-04-PR — Legacy repositories

> The repositories of the libparistraceroute organization on GitHub MUST be incorporated into the Paris Traceroute organization as legacy projects.

The earlier implementations are hosted in the libparistraceroute organization on GitHub: the libparistraceroute C library, whose paris-traceroute command is the most recent earlier implementation; the original implementation, preserved as paris-traceroute-OLD; and fakeroute, a tool that simulates load-balanced topologies for the testing of traceroute-like programs.

Left where they are, these repositories would divide the identity that [NAME-01-PR](#NAME-01-PR) claims: a user searching GitHub for Paris Traceroute would find the older software first, with nothing to say that a successor exists. They are therefore to be transferred into the Paris Traceroute organization, with GitHub redirecting the old addresses to the new, with the two implementations archived as read-only and their READMEs revised to state that the utility of this document is their successor and to direct bug reports to its issue tracker.

Their commit and issue histories remain visible, which serves the credibility purpose of [SOFT-01-PR](#SOFT-01-PR): the organization then exhibits twenty years of continuous lineage rather than a tool sprung from nowhere. fakeroute is a different matter: it is not superseded but potentially useful, in particular to the compatibility test suite of [DELV-04-PR](#DELV-04-PR), and if found to be so may continue as a live project of the organization.

### Backward Compatibility

Backward compatibility is the central requirement of this project. It is defined at the utility's two interfaces with the outside world: the command line that it accepts, and the output that it produces. It is deliberately not defined at the level of the utility's internals or of the packets that it sends, as the negative requirements below make clear.

<a id="COMP-01-PR"></a>

#### COMP-01-PR — Reference implementation

> Backward compatibility MUST be assessed against Traceroute for Linux, version 2.1.6, by Dmitry Butskoy.

This is the traceroute implementation most widely distributed with Linux distributions, having replaced the original Van Jacobson implementation in Debian, Ubuntu, Fedora, RHEL, openSUSE, Arch Linux, Gentoo, and others. Version 2.1.6, released 2024-09-13, is the most recent release at the time of writing. If a newer version of the reference implementation is released during development, the functional specification may designate that newer version as the reference instead.

<a id="COMP-02-PR"></a>

#### COMP-02-PR — Command-line compatibility

> The utility MUST accept every command-line option and argument that the reference implementation accepts, with the same syntax and the same semantics, and MUST reject invalid invocations with the same usage errors.

This covers the short and long forms of every option, the positional host and packetlen arguments, option defaults, and the interactions among options (for example, the way that a probing method selected by -I, -T, or -U conditions the meaning of -p). It also covers the utility's response to invalid invocations: an unrecognized option or a malformed argument produces a usage error in the same manner as the reference implementation, so that calling systems that detect failures continue to detect them.

<a id="COMP-03-PR"></a>

#### COMP-03-PR — Output compatibility

> For every invocation that is valid for the reference implementation, the utility MUST produce output that any parser of the reference implementation's output parses with the same result.

By output we mean everything that a calling system can observe of a completed run: the format of what is written to standard output (the header line, the hop lines with their hop numbers, host names and IP addresses, round-trip times, asterisks for missing replies, and annotations such as !H or !N), what is written to standard error, and the exit status.

Note that this is a requirement of format, not of values. Two runs of traceroute itself do not produce byte-identical output, since round-trip times vary and routes change. The requirement is that the structure of the output be such that a parser written against the reference implementation extracts the same fields, with the same meanings, from the utility's output.

<a id="COMP-04-PR"></a>

#### COMP-04-PR — Calling-system compatibility

> Any calling system that invokes the reference implementation and parses its output MUST be able to invoke the utility in its place, without modification, and continue to function.

This requirement restates [COMP-02-PR](#COMP-02-PR) and [COMP-03-PR](#COMP-03-PR) from the perspective of the user, and it is the acceptance criterion for backward compatibility: the test of the utility is substitution. The functional specification should define a corpus of representative calling systems (shell scripts, monitoring tools, and libraries that wrap traceroute) against which substitution will be verified.

<a id="COMP-05-PR"></a>

#### COMP-05-PR — Companion commands

> The package MUST provide the same companion commands as the reference implementation's package, each with the same behavior.

The reference implementation is installed not only as the traceroute command but also as traceroute6 and as a tcptraceroute wrapper, each with its man page. A calling system that invokes one of these finds it, with the same behavior, when the utility's package is installed in place of the reference implementation's.

<a id="COMP-06-PR"></a>

#### COMP-06-PR — Privilege parity

> For each probing method, the utility MUST NOT require greater operating system privileges than the reference implementation requires for that method. In multipath mode, tracing with a given probing method SHOULD likewise require no greater privileges than classic tracing with that method; where this cannot be achieved, the multipath mode MAY require the `CAP_NET_RAW` capability, and the man page MUST document that requirement.

The reference implementation allows certain methods, notably the default UDP method, to be used by unprivileged users, while other methods require elevated privileges, such as the `CAP_NET_RAW` capability, a setuid installation, or root.

A tool that demanded root where the reference implementation does not would fail as a drop-in replacement in practice, even with its command line and output in perfect order, because calling systems run with whatever privileges they happen to have.

Note that this requirement constrains the design more than any other: unprivileged UDP tracing on Linux is achieved through kernel-mediated datagram sockets and the `MSG_ERRQUEUE` mechanism, as in the reference implementation, and is not available to an engine that only constructs raw packets, as the probing engine of the implementation basis does. Classic mode therefore requires support for kernel-socket probing that the basis does not presently provide.

<a id="COMP-07-NR"></a>

#### COMP-07-NR — No wire-level replication

> The utility IS NOT REQUIRED to reproduce the reference implementation's behavior on the wire.

Backward compatibility is defined at the command-line and output interfaces, not at the packet level. The utility may construct its probe packets differently from the reference implementation. Indeed, the flow identifier control required by [SPTH-01-PR](#SPTH-01-PR) means that in some respects it must. What is required is that the differences not be observable through the interfaces that [COMP-02-PR](#COMP-02-PR) and [COMP-03-PR](#COMP-03-PR) govern.

### Single-Path Tracing Under Load Balancing

Multipath tracing presupposes something simpler: that a single path through a load-balanced topology be traced correctly. Classic traceroute varies header fields that belong to the flow identifier, so its probes are scattered across several load-balanced paths, and the route that it reports can be an arbitrary interleaving of them.

The requirements in this category correct this defect: they make the trace of classic mode a true single path, and they give the user the means to choose which of the load-balanced paths that is. They concern the tracing of a single path only; the enumeration of the several paths is the subject of the category that follows.

<a id="SPTH-01-PR"></a>

#### SPTH-01-PR — Steady flow identifier in classic mode

> In classic mode, the utility MUST hold the flow identifier constant across all of the probes of a trace.

Per-flow load balancers assign packets to paths based on the flow identifier. Classic traceroute varies header fields that are part of the flow identifier from probe to probe (in the default UDP method, the destination port) and so its probes are scattered across several paths without it being aware. The single route that it then reports can be an erroneous interleaving of those paths: false links, missed links, and missed nodes.

Holding the flow identifier constant, which is the Paris Traceroute correction, ensures that all of the probes follow a single path and that the reported trace is a true route. The correction is established practice in measurement infrastructure: the traceroute run by the probes of RIPE Atlas applies it through their Paris ID mechanism, and every Atlas result records the Paris ID under which it was traced. In every other respect the trace proceeds exactly as in the reference implementation: by default, three probes per hop.

This changes what the utility puts on the wire relative to the reference implementation, which [COMP-07-NR](#COMP-07-NR) expressly permits, and it is invisible at the output interface, since the output format carries no trace of how flow identifiers were chosen.

How the utility matches replies to its probes without varying the flow identifier (the reference implementation relies on the varying destination port for this) is left to the design specification.

<a id="SPTH-02-PR"></a>

#### SPTH-02-PR — Selectable flow identifier

> The utility MUST provide a new command-line option by which the user sets the flow identifier to a value other than the default. The output produced under this option MAY differ from the classic output format of [COMP-03-PR](#COMP-03-PR), but SHOULD adhere to it as closely as possible.

With a steady default flow identifier, every run of the utility in classic mode follows the same load-balanced path, so long as the network's configuration holds. This option gives the user manual access to the other paths: successive invocations with different flow identifier values trace different load-balanced paths, one path per run, without invoking the multipath mode.

The form of the option's argument, an opaque integer that the utility maps onto header fields, or something more explicit, is left to the functional specification, and the option is subject to the idiomatic conventions of [MPTH-03-PR](#MPTH-03-PR). The RIPE Atlas Paris ID mechanism is an example of how this might be done.

Because a selectable flow identifier is an extension of the reference implementation, its output need not match the classic format exactly; adhering to it as closely as possible lets a parser of classic output parse this output with minimal modifications.

<a id="SPTH-03-PR"></a>

#### SPTH-03-PR — Single path with multipath alert

> The utility MUST provide an option of single path probing that alerts to the presence of multiple paths.

When the multipath alert option is selected, the output consists of a single path, but one that is annotated to alert the user to the presence of multiple paths. Multipath probing is required in order to enable this, but the output is still a single path output.

### Multipath Tracing

The distinguishing new capability of the utility is multipath tracing: the discovery of the multiple load-balanced paths that packets may take between the source and the destination, rather than the single path that classic mode, with the steady flow identifier of [SPTH-01-PR](#SPTH-01-PR), reports.

The requirements in this category mandate this capability and the manner of its integration, which is to feel to a traceroute user like a natural extension of the familiar tool.

<a id="MPTH-01-PR"></a>

#### MPTH-01-PR — Multipath tracing mode

> The utility MUST provide a multipath mode, selected by one or more new command-line options, that discovers the load-balanced paths between the source and the destination.

When no multipath option is given, the utility runs in classic mode, in which the compatibility requirements of the previous section apply in full. The multipath mode is strictly opt-in: its existence must not be observable in classic mode.

<a id="MPTH-02-PR"></a>

#### MPTH-02-PR — MDA-based path enumeration

> The multipath mode MUST enumerate load-balanced paths using the Multipath Detection Algorithm (MDA) and that algorithm’s explicit statistical guarantees on the completeness of its discovery.

The Multipath Detection Algorithm (MDA) sends, at each path divergence point, a sufficient number of probe packets with distinct flow identifiers to bound, at a stated confidence level, the probability that a next-hop interface has gone undiscovered.

<a id="MPTH-03-PR"></a>

#### MPTH-03-PR — Idiomatic command-line extension

> The multipath options MUST follow the command-line conventions of the reference implementation.

The new options must look and behave like traceroute options: a single-letter short form where one is available, a corresponding long form, the same argument syntax as existing options, and sensible defaults that allow a bare invocation of the multipath mode with no further parameters.

A user who knows traceroute should be able to read the synopsis line of the man page and use the multipath mode without feeling that they have switched tools.

The specific option letters and names are left to the functional specification.

<a id="MPTH-04-PR"></a>

#### MPTH-04-PR — Idiomatic output extension

> The output of the multipath mode MUST read as a natural extension of the classic traceroute output.

The multipath output should build on the formats and habits that traceroute users already have: hop-numbered lines, host names with addresses in parentheses, round-trip times, asterisks for missing replies, while adding what is genuinely new, namely the structure of divergences and convergences among paths.

A user looking at multipath output should recognize it immediately as traceroute output, enriched.

The precise format is left to the functional specification.

<a id="MPTH-05-PR"></a>

#### MPTH-05-PR — Probing method parity

> The multipath mode MUST support the UDP, ICMP, TCP, and DCCP probing methods, for both IPv4 and IPv6, and, in general, every probing method that the reference implementation supports. Where a probing method affords no way to vary the flow identifier, the man page MUST document the resulting limitation.

The reference implementation offers UDP probing in several variants, ICMP ECHO probing, TCP probing, and further methods such as DCCP and generic IP datagrams.

A user of any of these methods should be able to add the multipath option and obtain multipath results, for IPv4 and IPv6 destinations alike. UDP, ICMP, and TCP are the methods in wide use, and TCP support is in any case implied by the tcptraceroute companion command of [COMP-05-PR](#COMP-05-PR); multipath options for these are a must.

At the time of this writing, the probing engine of the implementation basis supports only UDP and ICMP, so each further method, TCP included, will be a new development.

The methods differ in the degrees of freedom that their headers give for varying the flow identifier: UDP, TCP, and DCCP offer port numbers; ICMP offers fields such as the checksum and the sequence number; generic IP datagrams may offer nothing at all. Where a method gives limited or no ability to vary the flow identifier, multipath mode may not be possible for it.

<a id="MPTH-06-NR"></a>

#### MPTH-06-NR — No parser guarantee for multipath output

> Existing parsers of traceroute output ARE NOT REQUIRED to be able to parse the multipath mode's output.

The multipath output is new, and a calling system written against the reference implementation cannot be expected to understand it. The compatibility guarantee of [COMP-03-PR](#COMP-03-PR) and [COMP-04-PR](#COMP-04-PR) applies to invocations that are valid for the reference implementation; an invocation that uses a multipath option is by definition not among them.

<a id="MPTH-07-PR"></a>

#### MPTH-07-PR — Choice of multipath algorithm

> The utility MUST permit a choice among multipath enumeration algorithms, the Multipath Detection Algorithm (MDA) being one of them; the MDA SHOULD be the default.

The Multipath Detection Algorithm (MDA) was the first algorithm to enumerate the concurrent paths through a load balanced routing topology, but there are others, and new ones might be designed, so the utility anticipates a choice among them.

<a id="MPTH-08-PR"></a>

#### MPTH-08-PR — Multipath with per-destination load balancing alert

> The utility SHOULD provide an option of multipath probing that alerts to the presence of per-destination load balancing.

When the per-destination alert option is selected, the output consists of a multipath trace towards a single destination, but one that is annotated to alert the user to the presence of per-destination load balancing.

### Output Formats

<a id="OFMT-01-PR"></a>

#### OFMT-01-PR — Machine-readable output

> Machine-readable output formats SHOULD be provided, selectable by option, alongside the classic text output.

Machine-readable output could have been deferred to a future version (see [FPRO-01-PR](#FPRO-01-PR)); it is instead part of the present set of requirements because the implementation basis provides it at little cost: the pantrace library, on which the basis builds, already converts among the standard traceroute interchange formats (RIPE Atlas, Iris, and Scamper warts, among others), and a JSON rendering serves calling systems that prefer structured data.

Among these, the RIPE Atlas format merits particular mention: the volume of measurements that the Atlas platform produces has made its JSON format the one in which many researchers expect traceroute data, and offering it makes the utility’s results directly consumable by the analysis pipelines those researchers already operate.

The formats to be offered, and the options by which they are selected, are left to the functional specification. Machine-readable output is strictly opt-in: when it is not requested, the output interface of [COMP-03-PR](#COMP-03-PR) applies unchanged, and existing parsers are unaffected.

### Packaging & Distribution

The utility is to be made available in the most classic manner for Linux packages. For a utility of this kind, that manner is well established, and it is the manner in which the reference implementation itself is distributed: a versioned source tarball, released publicly, that builds and installs with the conventional tools, complemented by binary packages for the major package managers and, ultimately, by inclusion in the distributions' own repositories.

<a id="PACK-01-PR"></a>

#### PACK-01-PR — Source tarball

> Each release of the utility MUST be published as a versioned source tarball that builds and installs by the conventional procedure. The tarball MUST build without network access, with the sources of all dependencies included. Version numbers SHOULD follow the usual major.minor.patch convention, and the release SHOULD declare a minimum supported version of the Rust toolchain.

By the conventional procedure we mean that a user who downloads and unpacks the tarball can build and install the utility with make and make install, optionally preceded by a configure step.

The reference implementation distributes itself in exactly this way (traceroute-2.1.6.tar.gz, built with a plain make).

Because the implementation basis is in Rust, whose build tool fetches dependency sources from the network by default, the make targets may wrap the cargo build, and building without network access requires the sources of all dependencies to be incorporated into the tarball.

<a id="PACK-02-PR"></a>

#### PACK-02-PR — Standard package contents

> The source release MUST include the components that are typical of a Linux package.

These are: the license text (a COPYING file), a README, a changelog or NEWS file recording what changed in each release, the build files, the man page sources, the compatibility test suite of [DELV-04-PR](#DELV-04-PR), and the sources of the utility itself. Anyone familiar with unpacking Linux source packages should find what they expect to find.

<a id="PACK-03-PR"></a>

#### PACK-03-PR — Binary packages

> Binary packages in the .deb and .rpm formats SHOULD be provided for each release.

These are the package formats of the Debian and Red Hat families respectively, which between them cover the large majority of Linux installations. Providing them allows installation through the standard package managers without a build step. The packaging metadata (a debian directory, an RPM spec file) should be maintained alongside the source so that the distributions can reuse it.

<a id="PACK-04-PR"></a>

#### PACK-04-PR — Distribution repositories

> Inclusion of the package in the repositories of the major Linux distributions SHOULD be pursued.

The truly classic channel for a Linux utility is the distribution repository: the way a user obtains the reference implementation is apt install traceroute or dnf install traceroute, from their distribution, not from the project's own site. Acceptance into Debian and Fedora, from which derivative distributions inherit, is the goal.

This is stated as should rather than must because it depends on the decisions of outside parties; see [PACK-06-NR](#PACK-06-NR). Acceptance is also conditioned by the distributions’ policies for packaging Rust software: Debian, for example, builds Rust programs against separately packaged crates rather than vendored sources, so every dependency that is not already packaged is an obstacle. The dependency footprint of the utility therefore bears directly on this requirement; see [PACK-07-PR](#PACK-07-PR).

<a id="PACK-05-PR"></a>

#### PACK-05-PR — Drop-in installability

> The package MUST allow a system administrator to arrange that invoking traceroute invokes the utility.

Backward compatibility is only useful if the utility can actually stand in for the reference implementation, which calling systems invoke under the name traceroute.

The standard means of bringing this about is through the alternatives mechanism, which the Debian family provides as update-alternatives and the Red Hat family as alternatives.

Under this mechanism, /usr/bin/traceroute is not a binary but a managed symbolic link: each package that can play the role of traceroute registers its own binary as a candidate with a priority, the highest priority wins by default, and the administrator chooses among the candidates with a single, reversible command (update-alternatives --config traceroute), all without corrupting or conflicting with the distribution’s own traceroute package, whose binary remains on disk.

Debian already arbitrates between Traceroute for Linux and inetutils-traceroute in this way, and defines a traceroute virtual package so that other packages may depend on the role rather than on an implementation. The utility’s package registers simply as a further candidate.

The companion commands of [COMP-05-PR](#COMP-05-PR), traceroute6 and the tcptraceroute wrapper, are each their own alternative, and the man pages switch together with the binaries.

Two limits of the mechanism are accepted. First, the choice is system-wide and is made with administrator privileges; there is no per-user setting, and a user without those privileges can at most shadow the command through their own PATH. Second, the mechanism is not universal: it is standard on the Debian and Red Hat families, which between them cover the large majority of installations, but Arch Linux does not use it and Gentoo has its own eselect. The mechanism chosen for each packaging target is therefore left to the design specification.

<a id="PACK-06-NR"></a>

#### PACK-06-NR — No repository acceptance guarantee

> Acceptance of the package into distribution repositories IS NOT REQUIRED for the product to be considered complete.

Whether Debian, Fedora, or any other distribution accepts the package is a decision made by those projects on their own timetables. The product is complete when the tarball and binary packages of [PACK-01-PR](#PACK-01-PR) through [PACK-03-PR](#PACK-03-PR) are published and the submission efforts of [PACK-04-PR](#PACK-04-PR) are underway.

<a id="PACK-07-PR"></a>

#### PACK-07-PR — Minimal dependency footprint

> The utility’s build-time and run-time dependencies SHOULD be kept to the minimum that the implementation requires.

The reference implementation depends on nothing beyond the C library, which is part of why every distribution carries it. The utility, built on the implementation basis, will depend at run time on libpcap, which every major distribution packages, and at build time on a tree of Rust crates. Run-time dependencies beyond libpcap should be avoided.

Each build-time dependency that is not already packaged in Debian and Fedora is an obstacle to [PACK-04-PR](#PACK-04-PR), and the dependency tree should be reviewed with that cost in mind before it grows.

<a id="PACK-08-NR"></a>

#### PACK-08-NR — No packaging beyond Linux

> Packaging and installation channels for operating systems other than Linux ARE NOT REQUIRED for the product to be considered complete; should any such channel be pursued, the work MUST NOT dilute the drop-in role that the package plays on Linux.

The utility described by this document is a Linux tool ([DELV-01-PR](#DELV-01-PR)), and its packaging requirements, [PACK-01-PR](#PACK-01-PR) through [PACK-07-PR](#PACK-07-PR), are Linux requirements.

Looking to the future, the customary channels for more widespread distribution would be through the ports collections of the BSDs, Homebrew on macOS, and winget on Windows; the source tarball of [PACK-01-PR](#PACK-01-PR), which builds wherever the Rust toolchain and libpcap are available, will help make this possible.

Nothing in this document forbids serving those channels early; it is simply not required.

### Future Proofing

<a id="FPRO-01-PR"></a>

#### FPRO-01-PR — Allow future features

> The design of the utility SHOULD NOT hinder the pursuit of features envisioned for future versions.

Although this document defines the final product, the product may nonetheless evolve. Among the features that can be envisioned for future versions are: machine-readable output formats beyond those of [OFMT-01-PR](#OFMT-01-PR); richer multipath analyses; and use of the utility as a library or as a probing component by other systems, such as IP Routes Live (IPRL). This is a requirement only for awareness and thoughtfulness, not for detailed planning that would slow down the development of the product.

## References

- <a id="ref-sigcomm-2022"></a>[ACM SIGCOMM 2022] ACM SIGCOMM, “IMC Test of Time Award,” 2022. The 2022 award recognizes “Avoiding traceroute anomalies with Paris traceroute” (IMC 2006). https://www.sigcomm.org/awards/imc-test-of-time-award
- <a id="ref-augustin-2006"></a>[Augustin et al. 2006] B. Augustin, X. Cuvellier, B. Orgogozo, F. Viger, T. Friedman, M. Latapy, C. Magnien, and R. Teixeira, “Avoiding traceroute anomalies with Paris traceroute,” in Proceedings of the ACM SIGCOMM Internet Measurement Conference (IMC), 2006. https://doi.org/10.1145/1177080.1177100
- <a id="ref-augustin-2007"></a>[Augustin, Friedman and Teixeira 2007] B. Augustin, T. Friedman, and R. Teixeira, “Multipath tracing with Paris traceroute,” in Proceedings of the IEEE Workshop on End-to-End Monitoring Techniques and Services (E2EMON), 2007. https://doi.org/10.1109/E2EMON.2007.375313
- <a id="ref-butskoy-2007"></a>[Butskoy 2007] Dmitry Butskoy, “Traceroute for Linux,” software, 2007-07-30. SourceForge project page: https://traceroute.sourceforge.net/
- <a id="ref-jacobson-1988"></a>[Jacobson 1988] Van Jacobson, “4BSD routing diagnostic tool available for ftp,” email to the ietf and end2end-interest mailing lists, 1988-12-20. Archived copy: https://gist.github.com/thiteixeira/50cf5f9c26ca0216e4aa6d42b2440216
- <a id="ref-kuznetsov-1999"></a>[Kuznetsov and Yoshifuji 1999] Alexey Kuznetsov and Hideaki Yoshifuji, “iputils,” software, 1999-01-07. GitHub repository: https://github.com/iputils/iputils
- <a id="ref-lohrer-2024"></a>[Lohrer 2024] Téo Lohrer, “Voyage,” software, 2024-11-29. GitHub repository: https://github.com/dioptra-io/voyage
- <a id="ref-veitch-2009"></a>[Veitch et al. 2009] D. Veitch, B. Augustin, R. Teixeira, and T. Friedman, “Failure control in multipath route tracing,” in Proceedings of IEEE INFOCOM, 2009. https://doi.org/10.1109/INFCOM.2009.5062055

## Appendix A: The Reference Implementation in the Distributions

This appendix records a verification, performed on 2026-06-11, of the claims that this document makes about the distribution of the reference implementation. The package repositories or package databases of the major Linux distributions were consulted directly, at the sources linked in the table. In every distribution consulted, the package installed under the name traceroute is Traceroute for Linux, with traceroute.sourceforge.net as its declared upstream.

| Distribution | Release checked | Package version | Verification source |
| --- | --- | --- | --- |
| Debian | 13 “trixie” (stable) | 1:2.1.6-1 | qa.debian.org |
| Debian | 12 “bookworm” (oldstable) | 1:2.1.2-1 | qa.debian.org |
| Ubuntu | 26.04 LTS “resolute” | 1:2.1.6-1build1 (universe) | people.canonical.com |
| Ubuntu | 24.04 LTS “noble” | 1:2.1.5-1 (universe) | people.canonical.com |
| Fedora | Rawhide (F44) | 3:2.1.6-4.fc44 | mdapi.fedoraproject.org |
| RHEL family (via Rocky Linux) | 10 | 2.1.6-3.el10 (BaseOS) | dl.rockylinux.org |
| RHEL family (via Rocky Linux) | 9 | 2.1.1-1.el9 (BaseOS) | dl.rockylinux.org |
| Arch Linux | rolling | 2.1.6-1 (extra) | archlinux.org |
| Gentoo | stable tree | 2.1.5 | packages.gentoo.org |
| openSUSE | Tumbleweed (Factory) | 2.1.6 | api.opensuse.org |
| Alpine Linux | edge | 2.1.6-r0 (community) | pkgs.alpinelinux.org |

The verification supports the claims of the Background section and of [COMP-01-PR](#COMP-01-PR): every major distribution carries the reference implementation as its traceroute, and the most recent upstream release, 2.1.6 of 2024-09-13, is carried by the current releases of Debian, Ubuntu, Fedora, the RHEL 10 family, Arch Linux, openSUSE Tumbleweed, and Alpine Linux.

The 1: and 3: prefixes are packaging epochs, not upstream versions. RHEL does not expose its repositories publicly; its versions are verified through Rocky Linux, which rebuilds RHEL sources.

Two nuances deserve note. First, long-term-support and enterprise releases lag upstream: Ubuntu 24.04 LTS and Gentoo’s stable tree carry 2.1.5, Debian 12 carries 2.1.2, and the RHEL 9 family carries 2.1.1, so the compatibility corpus of [DELV-04-PR](#DELV-04-PR) may encounter older reference behavior in the field. Second, in Ubuntu the package sits in the universe component rather than main, so it is present in the archive but not in a default installation.

## Appendix B: Other Traceroute Implementations

This appendix surveys the traceroute implementations other than the reference implementation.

It distinguishes two groups. The first consists of the traceroutes that are deployed as components of operating systems: these define the environments in which the utility must coexist and, on the platforms beyond Linux to which future versions may be ported ([DELV-01-PR](#DELV-01-PR), [FPRO-01-PR](#FPRO-01-PR)), the native tools that it does not attempt to replace. The second consists of the traceroutes whose interest is historical or scientific: these define the lineage from which the utility descends and the research context in which it will be used.

A pair of kindred diagnostic tools, in wide contemporary use without belonging to either group, follows; and a closing table records which of the four properties of the Motivation section each implementation provides. URLs are given in full so that they can be read in a printed version of this document.

Where more than one implementation from the first table is installed on a Linux system, the alternatives mechanism of [PACK-05-PR](#PACK-05-PR) arbitrates the right to the name traceroute; Traceroute for Linux wins that arbitration by default wherever it is installed.

Appendix A records, distribution by distribution, the versions of the reference implementation carried.

Table B-1. Traceroutes deployed with operating systems

| Implementation | Deployed where | Notes | URL |
| --- | --- | --- | --- |
| Traceroute for Linux | Every major Linux distribution, under the name traceroute | The reference implementation of this document ([COMP-01-PR](#COMP-01-PR)) | https://traceroute.sourceforge.net/ |
| GNU inetutils traceroute | Debian family, as the package inetutils-traceroute | A simpler implementation; loses the default alternatives arbitration to Traceroute for Linux | https://www.gnu.org/software/inetutils/ |
| BusyBox traceroute applet | Alpine Linux and embedded systems | Minimal; holds the name traceroute on Alpine until the full package is installed | https://www.busybox.net/ |
| tracepath (iputils) | Most Linux systems | A related path-discovery tool, unprivileged; often present where no traceroute is installed | https://github.com/iputils/iputils |
| BSD traceroute | The base systems of FreeBSD, OpenBSD, and NetBSD | Descendants of the original Van Jacobson implementation | https://man.freebsd.org/cgi/man.cgi?query=traceroute |
| macOS traceroute | macOS, in the base system | Likewise a Van Jacobson descendant, from Apple’s `network_cmds` collection | https://github.com/apple-oss-distributions/network_cmds |
| tracert | Windows, in the base system | ICMP-only, with its own command line and output format | https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert |

Table B-2. Traceroutes of historical or scientific interest

| Implementation | Interest | Notes | URL |
| --- | --- | --- | --- |
| Van Jacobson traceroute (1988) | Historical | The original traceroute, announced 1988-12-20; ancestor of the BSD and macOS implementations; see the provenance note following this table | https://ee.lbl.gov/ |
| tcptraceroute | Historical | Pioneered TCP probing; its role is absorbed by the reference implementation’s wrapper ([COMP-05-PR](#COMP-05-PR)) | https://github.com/mct/tcptraceroute |
| paris-traceroute and libparistraceroute | Historical and scientific | This project’s ancestors ([SOFT-04-PR](#SOFT-04-PR)); introduced the steady flow identifier of [SPTH-01-PR](#SPTH-01-PR); see the history note following this table | https://github.com/libparistraceroute |
| scamper | Scientific | CAIDA’s bulk measurement engine; implements MDA traceroute; defines the warts format | https://www.caida.org/catalog/software/scamper/ |
| RIPE Atlas traceroute | Scientific | Runs on the probes of the RIPE NCC’s measurement platform; applies the Paris correction through its Paris ID mechanism; its JSON result format is a standard among researchers ([OFMT-01-PR](#OFMT-01-PR)) | https://atlas.ripe.net/ |
| Pietrasanta traceroute | Scientific | Catchpoint’s ECMP-aware fork of Traceroute for Linux; the closest antecedent to this project (see Motivation) | https://github.com/catchpoint/Pietrasanta-traceroute |
| Dublin Traceroute | Scientific | NAT-aware multipath tracerouting, building on the Paris traceroute techniques | https://dublin-traceroute.net/ |

Table B-3. Kindred network-diagnostic tools

| Tool | Notes | URL |
| --- | --- | --- |
| mtr | Combines traceroute and ping in a continuously updating display | https://github.com/traviscross/mtr |
| Trippy | A tool written in Rust, in the spirit of mtr, that draws on Paris traceroute and the MDA literature | https://github.com/fujiapple852/trippy |

Table B-4 closes the survey by returning to the four properties of the Motivation section, which the utility is to be the first to satisfy jointly. The entries for the existing implementations summarize the notes of the tables above and the published descriptions of the utility; the final row records what this document requires of the utility.

Table B-4. Coverage of the four motivating properties

| Implementation | Steady flow identifier (SPTH-01-PR) | MDA multipath tracing (MPTH-02-PR) | Command-line compatibility (COMP-02-PR) | Output compatibility (COMP-03-PR) |
| --- | --- | --- | --- | --- |
| Traceroute for Linux | No | No | Yes: it is the reference implementation | Yes: it is the reference implementation |
| paris-traceroute and libparistraceroute | Yes: introduced the correction | Yes | No | No |
| scamper | Yes | Yes | No | No |
| Dublin Traceroute | Yes | Partial: enumerates paths, without MDA’s statistical guarantees | No | No |
| RIPE Atlas traceroute | Yes: its Paris ID mechanism | No | No: serves a measurement platform, not the command line | No: its own JSON format |
| Pietrasanta traceroute | Partial: steadies the flow of its TCP probes | No | Yes | Yes |
| the utility of this document | Required: [SPTH-01-PR](#SPTH-01-PR) | Required: [MPTH-02-PR](#MPTH-02-PR) | Required: [COMP-02-PR](#COMP-02-PR) | Required: [COMP-03-PR](#COMP-03-PR) |
