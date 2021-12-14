# Functional Programming Garbage Collector (fpgc)

This is intended to be a C library implementing a precice garbage collector.
Note: fpgc does not make C a garbage-collected lnguage (like e.g. Guix's libgc), but rather exposes a C interface to managed memory.
The design specifically focuses on supporting functional programming runtime systems, but it is not limited to either functional programming, or runtime systems.

Distinguishing features of fpgc are:
  * generational moving garbage collector with remembered sets
  * allocation and collection within a manager does not synchronize
  * no global state: create multiple memory managers
  * objects in one manager _do not_ refer to objects from another
  * any thread can use any memory manager(s), with support for synchronizing which engines are in-use
  * new managers can take very little memory overhead, but can become more efficient by growing if they turn out to be long-lived
  * ownership of the (entire) memory of a manager can be transferred to another manager
  * interoperates with external code: alignment restrictions, pinned objects, finalizers
  * agnotic to stack layout: the rootset is explicitly managed rather than attempting a stack trace with a fixed calling convention
  * compile-time tunable parameters for various sizes and thresholds

What this means is that there is no support for shared mutable state, which is a common source of bugs and performance bottlenecks anyway.
In particular, we eliminate synchronization overhead on allocation, writing references into existing objects, and (the big one) collection.
W.r.t. that last bottleneck, what this means is that even though each individual memory manager uses a stop-the-word strategy, other threads can continue computing as normal.
This ishould make the system more suited to interactive applications which are sensitive to collection pauses.

Instead of shared state, message-passing is the prefered method of inter-thread communication, which is also applicable to distributed computing scenarios.
To aid with message-passing, this library provides tools to serialize and deserialize managed objects, including their referential structure.
This is not a complete serializer however, since the collector does not know the contents of objects: attempting to serialize a local resource (such as a file handle or a reference to unmanaged memory) may fail if the message is deserialized on a different compute node.
If the last action of a thread (and its manager) is to send a message, that manager's memory can be transfered in whole to another manager; this helps support efficient programming with promises.

References to shared mutable memory can of course be supported indirectly (e.g. through pinning, or managed smart pointers to unmanaged memroy).
However, I envy Erlang its ability to be distributed across networks, and it would be difficult for this collector to acheive such scalability if it attempted to manage shared mutable state.

