
## Introduction

We propose augmenting the Origin Private File System (OPFS) with a new surface
that brings very performant access to data. This new surface differs from
existing ones by offering in-place and exclusive write access to a file’s
content. This change, along with the ability to consistently read unflushed
modifications and the availability of a synchronous variant on dedicated
workers, significantly improves performance and unblocks new use cases for the
File System Access API.

More concretely, we would add a *createAccessHandle()* method to the
*FileSystemFileHandle* object. It would return an *AccessHandle* that contains
a [duplex stream](https://streams.spec.whatwg.org/#other-specs-duplex) and
auxiliary methods. The readable/writable pair in the duplex stream communicates
with the same backing file, allowing the user to read unflushed contents.
Another new method, *createSyncAccessHandle()*, would only be exposed on Worker
threads. This method would offer a more buffer-based surface for reading and
writing. The creation of AccessHandle also creates a lock that prevents write
access to the file across (and within the same) execution contexts.

This proposal is part of our effort to merge [Storage Foundation
API](https://github.com/WICG/storage-foundation-api-explainer) and File System
Access API. For more context the origins of this proposal, and alternatives
considered, please check out: [Merging Storage Foundation API and the Origin
Private File
System](https://docs.google.com/document/d/121OZpRk7bKSF7qU3kQLqAEUVSNxqREnE98malHYwWec),
[Recommendation for Augmented
OPFS](https://docs.google.com/document/d/1g7ZCqZ5NdiU7oqyCpsc2iZ7rRAY1ZXO-9VoG4LfP7fM).

## Goals & Use Cases

Our goal is to give developers flexibility by providing generic, simple, and
performant primitives upon which they can build higher-level storage
components. The new surface is particularly well suited for Wasm-based
libraries and applications that want to use custom storage algorithms to
fine-tune execution speed and memory usage.

A few examples of what could be done with *AccessHandles*:

*   Distribute a performant Wasm port of SQLite. This gives developers the
    ability to use a persistent and fast SQL engine without having to rely on
    the deprecated WebSQL API.
*   Allow a music production website to operate on large amounts of media, by
    relying on the new surface's performance and direct buffered access to
    offload sound segments to disk instead of holding them in memory.
*   Provide a fast and persistent [Emscripten](https://emscripten.org/)
    filesystem to act as generic and easily accessible storage for Wasm.

## Non-goals

This proposal is focused only on additions to the [Origin Private File
System](https://wicg.github.io/file-system-access/#sandboxed-filesystem), and
doesn't currently consider changes to the rest of File System Access API or how
files in the host machine are accessed.

## What makes the new surface fast?

There are a few design choices that primarily contribute to the performance of
AccessHandles:

*   Write operations are not guaranteed to be immediately persistent, rather
    persistency is achieved through calls to *flush()*. At the same time, data
can be consistently read before flushing. This allows applications to only
schedule time consuming flushes when they are required for long-term data
storage, and not as a precondition to operate on recently written data.
*   The exclusive write lock held by the AccessHandle saves implementations
    from having to provide a central data access point across execution
contexts. In multi-process browsers, such as Chrome, this helps avoid costly
inter-process communication (IPCs) between renderer and browser processes.
*   Data copies are avoided when reading or writing. In the async surface this
    is achieved through SharedArrayBuffers and BYOB readers. In the sync
surface, we rely on user-allocated buffers to hold the data.

For more information on what affects the performance of similar storage APIs,
see [Design considerations for the Storage Foundation
API](https://docs.google.com/document/d/1cOdnvuNIWWyJHz1uu8K_9DEgntMtedxfCzShI7d01cs)

## Proposed API

### New data access surface

```javascript
// In all contexts
const handle = await file.createAccessHandle();
await handle.writable.getWriter().write(buffer);
const reader = handle.readable.getReader({mode: "byob"});
// Assumes seekable streams, and SharedArrayBuffer support are available
var done = await reader.read(buffer, {at: 1});

// Only in a worker context
const handle = await file.createSyncAccessHandle();
var writtenBytes = handle.write(buffer);
var readBytes = handle.read(buffer {at: 1});
```

As mentioned above, a new *createAccessHandle()* method would be added to
*FileSystemFileHandle*. Another method, *createSyncAccessHandle()*, would be
only exposed on Worker threads. An IDL description of the new interface can be
found in the [Appendix](#appendix).

The reason for offering a Worker-only synchronous interface, is that consuming
asynchronous APIs from Wasm has severe performance implications (more details
[here](https://docs.google.com/document/d/1lsQhTsfcVIeOW80dr467Auud_VCeAUv2ZOkC63oSyKo)).

This proposal assumes that [seekable
streams](https://github.com/whatwg/streams/issues/1128) will be available. If
this doesn’t happen, we can emulate the seeking behavior by extending the
default reader and writer with a *seek()* method.

### Locking semantics

```javascript
const handle1 = await file.createAccessHandle();
try {
  const handle2 = await file.createAccessHandle();
} catch(e) {
  // This catch will always be executed, since there is an open access handle
}
await handle1.close();
// Now a new access handle may be created
```

*createAccessHandle()* would take an exclusive write lock on the file that
prevents the creation of any other access handles or  *WritableFileStreams*.
Similarly *createWritable()* would take a shared write lock that blocks the
creation of access handles, but not of other writable streams. This prevents
the file from being modified from multiple contexts, while still being
backwards compatible with the current OPFS spec and supporting multiple
*WritableFileStreams* at once.

Creating a [File](https://www.w3.org/TR/FileAPI/#dfn-file) through *getFile()*
would be possible when a lock is in place. The returned File behaves as it
currently does in OPFS i.e., it is invalidated if file contents are changed
after it was created. It is worth noting that these Files could be used to
observe changes done through the new API, even if a lock is still being held.

## Open Questions

### Naming

The exact name of the new methods hasn’t been defined. The current placeholder
for data access is *createAccessHandle()* and *createSyncAccessHandle()*.
*createUnflushedStreams()* and *createDuplexStream()* have been suggested.

### Assurances on non-awaited consistency

It would be possible to clearly specify the behavior of an immediate async read
operation after a non-awaited write operation, by serializing file operations
(as is currently done in Storage Foundation API). We should decide if this is
convenient, both from a specification and performance point of view.

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
  // Assumes seekable streams are available. The
  // Seekable extended attribute is ad-hoc notation for this proposal.
  [Seekable] readonly attribute WritableStream writable;
  [Seekable] readonly attribute ReadableStream readable;

  // Resizes the file to be size bytes long. If size is larger than the current
  // size the file is padded with null bytes, otherwise it is truncated.
  Promise<undefined> truncate([EnforceRange] unsigned long long size);
  // Returns the current size of the file.
  Promise<unsigned long long> getSize();
  // Persists the changes that have been written to disk
  Promise<undefined> flush();
  // Flushes and closes the streams, then releases the lock on the file
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
