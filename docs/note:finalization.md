what is the strategy to find object that need finalization?

The only think I can think is to have a linked list embedded in the finalizing objects.
That does mean two extra pointers per finalizing object,: one for the finalizer function pointer, and one to the lniked list.
These list, I supose, should be block-local however, to ease the time to traverse when scavenging.
