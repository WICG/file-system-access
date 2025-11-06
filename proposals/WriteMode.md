# A "write"-only permission mode for the File System Access API

This explainer describes a proposed "write" permission mode for the [File System Access API](https://wicg.github.io/file-system-access/).

## Authors

* [Ming-Ying Chung](https://github.com/mingyc) - Google

## Introduction

The File System Access API allows web applications to read, write, and manage files on the local file system, with the user's permission. It provides a powerful way for web apps to interact with files in a way that was previously only possible for native applications.

Currently, the API defines two access modes for file system entries: `"read"` and `"readwrite"`. These modes are used when an application queries for or requests permission to a file or directory.

## Motivation

The problem with the current permission model is that operations requiring only file system modification, such as `FileSystemHandle.remove()`, are forced to request the broad `"readwrite"` permission. This is not ideal from a security perspective, as it grants more permission than necessary.

To address this, we propose a more granular, "write-only" permission mode.

## Proposed Solution

We propose adding a new `"write"` permission mode to the `FileSystemPermissionMode` enum in the File System Access API.

```javascript
enum FileSystemPermissionMode {
  "read",
  "readwrite",
  "write" // new
};
```

This new mode will allow applications to request only the permission to write to a file or directory, without also gaining permission to read from it. This follows the principle of least privilege, enhancing security.

Several existing methods that currently request `"readwrite"` permission will be updated to use the more appropriate permission level. Operations that only modify or create files will now be able to request `"write"` permission instead.

Key changes include:

*   **`FileSystemHandle.remove()`**: This operation will now only require `"write"` permission.
*   **`FileSystemFileHandle.createWritable()`**: Creating a writable stream to a file will now be a `"write"`-only operation.
*   **`FileSystemDirectoryHandle.getFileHandle({create: true})`**: Creating a new file handle will require `"write"` permission.
*   **`FileSystemDirectoryHandle.getDirectoryHandle({create: true})`**: Creating a new directory handle will require `"write"` permission.
*   **`FileSystemDirectoryHandle.removeEntry()`**: This operation will now only require `"write"` permission.

## Developer Experience

The introduction of the `"write"` mode is designed to be largely transparent to developers. It provides a clearer and more secure permission model without requiring changes to existing code.

While developers will not need to change their code to adopt this, the permission prompts shown to the user may become more specific, and the browser will enforce a stricter, more secure permission grant. Sites with existing already granted permissions should continue to work as before.

For example, when deleting a file:

```javascript
// Assume 'dirHandle' is a FileSystemDirectoryHandle the user has granted access to.

// In the old model, this would require "readwrite" permission.
// With this change, it will only require "write" permission on the directory.
await dirHandle.removeEntry('file-to-delete.txt');
```

Similarly, when creating a file and writing to it:

```javascript
// Assume 'dirHandle' is a FileSystemDirectoryHandle.
// This operation will now only require "write" permission.
const newFileHandle = await dirHandle.getFileHandle('new-file.txt', { create: true });

// Creating a writer will also fall under the "write" permission grant.
const writable = await newFileHandle.createWritable();
await writable.write('Some content');
await writable.close();
```

## Security and Privacy Considerations

The primary motivation for this change is to improve security. By introducing a "write"-only mode, we can ensure that web applications are granted the minimum level of privilege necessary to perform an operation. This aligns the API with the principle of least privilege.

## Stakeholder Feedback / Opposition

*   Gecko : No signals.
*   WebKit : No signals.
*   Web developers: No signals.
*   Google: This is a Google proposal.
