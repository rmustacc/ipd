---
authors: Robert Mustacchi <rm@fingolfin.org>
state: pre-draft
---

# IPD 11 Processor Resource Management

Modern x86 processors are made up of multiple cores with varying level
of shared and exclusive caches as well as complex links that connect
them together and provide access to other memory resources such as DRAM
and non-volatile, persistent memory. In some usage cases, the fact that
the cache and memory bus is shared between multiple different programs,
zones, virtual machines, or really any distinct group can sometimes lead
to varying performance.

While there are various techniques that illumos has to try and create
independent domains to address these such as [processor
sets](https://illumos.org/man/1m/psrset) and [resource
pools](https://illumos.org/man/1m/pooladm), these operate at somewhat
broader granularities in the system and provide different abstractions.
Instead, Intel has introduced what they call [Resource Director
Technology
(RDT)](https://www.intel.com/content/www/us/en/architecture-and-technology/resource-director-technology.html)
and AMD has put back code into Linux indicating that they have a similar
variant, with a few minor differences.

These new technologies allow the following (based on the processor
generation):

 * Monitoring last-level cache occupancy (e.g. L3 cache)
 * Monitoring memory bandwidth usage
 * Allocating portions of the last-level cache
 * Allocating cache lines to code versus data
 * Allocating memory bandwidth

The rest of this IPD talks about some of the use cases, the hardware
features that exist and how they interact, and has some high-level
proposals for how we might look at enabling this in illumos.

## Use Cases

This section talks about some of the goals that we have with this work
and what we want to be able to enable:

1. Monitor the cache and memory bandwidth utilization of a thread,
process, zone, or other collection / grouping of entities in the system.

2. Allocate a fixed portion of the cache or memory bandwidth to a
particular application or portion there of. This would allow
administrators to provide some amount of quality of service. A common
example would be to enable a hypervisor to assign a hardware virtualized
guest a fixed portion of the cache from which they are utilizing.

3. Allowing a portion of the cache to be allocated and used for locked
down as part of a defense in depth strategy to protect against some
classic cache related side channel attacks. An example of this is
discussed in the [CATalyst
Paper](http://palms.ee.princeton.edu/system/files/CATalyst_vfinal_correct.pdf)
and an extension of it was part of Alex Wilson's proposal for having a
specific pool of pages as part of a [cache side-channel mitigation in
Joyent RFD
77](https://github.com/joyent/rfd/blob/master/rfd/0077/README.adoc#1233-cache-side-channel-mitigation).

4. The different cases above suggest that we have a need both for a
programmatic library interface as well as a command line utility which
can be used to perform different aspects. Having a library interface
makes it much easier for such a thing to be plugged into other parts of
the system as a building block.

## Hardware Features

Before we get into concrete proposals, it's worth taking some time to
discuss how these different interfaces that hardware provides actually
work and what they do.

The architectural enhancements that these extensions provide on x86
provide two new fundamental resources that can be associated with a
running logical CPU:

1. A resource monitoring ID
2. A class of service

### Resource Monitoring IDs

The first of these, a resource monitoring ID, basically is used to tag
operations that are going on while a given entity is executing on CPU.
The system can then come back and ask each logical CPU for the current
state of a monitorable resource based on a given resource ID. In this
case, the monitorable resources include:

* L3 cache occupancy
* L3 Total Memory Bandwidth
* L3 Local Memory Bandwidth

One important caveat of L3 cache occupancy is that the resource ID that
is active when a cache line is filled is effectively what is tagged as
part of the occupancy measurement. Depending on the implementation,
additional accesses may change the tag. Consider two differently tagged
entities accessing a shared line of libc text.

Critically, neither this nor the cache allocation technology change
which parts of the cache are valid for a logical processor to service a
cache hit from.

Another issue with RMID association for the L3 cache is that if an ID
has changed associations, then old values that are in the cache will
still have been filled with the old tag. One way around this is to
perform a full flush of the L3 cache on x86 with the `wbinvd`
instruction.

### Classes of Service

The second item that exists is a class of service. A class of service
describes a set of resource allocations and constraints. Currently, the
following resources are allocatable to a particular class of service:

* L3 cache filling
* Memory Bandwidth

There are a few different things which complicate this. Each type of
resource has a maximum number of classes of service that it supports.
For example, let's say a given system advertises that it has 8 L3 cache
classes and 4 memory bandwidth classes. The way that the system works is
that IDs 0-3 support both L3 cache and memory bandwidth, but IDs 4-7
only support L3 cache activities.

The take away from this is that a class is used across all of the
different resources and are not independent. While you can certainly use
a given class without enabling the other resources, it means that that
ID has been used up.

There are a couple of interesting constraints in these different
technologies that exist in different processor families. One is that
bandwidth limiting is done on a per-core basis. This means that if one
thread on a core has a generous bandwidth allotment, but the other has a
minimal one, then the minimal one will impact both threads.

On the other hand the L3 cache quality of service basically only
controls the act of cache filling. If the same cache line is being
accessed by multiple threads in different classes, then the associated
class may change, which could impact cache filling algorithms.

Both the L3 cache filling allocation and the memory bandwidth logic have
various additional constraints. For example, the way that the cache is
allocated has constraints on some platforms which make it harder to
potentially divide up.

### Associating Classes

A logical processor is associated with both a class of service and
resource monitoring ID by writing to the MSR `IA32_PQR_ASSOC`. This MSR
exists on a per-logical processor basis and contains both the current
class of service that should be assigned and the resource monitoring ID.
As processes are scheduled in and out, we'll want to update the
currently assigned PQR value. This behaves the same on both AMD and
Intel.

## Interface Proposals

While today architectural resource monitoring on x86 covers different
aspects of the memory hierarchy, we'd like to put together something
that doesn't constrain us too much in the future. With that, I have a
few concrete proposals and suggestions, while the full details of the
interface still need to be worked out as part of broader exploration and
prototyping.

### A New System Abstraction

While the system today has the ability to create resource pools,
processor sets, and other entities that manage the system, I believe
that this should be a fundamentally orthogonal construct. That is,
resource pools or other things could be extended to include something
that manages this, but we still should have our own first class thing
that exists. It'll be up to consumers to ensure that CPU binding is done
in whatever way makes sense.

### Uniform Groups

One of the challenges with this interface is that hardware has wildly
different ways of expressing these capabilities and the limits thereof.
For example, we may have a different number of resource monitoring IDs
and different QoS mechanisms may have a different number of classes of
service. With that in mind, I'd concretely propose the following:

1. We have our own notion of a processor resource group. Processes and
individual threads can be added and removed from these like other tools
allow for, which allows for specifying groups, projects, etc.

2. These resource groups are global in the system. Until a later time,
where it makes sense to try and limit the size and scope of these, these
groups should be global, even if the resource allocations is only from a
specific set or subset of processors. This is designed to simplify how
we handle things from a user perspective. We don't want to have to force
complex topologies onto the user per se until such time as that's
required.

3. There is no distinction between groups where we want to monitor
something or assign resources. A single entity that we allocate will be
fully featured. This means that we will always be able to monitor all of
the resources and also be able to assign every class of service.

4. A side effect of the above is that there will be a limited number of
groups. This will be sized based on the smallest available resource.
This is purely an implementation detail and should not impact users at
this time. If we want to loosen, then we can do so without changing the
API. However, the goal is that a user does not need to think
about the details of how many there are and how to multiplex between
one that can be monitored or one that can do memory bandwidth
allocations.

While these do make our ability to use some of the resources a little
bit harder, they simplify the initial way that we can present this to
user land and allow us to a bit more flexibility in terms of how this
might look on other architectures.

### Rough Commands and Interfaces

At this time, it's a bit hard to have a firm set of definitions in terms
of command line or library interfaces. I would suggest that we add a new
command `prmadm` (process resource management administration) which has
a number of high level features:

* Information about the capabilities of the system. This would include
things like: how many groups exist, which monitoring and QoS items are
available, what are the constraints on carving them up.

* Creating, listing, and deleting processor resource management (prm)
groups.

* Adding and removing processes from prm groups based on the standard
array of things like pid, tid, zone, project, etc.

* Assigning a particular QoS to a group.

* Monitoring a particular resource of a group. This would probably
operate only on a single group, but would allow flexibility across CPUs.

At this time, it's hard to project what a useful library interface
should be. This IPD will be updated again when we have a more useful
sense of that.

### Rough Kernel Design Sketch

We would probably implement this as a new pseudo-device driver that
exposed a character device that took ioctls to drive this. When
processes or threads were keyed into this, we would use the scheduling
context ops to affect this change. One small issue with the context ops
usage is that it may lead to a bit more shuffling of the associated IDs
on the system.

While it's most likely that this would never lead x86, we should try and
implement this in a generic manner. If we do need per-CPU data, we can
always extend the x86 machcpu.

It's unclear whether it'll make sense to do generic x86_feature
detection in cpuid.c and add items here to the feature set or if we
should do that in this subsystem. As we gain experience by exploring
this, we'll come back and flesh this section out.

### Persistence

As part of the initial implementation, I think we probably don't want
this to be persistent as part of what we develop here. Processor sets
and binding aren't persistent on their own, but various tooling makes
them persistent in their own context. For example, it might make sense
to build this into things like zones or other infrastructure.
