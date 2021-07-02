# Adding AccessHandles to files

## Authors:

* Emanuel Krivoy (fivedots@chromium.org)
* Richard Stotz (rstz@chromium.org)

## Participate

* [Issue tracker](https://github.com/WICG/file-system-access/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals and Use Cases](#goals-and-use-cases)
- [Non-goals](#non-goals)
- [Proposed API](#proposed-api)
  - [New data access surface](#new-data-access-surface)
  - [Locking semantics](#locking-semantics)
- [Open Questions](#open-questions)
  - [Naming](#naming)
  - [Assurances on non-awaited consistency](#assurances-on-non-awaited-consistency)
- [Appendix](#appendix)
  - [AccessHandle IDL](#accesshandle-idl)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This is a proposal to support
[Storage Foundation API's](https://github.com/WICG/storage-foundation-api-explainer)
use cases through the Origin Private File System (OPFS), to provide a unified
interface and a simpler model to developers.

Storage Foundation’s main objective is to support performance-sensitive
applications through a very fast and generic storage backend. It especially
focuses on ported WebAssembly programs for which existing storage web APIs
don’t match the expectations of native code. There are a few design choices
that primarily contribute to Storage Foundation’s performance:

*   Developers get control of when data is flushed to disk, and reads can be
    executed consistently without flushing. This allows programs to only schedule
    time consuming flushes when they are required for persistency, and not as a
    precondition to operate on recently written data.
*   Its design ensures that implementations can access data from “the renderer
    process” (i.e. without needing to rely on a component that coordinates
    between execution contexts) to avoid costly inter-process communication
    (IPCs). Examples of this approach include the allocation-based quota system
    and only allowing one open file handle per file.
*   We avoid extra memory allocations by taking a buffer when reading.

In order to bring this use cases to OPFS, we propose adding a method to
*FileSystemFileHandle* that returns an *AccessHandle*, a new interface.
*AccessHandles* provide the right kind of access and locking semantics to achieve
similar performance to Storage Foundation API.

Further details on the origins of this proposal, and alternatives considered,
can be found in the following documents:
[Merging Storage Foundation API and the Origin Private File System](https://docs.google.com/document/d/121OZpRk7bKSF7qU3kQLqAEUVSNxqREnE98malHYwWec),
[Recommendation for Augmented OPFS](https://docs.google.com/document/d/1g7ZCqZ5NdiU7oqyCpsc2iZ7rRAY1ZXO-9VoG4LfP7fM).

## Goals and Use Cases

Out goals and use cases are the same as
[Storage Foundation API](https://github.com/WICG/storage-foundation-api-explainer),
namely to give developers flexibility by providing generic, simple, and
performant primitives upon which they can build higher-level components. The
new surface is particularly well suited for Wasm-based libraries and
applications that want to use custom storage algorithms to fine-tune execution
speed and memory usage.

A few examples of what could be done with *AccessHandles*:

*   Allow tried and true technologies to be performantly used as part of web
    applications e.g. using a port of your favorite storage library
    within a website
*   Distribute a Wasm module for WebSQL, allowing developers to us it across
    browsers and opening the door to removing the unsupported API from Chrome
*   Allow a music production website to operate on large amounts of media, by
    relying on the new surface's performance and direct buffered access
    to offload sound segments to disk instead of holding them in memory
*   Provide a persistent [Emscripten](https://emscripten.org/) filesystem that
    outperforms
    [IDBFS](https://emscripten.org/docs/api_reference/Filesystem-API.html#filesystem-api-idbfs)
    and has a simpler implementation

## Non-goals

This proposal is focused only on additions to the
[Origin Private File System](https://wicg.github.io/file-system-access/#sandboxed-filesystem),
and doesn't currently consider changes to the rest of File System Access API or
how files in the host machine are accessed.

## Proposed API

### New data access surface

```javascript
//In all contexts
const handle = await file.createAccessHandle();
await handle.writable.getWriter().write(buffer);
const reader = handle.readable.getReader({mode: "byob"});
//Assumes seekable streams are available
var {value: buffer, done} = await reader.read(buffer, {at: 1});

//Only in a worker context
const handle = file.createSyncAccessHandle();
var writtenBytes = handle.write(buffer);
var readBytes = handle.read(buffer {at: 1});
```

A new *createAccessHandle()* method would be added to *FileSystemFileHandle*.
It  would return an *AccessHandle* that contains a
[duplex stream](https://streams.spec.whatwg.org/#other-specs-duplex) and
auxilliary methods. The readable/writable pair in the duplex stream would
communicate with the same backing file, allowing the user to read unflushed
contents. Another new method, *createSyncAccessHandle()*, would be only exposed
on Worker threads. This method would offer a more buffer based surface for read
and write. An IDL description of the new interface can be found in the
[Appendix](#appendix).

The reason for offering a Worker-only synchronous interface, is that consuming
asynchronous APIs from Wasm has severe performance implications (more details
[here](https://docs.google.com/document/d/1lsQhTsfcVIeOW80dr467Auud_VCeAUv2ZOkC63oSyKo)).
We've opted for slightly different interfaces between the async and sync
versions, because there currently isn’t support for synchronous streams. The
amount of effort that would be required to spec them in a way that makes them
compatible with asynchronous streams would be prohibitively high, and likely
not worth it to support a single API.  Furthermore, we hope to discourage the
use of the sync surface unless a developer is dealing with legacy Wasm ports,
and therefore there is not much benefit to adding them.

This proposal assumes that
[seekable streams](https://github.com/whatwg/streams/issues/1128) will be
available. If this doesn’t happen, We can emulate the seeking behavior by
extending the default reader and writer with a *seek()* method.

### Locking semantics

```javascript
const handle1 = await file.createAccessHandle();
try {
  const handle2 = await file.createAccessHandle();
} catch(e) {
  //This catch will always be executed, since there is an open access handle
}
await handle1.close();
//Now a new access handle may be created
```

In order to avoid multiple contexts modifying a file at the same time, locking
semantics would be added to the new surface. At any given time, and across
execution contexts, there would only be either a single *AccessHandle* per
FileHandle, or potentially multiple Writables created through
*createWritable()*.

When creating an *AccessHandle*, a lock will be taken. This lock will be released
when the *AccessHandle* is closed or destroyed. When a Writable is created, a
lock is taken if there are no other open Writables. This lock will be released
once the last Writable closed or destroyed. Only one lock may be taken at a
given time for a given FileHandle.

There are some important edge cases to mention:

*   Locks acquired by creating a sync handle also prevent the creation of async handles.
*   Creating a File through *getFile()* is possible when a lock is in place. The
    returned File behaves as it currently does in OPFS i.e. it is invalidated
    if file contents change after it was created.  In our particular case this
    means that Files created while there is an active handle will be invalidated
    when a flush is executed (either explicitly through flush() or implicitly by
    the OS). It also means that these Files could be used to observe flushed
    changes done through the new API, even if a lock is still being held.

## Open Questions

### Naming

The exact name of the new methods hasn’t been defined. The current placeholder
for data access is *createAccessHandle()* and *createSyncAccessHandle()*.
*createUnflushedStreams()* and *createDuplexStream()* have been suggested.

### Assurances on non-awaited consistency

It would be possible to clearly specify the behavior of an immediate async read
after a non-awaited write, by serializing file operations (as is currently done
in Storage Foundation API). We should decide if this is convenient, both from a
specification and performance point of view.

## Appendix

### AccessHandle IDL

```webidl
interface FileSystemFileHandle : FileSystemHandle {
  Promise<File> getFile();
  Promise<FileSystemWritableFileStream> createWritable(optional FileSystemCreateWritableOptions options = {});

  Promise<FileSystemAccessHandle> createAccessHandle();
  [Exposed=DedicatedWorker]
  Promise<FileSystemSyncAccessHandle> createSyncAccessHandle();
};

interface FileSystemAccessHandle {
  //Assumes seekable streams are available. The
  //Seekable extended attribute is ad-hoc notation for this proposal.
  [Seekable] readonly attribute WritableStream writable;
  [Seekable] readonly attribute ReadableStream readable;

  //Resizes the file to be size bytes long. If size is larger than the current
  //size the file is padded with null bytes, otherwise it is truncated.
  Promise<undefined> truncate([EnforceRange] unsigned long long size);
  //Returns the current size of the file.
  Promise<unsigned long long> getSize();
  //Persists the changes that have been written to disk
  Promise<undefined> flush();
  //Cancels the streams and releases the lock on the file
  Promise<undefined> close();
};

[Exposed=DedicatedWorker]
interface FileSystemSyncAccessHandle {
  unsigned long long read([AllowShared] BufferSource buffer,
                             FilesystemReadWriteOptions options);
  unsigned long long write([AllowShared] BufferSource buffer,
                              FilesystemReadWriteOptions options);

  Promise<undefined> truncate([EnforceRange] unsigned long long size);
  Promise<unsigned long long> getSize();
  Promise<undefined> flush(); 
  Promise<undefined> close();
};

dictionary FilesystemReadWriteOptions {
  [EnforceRange] unsigned long long at;
};
```

## References & acknowledgements

Many thanks for valuable feedback and advice from:

Domenic Denicola, Marijn Kruisselbrink, Victor Costan
