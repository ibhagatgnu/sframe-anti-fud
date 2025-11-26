.. contents:: **Contents**

======================================================================
Some Clarifications to FUD (Fear, Uncertainty, and Doubt) about SFrame
======================================================================

This document addresses some recent claims and questions about the SFrame stack
trace format that are either simply untrue or at best misleading in some way.

Claim: "SFrame maintainer does not address the concerns"
--------------------------------------------------------
Not true.  The SFrame maintainer (and the folks involved in the community at
large) have consistently acknowledged and followed up on the all the concerns
and questions raised to them.  Shifting goal posts, and unhinged circular
argumentation pattern is extremely unproductive (a pattern called out even in
the LLVM community and eventually raised for mediation by the LLVM council).

Few notable clarifications/rebuttals `here
<https://lore.kernel.org/linux-perf-users/87h5vg5tvj.fsf@oracle.com/>`__ , `here
<https://lore.kernel.org/linux-perf-users/2d713719-709d-4b46-8234-2dfe948b836a@oracle.com/>`__,
`here <https://lore.kernel.org/linux-perf-users/18064090-3418-4005-b35e-1afaeb2b4c95@oracle.com/>`__,
`here <https://sourceware.org/pipermail/binutils/2025-October/144899.html>`__, and `here <https://sourceware.org/pipermail/binutils/2025-October/144967.html>`__.

Claim: "Multiple people have raised concerns. The pushback against SFrame is unprecedented"
-------------------------------------------------------------------------------------------
Not True.  As with any new technology, there are bound to be opinions.
"Unprecented pushback" is hyperbole and incorrect characterisation.

Obtaining precise userspace stacks from kernel has substantial implications on
our GNU/Linux ecosystem, and the `GNU/Linux community is interested in SFrame
<https://lwn.net/Articles/1029189/>`_.  Major distros have showed interest in
evaluating SFrame (`gentoo
<https://wiki.gentoo.org/wiki/Project:Toolchain/SFrame>`_ and `Fedora
<https://fedoraproject.org/wiki/Changes/SFrameInBinaries>`_).

Claim: SFrame violates ELF conventions because it is not amenable to concatenation
----------------------------------------------------------------------------------

The ELF specification says that when a linker sees input sections whose type is
in the OS-specific range but otherwise unknown, these sections `shall be
concatenated ([1])
<https://gabi.xinuos.com/elf/03-sheader.html#rules-for-linking-unrecognized-sections>`_.
into an output section of the same type.  SFrame sections have a distinct
section type of SHT_GNU_SFRAME.  Additionally, the ELF specification also says
that if, however, the section carries the flag SHF_OS_NONCONFORMANT, the link
editor shall stop the link process and `emit an error ([2])
<https://gabi.xinuos.com/elf/03-sheader.html#section-flags>`_.  There is a
proposal for the gABI, currently under discussion, to complement the existing
flag with a new `SHF_OS_NONCONFORMANT_DISCARD ([3])
<https://groups.google.com/g/generic-abi/c/3ZMVJDF79g8>`_ :

+-----------------------------------------+------------------------------+
| Unknown input section                   |   Linker behavior            |
+=========================================+==============================+
| Unknown input section                   | Linker behavior              |
+-----------------------------------------+------------------------------+
| No SHF_OS_NONCONFORMANT flag            | Concatenate                  |
+-----------------------------------------+------------------------------+
| SHF_OS_NONCONFORMANT                    | Abort link with an error     |
+-----------------------------------------+------------------------------+
| SHF_OS_NONCONFORMANT_DISCARD (Proposed) | Discard/ignore input section |
+-----------------------------------------+------------------------------+

If the proposed new flag gets adopted by the gABI then linkers that do not
know about SFrame, or any other non-concatenable format, will not be generating
invalid data.  In absence of this new flag, the linkers can check for the
section type and take appropriate action.

In any case, other formats like EH-Frame are also not strictly amenable to
concatenation in practice.

Question: But why is SFrame not concatenable?
---------------------------------------------

SFrame prioritizes ``consumer simplicity`` (parsers/tracers) over
``producer simplicity`` (linkers).  Prospective users of the format find this
simplicity to be beneficial: the data is self-contained, searchable with an
always-present index, without variable-length fields, without run-time
relocations, etc.  We have invested considerable effort in trying to come with
a concatenable format that still features these attributes, but to no avail:
making the data concatenable invariably translates into some extra complexity
(and size overheads) that makes the resulting format not suitable for its
intended use.  The problem is made more challenging because an acceptable
solution must not involve usage of post-processing tools or to shift the
complexity (e.g., index creation) to the users.

Concern: But this means SFrame awareness is necessary to link SFrame
--------------------------------------------------------------------

Yes, a format that is not amenable to concatenation requires some linker
awareness, because the linker needs to know how to combine or merge these input
sections.  To what extent this is a problem in practice is open to
interpretation, but it is certainly not a new situation:   EH Frame requires
linker awareness.  Also other formats, like gdb-index, the Apple Compact Unwind
Info, and the OpenVMS Extensions proposal (currently RFC) also require an
index, and hence linker awareness.  Taking a step back here, we don't see how
this is more of a problem about SFrame than for other already existing formats.

