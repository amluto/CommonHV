CommonHV, a common hypervisor interface
=======================================

This is CommonHV draft 1.

The CommonHV specification is Copyright (c) 2014 Andrew Lutomirski.

Licensing will be determined soon.  The license is expected to be extremely
liberal.  I am currently leaning towards CC-BY-SA for the specification and
an explicit license permitting anyone to implement the specification
with no restrictions whatsoever.

I have not patented, nor do I intend to patent, anything required to implement
this specification.  I am not aware of any current or future intellectual
property rights that would prevent a royalty-free implementation of
this specification.

I would like to find a stable, neutral steward of this specification
going forward.  Help with this would be much appreciated.

Scope
-----

CommonHV is a simple interface for communication
between hypervisors and their guests.

CommonHV is intended to be very simple and to avoid interfering with
existing paravirtual interfaces.  To that end, its scope is limited.
CommonHV does only two types of things:

  * It provides a way to enumerate other paravirtual interfaces.
  * It provides a small, extensible set of paravirtual features that do not
    modify or replace standard system functionality.

For example, CommonHV does not and will not define anything related to
interrupt handling or virtual CPU management.

For now, CommonHV is only applicable to the x86 platform.

Discovery
---------

A CommonHV hypervisor MUST set the hypervisor bit (bit 31 in CPUID.1H.0H.ECX)
and provide the CPUID leaf 4F000000H, containing:

  * CPUID.4F000000H.0H.EAX = max_commonhv_leaf
  * CPUID.4F000000H.0H.EBX = 0x6D6D6F43
  * CPUID.4F000000H.0H.ECX = 0x56486E6F
  * CPUID.4F000000H.0H.EDX = 0x66746e49

EBX, ECX, and EDX form the string "CommonHVIntf" in little-endian ASCII.

max_commonhv_leaf MUST be a number between 0x4F000000 and 0x4FFFFFFF.  It
indicates the largest leaf defined in this specification that is provided.
Any leaves described in this specification with EAX values that exceed
max_commonhv_leaf MUST be handled by guests as though they contain
all zeros.

CPUID leaf 4F000001H: hypervisor interface enumeration
------------------------------------------------------

If max_commonhv_leaf >= 0x4F000001, CommonHV provides a list of tuples
(location, signature).  Each tuple indicates the presence of another
paravirtual interface identified by the signature at the indicated
CPUID location.  It is expected that CPUID.location.0H will have
(EBX, ECX, EDX) == signature, although whether this is required
is left to the specification associated with the given signature.

If the list contains N tuples, then, for each 0 <= i < N:

  * CPUID.4F000001H.i.EBX, CPUID.4F000001H.i.ECX, and CPUID.4F000001H.i.EDX
    are the signature.
  * CPUID.4F000001H.i.EAX is the location.

CPUID with EAX = 0x4F000001 and ECX >= N MUST return all zeros.

To the extent that the hypervisor prefers a given interface, it should
specify that interface earlier in the list.  For example, KVM might place
its "KVMKVMKVM" signature first in the list to indicate that it should be
used by guests in preference to other supported interfaces.  Other hypervisors
would likely use a different order.

The exact semantics of the ordering of the list is beyond the scope of
this specification.

CPUID leaf 4F000002H: miscellaneous features
--------------------------------------------

CPUID.4F000002H.EAX is nonzero if the CommonHV RNG interface is available.
CPUID.4F000002H.EBX, CPUID.4F000002H.ECX, and CPUID.4F000002H.EDX are reserved
and must be zero in hypervisors compliant with this version of the CommonHV
specification.

### CommonHV RNG

If CPUID.4F000002H.EAX is nonzero, then it contains an MSR index used to
communicate with a hypervisor random number generator.  This MSR is
referred to as MSR_COMMONHV_RNG.

rdmsr(MSR_COMMONHV_RNG) returns a 64-bit best-effort random number.  If the
hypervisor is able to generate a 64-bit cryptographically secure random number,
it SHOULD return it.  If not, then the hypervisor SHOULD do its best to return
a random number suitable for seeding a cryptographic RNG.

A guest is expected to read MSR_COMMONHV_RNG several times in a row.
The hypervisor SHOULD return different values each time.

rdmsr(MSR_COMMONHV_RNG) MUST NOT result in an exception, but guests MUST
NOT assume that its return value is indeed secure.  For example, a hypervisor
is free to return zero in response to rdmsr(MSR_COMMONHV_RNG).

wrmsr(MSR_COMMONHV_RNG) offers the hypervisor up to 64 bits of entropy.
The hypervisor MAY use it as it sees fit to improve its own random number
generator.  A hypervisor SHOULD make a reasonable effort to avoid making
values written to MSR_COMMONHV_RNG visible to untrusted parties, but
guests SHOULD NOT write sensitive values to wrmsr(MSR_COMMONHV_RNG).

A hypervisor is free to ignore wrmsr(MSR_COMMONHV_RNG), but wrmsr to
MSR_COMMONHV_RNG MUST NOT result in an exception.

Note that the CommonHV RNG is not intended to replace stronger, asynchronous
paravirtual random number generator interfaces.  It is intended primarily
for seeding guest RNGs early in boot.

Future extension
----------------

CPUID leaves beyond those defined in this version of the CommonHV specification
should be ignored by guests written for this version of the specification.
