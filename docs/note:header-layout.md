small (in-block) object header
  nPtrs: uint(N-3)
  nBytes: uintN
  nElems: uintN
  alignment: uintM
  inRememberedSet: bool
  blockSize: {small, large, noblock}
  hasFinalizer: bool
    finalizer: void (*)(gcobj*)
    nextCleanupObj: gcobj*

block header:
  owningManager: Manager*
  generation: {0,1,2,3}
  marked: bool
  pinned: bool

huge object header (> large block size):
  owningManager: Manager*
  finalizer: void(*)(gcobj*)
  nextCleanupObj: gcobj*
  nPtrs: size_t
  nBytes: size_t
  nElems: size_t
  alignment: uint6
  generation: {0,1,2,3}
  inRememberedSet: bool
  marked: bool
  pinned: bool