Concern: It is better to not repeat mistakes from the past, like EH Frame
-------------------------------------------------------------------------

In an ideal world we would heartily agree.  However, the fact is that it is
extremely difficult to devise an unwinding or stack walking format that doesn't
require some form of linker awareness.  At least, we haven't manage to figure
out one that would satisfy our user's requirements.  Something better may
appear in the future?  We hope so, but we need to be practical and provide a
solution now.

Concern: The level of linker awareness required by SFrame is higher than in EH Frame
------------------------------------------------------------------------------------

It may be or may be not (implementation design choices will matter), but if so,
certainly not by much.  On one side, the input EH Frame sections can be
concatenated (if linkers skip merging CIEs) but then the linker generates an
additional section with an explicit index to help the users to locate entries
quickly.  On the other, the input SFrame sections get merged such that the
resulting output section is binary-searchable without the need of an external
index.  Conceptually speaking, these two operations are basically equivalent,
and are of similar complexity.

Question: SFrame versions seem to change frequently. Will this burden maintainers?
----------------------------------------------------------------------------------

It is true that there are a lot of differences between SFrame V2 and SFrame V3.
The reason for this is that V3 is the product of all the feedback we have
gotten from one of the projects interested in adopting SFrame, the Linux
kernel.  We don't think it is reasonable to extrapolate this big gap between
versions to future revisions of the format.  The maintainers of SFrame are
aware that any change to the spec should be carefully weighted because of the
impact on the toolchains and any overhead for distros.  That said, no big
changes are expected after SFrame V3.

For future iterations, SFrame does provide unused space in the associated data
structures for extending the format in backwards-compatible ways. E.g.,  extra
bits in SFrame FDE info word can be used to add a new SFrame FDE type.  These
new types of SFrame FDE can ``co-exist`` with those from previous SFrame format
versions.  Similarly, SFrame header can be extended if necessary using the
existing provision of auxiliary header bytes.

SFrame V2 is not expected to be occur in the wild.  We are targeting to
converge to and start from (in userspace) SFrame version 3.

Question: Why does GAS emit a bunch of SFrame related warnings?
---------------------------------------------------------------

The warnings were added on user requests to help them understand coverage of
SFrame sections.  The users wanted to understand the cases when SFrame FDEs are
not generated in the output SFrame section and take appropriate action - fix
handwritten asm, file a bug, request features.  Across the various releases,
user reports based on these warnings have helped address issues.

Many of the current set of remaining warnings emitted are due to a `DRAP
pattern on x86_64
<https://sourceware.org/binutils/wiki/sframe/sframev3todo#Make_support_for_topmost_frames_more_flexible>`_,
a feature currently targetted for SFrame V3.  We have discussed the handling of
warnings `previously
<https://sourceware.org/pipermail/binutils/2025-August/143672.html>`_ on the
Bintuils mailing list.

Concern: But there are more ELF related problems with SFrame, like GC and section grouping
------------------------------------------------------------------------------------------

The problems related to GC and section grouping were discussed during the
SFrame talk at the GNU Tools Cauldron 2025.  and a solution was discussed,
proposed independently by Roland in the gABI discussion thread. The negative
side effect of increased object file sizes and link-time overheads, when
-ffunction-sections is used, will need to be evaluated by the stakeholders.
The issue is tracked via `PR ld/32769
<https://sourceware.org/bugzilla/show_bug.cgi?id=32769>`_ with a target
milestone of 2.46.  An alternative way of resolving this is to use similar
mechanisms as used for EH Frame, which (like .sframe) also manifests as one
.eh_frame section for all .text.* sections in the binary.  Given the proximity
of the target milestone and relative priority of other SFrame V3 tasks, using
the alternative way may be favorable.

Question: Is SFrame larger than .eh_frame?
--------------------------------------------

Our measurements on Gentoo binaries indicate SFrame is generally more compact
or equivalent, at the moment:

 * AArch64: SFrame is ~70% the size of .eh_frame (plus .eh_frame_hdr).
 * x86_64: SFrame is roughly 100% to .eh_frame (plus .eh_frame_hdr).

See `sframe-cmp github repo ([6]) <https://github.com/thesamesam/sframe-cmp>`_
for specific benchmark methodology.  The results are using GNU Binutils
implemenation.  That said, the size of SFrame sections may vary across
implementations, user packages, format versions and so on.  The important bit
to keep in perspective is the value it brings: fast, precise, and low-overhead
stack traces filling the current gap in observability in GNU/Linux systems left
unfulfilled by Frame pointers and EH Frame.

Concern: Even if SFrame is the same size than EH Frame or less, it is not a substitute for EH Frame so the size of objects will increase
-----------------------------------------------------------------------------------------------------------------------------------------

SFrame is ``not`` a replacement for .eh_frame; And it was ``never`` claimed to
be one.  SFrame is a ``Stack Tracing`` format, whereas .eh_frame is an
``Unwinding`` format.  Two different use cases with different requirements.

