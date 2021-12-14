weak references are the one feature I'm not totally sure how to implement
The thing is, if a weak reference is traced, it may not yet know if its target object is still alive, and thus whether it should also be saved.
Thus, (surviving) weak references need to be copied and updated after tracing all strong references.

My immediate thought is that weak references should be managed objects with two untraced pointer fields.
One points to their referent, the other to the next (or previously-allocated) weak reference in a linked list.
Once normal tracing is done, the weak reference list is then iterated.
Surviving weak references will already have been copied into the survivor space, but they will still need updating.
When a non-surviving weakref is encountered, the previous link must be updated to the next surviving weakref.
For surviving weak references, if their referent has survived (is marked/has a forwarding pointer), then their (untraced) pointer field must be updated to the forwarding pointer.
If their referent has not survived, their pointer field should be cleared.
Dereferencing a weak reference is therefore a double pointer-chase.
