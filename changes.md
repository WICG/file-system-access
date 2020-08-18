This file enumerates the changes that were made to the API surface between the Origin Trial as shipped in Chrome 83,
and what is or will be available in Chrome 86.

## File Picker API entry points

**Before (in Origin Trial)**
```javascript
let file1 = await window.chooseFileSystemEntries(
    {type: 'open-file'});
let files = await window.chooseFileSystemEntries(
    {type: 'open-file', multiple: true});
let file2 = await window.chooseFileSystemEntries(
    {type: 'save-file'});
let dir = window.chooseFileSystemEntries(
    {type: 'open-directory'});
```

**After (in Chrome M86)**
```javascript
let [file1] = await window.showOpenFilePicker();
let files = await window.showOpenFilePicker({multiple: true});
let file2 = await window.showSaveFilePicker();
let dir = await window.showDirectoryPicker();
```

## Specifying accepted file types

**Before (in Origin Trial)**
```javascript
await window.chooseFileSystemEntries({
  accepts: [
    {
      description: 'Text Files',
      mimeTypes: ['text/plain', 'text/html'],
      extensions: ['txt', 'text', 'html', 'htm']
    },
    {
      description: 'Images',
      mimeTypes: ['image/*'],
      extensions: ['png', 'gif', 'jpeg', 'jpg']
    }
  ]
});
```

**After (in Chrome M86)**
```javascript
await window.showOpenFilePicker({
  types: [
    {
      description: 'Text Files',
      accept: {
        'text/plain': ['.txt', '.text'],
        'text/html': ['.html', '.htm']
      }
    },
    {
      description: 'Images',
      accept: {
        'image/*': ['.png', '.gif', '.jpeg', '.jpg']
      }
    }
  ]
});
```

## Determining if a handle is a file or directory

**Before (in Origin Trial)**
```javascript
if (handle.isFile) {
  // handle is a file
} else if (handle.isDirectory) {
  // handle is a directory
} else {
  // can't happen
}
```

**After (in Chrome M86)**
```javascript
switch (handle.kind) {
  case 'file':
    // handle is a file
    break;
  case 'directory':
    // handle is a directory
    break;
  default:
    // can't happen
}
```

## Getting children of a directory

**Before (in Origin Trial)**
```javascript
let file1 = parent.getFile('name');
let file2 = parent.getFile('name2', {create: true});
let dir1 = parent.getDirectory('dir1');
let dir2 = parent.getDirectory('dir2', {create: true});
```

**After (in Chrome M86)**
```javascript
let file1 = parent.getFileHandle('name');
let file2 = parent.getFileHandle('name2', {create: true});
let dir1 = parent.getDirectoryHandle('dir1');
let dir2 = parent.getDirectoryHandle('dir2', {create: true});
```

## Directory iteration

**Before (in Origin Trial)**
```javascript
for await (let handle of parent.getEntries()) {
  // Use handle and/or handle.name
}
```

**After (in Chrome M86)**
```javascript
for await (let handle of parent.values()) { /* ... */ }
for await (let [name, handle] of parent) { /* ... */ }
for await (let [name, handle] of parent.entries()) { /* ... */ }
for await (let name of parent.keys()) { /* ... */ }
```

## Changes to permissions

**Before (in Origin Trial)**
```javascript
await handle.queryPermission();
await handle.queryPermission({writable: false});
await handle.requestPermission();
await handle.requestPermission({writable: false});
```

**After (in Chrome M86)**
```javascript
await handle.queryPermission();
await handle.queryPermission({mode: 'read'});
await handle.requestPermission();
await handle.requestPermission({mode: 'read'});
```

**Before (in Origin Trial)**
```javascript
await handle.queryPermission({writable: true});
await handle.requestPermission({writable: true});
```

**After (in Chrome M86)**
```javascript
await handle.queryPermission({mode: 'readwrite'});
await handle.requestPermission({mode: 'readwrite'});
```

## Origin Private/Sandboxed File System

**Before (in Origin Trial)**
```javascript
let root = await FileSystemDirectoryHandle.getSystemDirectory(type: 'sandbox');
```

**After (in Chrome M86)**
```javascript
let root = await navigator.storage.getDirectory();
```