+------------------+--------------------------------------+------------------------------------------------+
| Feature          | SFrame                               | .eh_frame                                      |
+==================+======================================+================================================+
| Primary Goal     | Fast Backtraces (Profiling/Tracing)  | Robust Exception Handling, Debugging           |
+------------------+--------------------------------------+------------------------------------------------+
| Capabilities     | Finds Return Address & CFA           | Restores CFA and Register Values               |
+------------------+--------------------------------------+------------------------------------------------+
| Complexity       | Low (Simple Lookups)                 | High (Relatively high overhead for in-kernel   |
|                  |                                      | parsers)                                       |
+------------------+--------------------------------------+------------------------------------------------+

Question: Why can't we just use .eh_frame for tracing?
------------------------------------------------------

EH Frame provides very complete and accurate information, enough to unwind the
stack.  However, the flexibility required for that comes at a cost:
interpreting EH Frame sections require to interpret DWARF expressions and other
complexities that some stack tracer implementations may not be willing to
accommodate.  Online tracing often happens in contexts that impose unusual
restrictions, e.g., low memory and execution time overheads, no dynamic memory
allocation.  The format providing the metadata to the stack tracer needs to be
simple enough to be practical in these environments.  This is a real issue: the
Linux kernel `removed their DWARF unwinder ([4])
<https://lwn.net/Articles/728347/>`_ back in 2017 due to this reason.

Question: What about Shadow stacks ?
------------------------------------
Some hardwares implement mechanishm to curb ROP-style attacks. Intel and AMD
chips now implement CET which includes `shadow stacks
<https://lwn.net/Articles/885220/>`_ and is available on many currently
shipping processors.  On AArch64 systems, the feature is known as GCS (Guarded
Control Stack).

Availability of shadow stack enabled hardware may improve over time for `some
architectures
<https://lore.kernel.org/linux-perf-users/baf2665c-49aa-4b8a-be26-69dc23876bee@sirena.org.uk/>`_,
but it will be a while before these are the majority of the deployed resources.
Some applications may choose to disable shadow stack for specific reasons.
Further, architectures which lack shadow stacks may need support from software
(in the form of metadata sections) for asynchronous stack tracing.

Question: Why not use Apple's "Compact Unwind" format?
------------------------------------------------------

The `Compact Unwind format ([7])
<https://faultlore.com/blah/compact-unwinding/>`_ used by Apple on Darwin
systems is supported by LLVM in Mach-O objects.  On one side, it only
complements EH Frame and can't replace it entirely: compact unwind info can
only be generated for functions which are compiled in a way the stack frame can
be located using a single opcode. In functions where that doesn't hold, the
unwinder or tracer should resort to EH Frame.  On the other side, this format
requires the collaboration of the compiler and the only way to guarantee code
will actually benefit from it would be to add more restrictions to the
different ABIs to increase the odds that compact unwind info will be generated
for all compiled functions. (See brief discussion on LLVM discourse `here ([5])
<https://discourse.llvm.org/t/rfc-improving-compact-x86-64-compact-unwind-descriptors/47471/14>`_).
That format is also not fully asynchronous and may miss some shapes of
prologues and epilogues, especially in presence of shrink wrapping and its
variants.

Concern: But there are some extensions from OpenVMS that improve the Apple compact format
------------------------------------------------------------------------------------------

The OpenVMS extensions proposal ([5]) is for asynchronous compact unwind
descriptors. The proposal (in RFC stage) is for adding the ability to describe
a variety of prologues and epilogues, which is vital for asynchronous stack
tracing and stack unwinding. The RFC, proposed in 2018, was for x86_64 only.
Other architecture/ABIs still need to be included for completeness.

It remains to be seen how this proposal manages the fine line of
space-efficiency while trying to be the goto format for both asynchronous stack
unwinding and fast, precise and low-overhead stack tracing.  The goal for
OpenVMS project is to subsume .eh_frame; a goal that still needs consensus with
the affected stakeholders (other compilers, possibly ABI maintainers, and of
course the Linux kernel).  SFrame's charter does not aim for that goal.

============
References
============

 [1] "ELF Object File Format" 3.8. Rules for Linking Unrecognized Sections"
    https://gabi.xinuos.com/elf/03-sheader.html#rules-for-linking-unrecognized-sections
 [2] "ELF Object File Format" 3.8. Rules for Linking Unrecognized Sections"
    https://gabi.xinuos.com/elf/03-sheader.html#section-flags
 [3] "Proposal for new flag SHF_OS_NONCONFORMING_DISCARD"
    https://groups.google.com/g/generic-abi/c/3ZMVJDF79g8
 [4] "Re: [PATCH 7/7] DWARF: add the config option"
    https://lwn.net/Articles/728347/
 [5] "[RFC] Improving compact x86-64 compact unwind descriptors"
    https://discourse.llvm.org/t/rfc-improving-compact-x86-64-compact-unwind-descriptors/47471/14
 [6] "Tool(s) and data to do with SFrame binary size."
    https://github.com/thesamesam/sframe-cmp
 [7] "The Apple Compact Unwinding Format: Documented and Explained"
    https://faultlore.com/blah/compact-unwinding/

