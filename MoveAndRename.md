# Adding FileSystemHandle::move() and FileSystemHandle::rename() methods

## Authors:

* Austin Sullivan (asully@chromium.org)

## Participate

* [Issue tracker](https://github.com/WICG/file-system-access/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Adding FileSystemHandle::move() and FileSystemHandle::rename() methods](#adding-filesystemhandlemove-and-filesystemhandlerename-methods)
  - [Authors:](#authors)
  - [Participate](#participate)
  - [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [API Surface](#api-surface)
  - [Open Questions - Naming](#open-questions---naming)
- [Handling Moves to Other Directories](#handling-moves-to-other-directories)
  - [(1) Moving to a Non-Local File System](#1-moving-to-a-non-local-file-system)
  - [(2) Moving out of the OPFS](#2-moving-out-of-the-opfs)
  - [(3) Moving to a Directory on the Local File System](#3-moving-to-a-directory-on-the-local-file-system)
  - [Open Questions - Move](#open-questions---move)
    - [Should behavior be different when moving a file vs moving a directory?](#should-behavior-be-different-when-moving-a-file-vs-moving-a-directory)
    - [What should happen if the operation cannot be completed successfully?](#what-should-happen-if-the-operation-cannot-be-completed-successfully)
- [Implementation Notes](#implementation-notes)
- [Alternatives Considered](#alternatives-considered)
  - [Only support "rename" functionality](#only-support-rename-functionality)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Overview

Currently, the API does not support an efficient way to move or rename files or
directories. This requires creating a new file/directory, copying over data
(recursively, in the case of a directory), and removing the original. This
process is slow, error prone (e.g. disk full), and can require re-uploading
files in some cases (e.g. Google Drive files on Chrome OS).

We propose adding new `FileSystemHandle::rename()` and
`FileSystemHandle::move()` methods. `rename()` guarantees atomic moves of a file
or directory without the need to duplicate data. `move()` only offers atomic
moves of a file or directory if it is moved within the same file system, while
moves to non-local file systems will not be guaranteed to be atomic and may
involve duplicating data. See the [Open Questions](#open-questions---move).

# API Surface

We propose adding a `rename()` method and a `move()` method with two variations.

`rename()` will rename a file or directory. It is guaranteed to be atomic.

The first `move()` variation allows a user to specify a `destination_directory`
to move to, which may or may not be on the same file system. The name of the
original file or directory is retained. This is not guaranteed to be atomic,
since the destination directory may be on a different file system and/or may be
subject to other checks (e.g. Safe Browsing in Chrome).

The second `move()` variation allows a user to specify a `new_entry_name` as
well as a `destination_directory` to move to. The same (lack of) atomicity
guarantees apply here as with the first `move()` variation.

```
interface FileSystemHandle {
    ...

    Promise<void> rename(USVString new_entry_name);
    Promise<void> move(FileSystemDirectoryHandle destination_directory);
    Promise<void> move(FileSystemDirectoryHandle destination_directory, USVString new_entry_name);

    ...
};
```

## Open Questions - Naming

- Should we prefer `move()` to move verbose alternatives, such as `moveTo()` or
  `moveToDirectory()`?
- Should we bother guaranteeing (in the spec) atomicity to all handles `move()`d
  on the same file system, or only for `rename()`d handles?

# Handling Moves to Other Directories

There are three cases to consider when moving to other directories:

## (1) Moving to a Non-Local File System

Moving across file systems cannot be guaranteed to be atomic if the operation
cannot be completed successfully (e.g. network connection issues, lack disk
space, etc.). This may result in partial writes to the target directory.

Note that this approach is no worse than what is available currently.

## (2) Moving out of the OPFS

[#310](https://github.com/WICG/file-system-access/pull/310) exposes a strong use
case for fast, atomic moves to other directories on the same file system. Files
can be written efficiently in the Origin Private File System using an
`AccessHandle`, then efficiently moved to a directory of the user's choice.
However, writes using an `AccessHandle` are not subjected to Safe Browsing checks
in Chrome. Moving files out of the OPFS to a user's directory will require
running Safe Browsing checks on all moved files.

For example, moving a large directory across the OPFS -> file system boundary
means locking and performing checks (Safe Browsing in Chrome) for an unbounded
number of files.

## (3) Moving to a Directory on the Local File System

This is the most straightforward case. Every `rename()` falls in this case. The
operation should be atomic and fast, since no Safe Browsing checks need to be
performed. However, providing these guarantees for `move()` in only this case
may be confusing.

## Open Questions - Move

### Should behavior be different when moving a file vs. moving a directory?

- For a file:
  - Atomicity is only guaranteed in (2) and (3), though is likely true for (1).
  - Speed is only guaranteed in (3).
- For a directory:
  - Atomicity is only easily guaranteed in (3). Atomicity for (2) would be
    complex (likely requiring locking each file as an unbounded number of files
    are scanned) and may not be the best option.
  - Speed is only guaranteed in (3), but the implications of this for (1) and
    (2) is much greater than when only moving a file.

### What should happen if the operation cannot be completed successfully?

- Abort the operation
  - Least complexity for browser implementers, meaning less room for
    implementation-specific behavior.
  - Simplest interface.
  - Potential for errors not caught in development.
  - Apps need code to handle the cross-filesystem case.
- Abort the operation and attempt to remove partial writes
  - This may not be possible. Ex: loss of network connection.
  - Adds lots of complexity.
- Abort the operation and throw an error
  - Less complexity.
  - No performance cliff.
  - More flexibility in handling errors for the app.

# Implementation Notes

- Write permission needs to have been granted to both the source and destination
  directories.
- Invalid or unsafe `new_entry_name`s will result in a `Promise` rejection.
- All entries contained within a directory which is moved no longer point to an
  existing file/directory. We currently have no plans to explicitly update or
  invalidate these handles, which would otherwise be an expensive recursive
  operation.

# Alternatives Considered

## Only support "rename" functionality

Pros:
- No need to handle (or spec) the case where moves are non-atomic
- Simpler interface

Cons:
- We should allow for efficient moves out of the OPFS, as exposed in
  [#310](https://github.com/WICG/file-system-access/pull/310)
