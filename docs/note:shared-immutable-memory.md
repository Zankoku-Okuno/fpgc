I keep saying "shared mutable state" is disallowed, but what I'm actually seeing is the disallowance of _shared_ state, cincluding immutable.
This is because the collector moves data around, so references to another manager's memory wil become invalid asynchronously.

I think this could be fixed by recursively-pinning objects.
Probably, only certain objects are even pinnable, so the user/HLL wil have to ensure it only shares shareable things.
(I'm thinking file handles, which really shouldn't be shared between threads without a lock)
