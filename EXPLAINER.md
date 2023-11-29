# What is all this?

At a high level what we're providing is several bits:

1. A modernized version of the existing (but not really standardized)
   [`Entry`](https://www.w3.org/TR/2012/WD-file-system-api-20120417/#idl-def-Entry)
   API in the form of new (names TBD) `FileSystemFileHandle` and
   `FileSystemDirectoryHandle` interfaces (see also the [wicg entries-api](https://wicg.github.io/entries-api/),
   a read-only and slightly more standardized subset of this same API).
2. A modernized version of the existing (and also not really standardized)
   [`FileWriter`](https://dev.w3.org/2009/dap/file-system/file-writer.html#the-filewriter-interface)
   interface.
3. Various entry points to get a handle representing a limited view of the
   local file system. I.e. either via a file picker, or to get access to
   certain well known directories. Mimicking things such as chrome's
   [`chrome.fileSystem.chooseEntry`](https://developer.chrome.com/apps/fileSystem#method-chooseEntry) API.

## Use-Cases

In native applications, there are common file access patterns that we aim to address with this API.

### Single-file Editor
1. Open a file from the user's file system
1. Edit the file and save the changes back to the file system
1. Open another file in the same manner described above
1. Auto-save any changes to the files in the browsing session
1. The files can be opened in any native or web applications concurrently
1. Changes to the files on disk, made in any native or other web application, are accessible
1. Access the files with the same access in future browsing sessions
1. Create a new file in the editor
1. Auto-save changes to the new file in a temporary location, even before the user has picked a file name/location

### Multi-file Editor
1. Open a directory that contains many files and sub-directories, represented hierarchically
1. Find and edit multiple files and save the changes back to the file system
1. Auto-save any further changes to the files in the browsing session
1. The files can be opened in any native or web applications concurrently
1. Changes to the files on disk, made in any native or other web applications, are accessible
1. Access the files with the same access in future browsing sessions
1. New files in the directory tree, that were not present at the time the root directory was
   opened, created in any native or other web application, are accessible

### File Libraries
1. Open one or more directories that contain many files and sub-directories
1. Changes to the files on disk, made in any native or other web applications, are accessible
1. Access the files with the same access in future browsing sessions
1. New files in the directory tree, that were not present at the time the root directory was
   opened, created in any native or other web application, are accessible
1. When the user chooses to do some work, access one or more of those files

## Goals

The main overarching goal here is to increase interoperability of web applications
with native applications, specifically where it comes to being able to operate on
the native file system.

Traditionally the file system is how different apps collaborate and share data on
desktop platforms, but also on mobile there is generally at least some sort of
concept of a file system, although it is less prevalent there.

Some example applications of the API we would like to address:

* A simple "single file" editor. Also possible integration with a "file-type
  handler" kind of API. Things like (rich) text editors, photo editors, etc.

* Multi-File editors. Things like IDEs, CAD style applications, the kind of apps
  where you work on a project consisting of multiple files, usually together in
  the same directory.

* Apps that want to work with "libraries" of certain types of files. I.e. photo
  managers, music managers/media players, or even drawing/publishing apps that
  want access to the raw font files for all fonts on the system.

But even though we'd like to design the API to eventually enable all these use
cases, initially we'd almost certainly be shipping a very limited API surface
with limited capabilities.

Additionally we want to make it possible for websites to get access to some
directory without having to first prompt the user for access. This enables use
cases where a website wants to save data to disk before a user has picked a
location to save to, without forcing the website to use a completely different
storage mechanism with a different API for such files. It also makes it easier
to write automated tests for code using this API.

## Non-goals

At least for now out of scope is access to the full file system, subscribing to
file change notifications, probably many things related to file metadata (i.e.
marking files as executable/hidden, etc). Also not yet planning to address how
this new API might integrate with `<input type=file>`.

# Example code

```javascript
// Show a file picker to open a file.
const [file_ref] = await self.showOpenFilePicker({
    multiple: false,
    types: [{description: 'Images', accept: {'image/*': ['.jpg', '.gif', '.png']}}],
    suggestedStartLocation: 'pictures-library'
});
if (!file_ref) {
    // User cancelled, or otherwise failed to open a file.
    return;
}

// Read the contents of the file.
const file_reader = new FileReader();
file_reader.onload = async (event) => {
    // File contents will appear in event.target.result.  See
    // https://developer.mozilla.org/en-US/docs/Web/API/FileReader/onload for
    // more info.

    // ...

    // Write changed contents back to the file. Rejects if file reference is not
    // writable. Note that it is not generally possible to know if a file
    // reference is going to be writable without actually trying to write to it.
    // For example, both the underlying filesystem level permissions for the
    // file might have changed, or the user/user agent might have revoked write
    // access for this website to this file after it acquired the file
    // reference.
    const writable = await file_ref.createWritable();
    await writable.write(new Blob(['foobar']));
    await writable.seek(1024);
    await writable.write(new Blob(['bla']));

    // |writable| is also a WritableStream, so you can for example pipe into it.
    let response = await fetch('foo');
    await response.body.pipeTo(writable);

    // pipeTo by default closes the destination pipe, otherwise an explicit
    // writable.close() call would have been needed to persist the written data.
};

// file_ref.file() method will reject if site (no longer) has access to the
// file.
let file = await file_ref.file();

// readAsArrayBuffer() is async and returns immediately.  |file_reader|'s onload
// handler will be called with the result of the file read.
file_reader.readAsArrayBuffer(file);
```

Also possible to store file references in IDB to re-read and write to them later.

```javascript
// Open a db instance to save file references for later sessions
let db;
let request = indexedDB.open("WritableFilesDemo");
request.onerror = function(e) { console.log(e); }
request.onsuccess = function(e) { db = e.target.result; }

// Show file picker UI.
const [file_ref] = await self.showOpenFilePicker();

if (file_ref) {
    // Save the reference to open the file later.
    let transaction = db.transaction(["filerefs"], "readwrite");
    let request = transaction.objectStore("filerefs").add( file_ref );
    request.onsuccess = function(e) { console.log(e); }

    // Do other useful things with the opened file.
};

// ...

// Retrieve a file you've opened before. Show's no filepicker UI, but can show
// some other permission prompt if the browser so desires.
// The browser can choose when to allow or not allow this open.
let file_id = "123"; // Some logic to determine which file you'd like to open
let transaction = db.transaction(["filerefs"], "readonly");
let request = transaction.objectStore("filerefs").get(file_id);
request.onsuccess = async function(e) {
    let ref = e.result;

    // Permissions for the handle may have expired while the handle was stored
    // in IndexedDB. Before it is safe to use the handle we should request at
    // least read access to the handle again.
    if (await ref.requestPermission() != 'granted') {
      // No longer allowed to access the handle.
      return;
    }

    // Rejects if file is no longer readable, either because it doesn't exist
    // anymore or because the website no longer has permission to read it.
    let file = await ref.file();
    // ... read from file

    // Rejects if file is no longer writable, because the website no longer has
    // permission to write to it.
    let file_writer = await ref.createWritable();
    // ... write to file_writer
}
```

The fact that handles are serializable also means you can `postMessage` them around:

```javascript
// In a service worker:
self.addEventListener('some-hypothetical-launch-event', async (e) => {
  // e.file is a FileSystemFileHandle representing the file this SW was launched with.
  let win = await clients.openWindow('bla.html');
  if (win)
    win.postMessage({openFile: e.file});
});

// In bla.html
navigator.serviceWorker.addEventListener('message', e => {
  let file_ref = e.openFile;
  // Do something useful with the file reference.
});
```

Also possible to get access to an entire directory.

```javascript
const dir_ref = await self.showDirectoryPicker();
if (!dir_ref) {
    // User cancelled, or otherwise failed to open a directory.
    return;
}
// Read directory contents.
for await (const [name, entry] of dir_ref) {
    // entry is a FileSystemFileHandle or a FileSystemDirectoryHandle.
    // name is equal to entry.name
}

// Get a specific file.
const file_ref = await dir_ref.getFile('foo.js');
// Do something useful with the file.

// Get a subdirectory.
const subdir = await dir_ref.getDirectory('bla', {create: true});

// No special API to create copies, but still possible to do so by using
// available read and write APIs.
const new_file = await dir_ref.getFile('new_name', {create: true});
const new_file_writer = await new_file.createWritable();
await new_file_writer.write(await file_ref.getFile());
await new_file_writer.close();

// Or using streams:
const copy2 = await dir_ref.getFile('new_name', {create: true});
(await file_ref.getFile()).stream().pipeTo(await copy2.createWritable());
```

You can also check if two references reference the same file or directory (or at
least reference the same path), as well as lookup the relative path of an entry
inside another directory you have access to.

If for example an IDE has access to a directory, and uses that to display a tree
view of said directory, this can be useful to be able to highlight a file in that
tree, even if the file is opened through a new file picker by opening an existing
file or saving to a new file.

```javascript
// Assume we at some point got a valid directory handle.
const dir_ref = await self.showDirectoryPicker();
if (!dir_ref) return;

// Now get a file reference by showing another file picker:
const file_ref = await self.showOpenFilePicker();
if (!file_ref) {
    // User cancelled, or otherwise failed to open a file.
    return;
}

// Check if file_ref exists inside dir_ref:
const relative_path = await dir_ref.resolve(file_ref);
if (relative_path === null) {
    // Not inside dir_ref
} else {
    // relative_path is an array of names, giving the relative path
    // from dir_ref to the file that is represented by file_ref:
    let entry = dir_ref;
    for (const name of relative_path) {
        entry = await entry.getChild(name);
    }

    // Now |entry| will represent the same file on disk as |file_ref|.
    assert await entry.isSameEntry(file_ref) == true;
}
```

To get access to a writable directory without having to ask the user for access,
we also provide a "sandboxed" file system. Files in this directory are not
exposed to native applications (or other web applications), but instead are
private to the origin. Storage in this sandboxed file system is subject to
quota restrictions and eviction measures like other web exposed storage mechanisms.

```javascript
const sandboxed_dir = await self.getSandboxedFileSystem();

// The website can freely create files and directories in this directory.
const cache_dir = await sandboxed_dir.getDirectory('cache', {create: true});
for await (const entry of cache_dir.values()) {
    // Do something with entry.
};

const new_file = await sandboxed_dir.getFile('Untitled 1.txt', {create: true});
const writer = await new_file.createWritable();
writer.write("some data");
await writer.close();
```

And perhaps even possible to get access to certain "well-known" directories,
without showing a file picker, i.e. to get access to all fonts, all photos, or
similar. Could still include some kind of permission prompt if needed.

```javascript
const font_dir = await FileSystemDirectoryHandle.getSystemDirectory({type: 'fonts'});
for await (const entry of font_dir.values()) {
    // Use font entry.
};
```

# Proposed security models

By far the hardest part for this API is of course going to be the security model
to use. The API provides a lot of scary power to websites that could be abused
in many terrible ways. There are both major privacy risks (websites getting
access to private data they weren't supposed to have access to) as well as
security risks (websites modifying executables, installing viruses, encrypting
the users data and demanding ransoms, etc). So great care will have to be taken
to limit how much damage a website can do, and make sure a user understands what
they are giving a website access to. Persistent access to a file could also be
used as some form of super-cookie (but of course all access to files should be
revoked when cookies/storage are cleared, so this shouldn't be too bad).

The primary entry point for this API is a file picker (i.e. a chooser). As such
the user always is in full control over what files and directories a website has
access to. Furthermore every access to the file (either reading or writing)
after a website has somehow gotten a handle is done through an asynchronous API,
so browser could include more prompting and/or permission checking at those
points. This last bit is particularly important when it comes to persisting
handles in IndexedDB. When a handle is retrieved later a user agent might want
to re-prompt to allow access to the file or directory.

Other parts that can contribute to making this API as safe as possible for users
include:

## Limiting access to certain directories

For example it is probably a good idea for a user agent to not allow the user
to select things like the root of a filesystem, certain system directories,
the users entire home directory, or even their entire downloads directory.

## Limiting write access to certain file types

Not allowing websites to write to certain file types such as executables will
limit the possible attack surface.

## Other things user agents come up with

# Staged implementation

At least in chrome we're not planning on implementing and shipping all this at
once. Quite likely an initial implementation will for example not include any of
the transferability/serializability and thus retainability of references. We do
want to add those feature in later iterations, so we're designing the API to
support them and hope to come up with a security model that can be adapted to
support them, but explicitly not supporting everything initially should make
things slightly less scary/dangerous and give more time to figure out how to
expose the really powerful bits.
