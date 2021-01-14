# Suggested file name and location

## Authors:

* Marijn Kruisselbrink

## Participate

* [Issue tracker](https://github.com/WICG/file-system-access/issues)

## Introduction

When initially shipping the File System Access API we shipped a bare minimum API
surface for showing file and directory pickers. We're now proposing adding a
couple of frequently requested (and commonly supported in other file picker
APIs) options to let websites provide a couple more opportunities to influence
the behavior of the file and directory pickers.

In particular this explainer addresses letting websites give suggestions for the
name and location of files and directories that are being saved or loaded.

## Goals and Use Cases

The overarching goal here is to provide websites the opportunity to provide the
file or directory picker implementation suggestions for both the initial
directory to show in the picker, and (in the case of save dialogs) the file name
to save a file as. Specifically to help address the following use cases:

### Save an existing file in a different directory

1. User opens a file from their local file system, or from some cloud provider.
2. User then picks the "Save As" option from the web application.
3. User saves the file in a different directory, but using the name the file
   already had (without having to re-type the name).

### Save a file using a different name in the same directory

1. Users opens a file from their local file system.
2. User then picks the "Save As" option from the web application.
3. User saves the file using a different name, in the directory the original
   file exists (without having to re-navigate to that directory).

### Prompt a user to open a image (or video, or audio file)

1. User picks "Insert image" in a web application
2. A file picker pops up starting out in the users "Pictures" folder.

### Remember different last-used directories for different purposes

1. User opens a document in a web application from directory A.
2. User inserts an image into the document, by selecting a file from directory B.
3. User opens a different document, from directory A without having to navigate
   to directory A again.
4. Further image insertion operations should have their file pickers start out
   in the directory of the last image insertion as well.

## Non-goals

All the above functionality is intended as hints/suggestions from websites to
the file picker implementation to improve the end user experience. However
implementations should not be required to always follow these suggestions. If
for example a suggested file name provided by the website is deemed too
dangerous to be allowed, implementations should be free to ignore the suggestion
(or adjust the suggested file name to not be dangerous anymore).

## Proposed API

### Specifying suggested file name to save as

```javascript
const file_ref = await self.showSaveFilePicker({
  suggestedName: 'README.md',
  types: [{
    description: 'Markdown',
    accept: {
      'text/markdown': ['.md'],
    },
  }],
});
```

### Specifying starting directory based on an existing handle.

There are two cases where it is useful to be able to specify a starting
directory based on an existing handle. The first of these if a website wants to
let a user pick a sibling to an existing file or directory. For this purpose we
propose adding a `startInParentOf` option, that can be passed to any of the file
and directory picker methods:

```javascript
const existing_handle = /* some FileSystemHandle, could be either a file or a directory */;
const file_ref = await self.showSaveFilePicker({
  startInParentOf: existing_handle
});
```

The second situation is where a website wants to prompt the user to open a file
in a directory it already has a handle to. For example if a website has a
"project" directory open, it might make sense for a "Save" dialog to start out
in that directory. For this purpose we propose additionally adding a `startIn`
option:

```javascript
const existing_dir_handle = /* some FileSystemDirectoryHandle */;
const file_ref = await self.showSaveFilePicker({
  startIn: existing_dir_handle
});
```

### Specifying a well-known starting directory

To support saving files in or opening files from certain well-known directories,
we also propose allowing passing certain string values to `startIn` to represent
these well-known directories. This would look something like:

```javascript
const file_ref = await self.showOpenFilePicker({
  startIn: 'pictures'
});
```

The possible values for `startIn` would be:

- `desktop`: The user's Desktop directory, if such a thing exists.
- `documents`: Directory in which documents created by the user would typically be stored.
- `downloads`: Directory where downloaded files would typically be stored.
- `home`: The user's Home directory.
- `music`: Directory where audio files would typically be stored.
- `pictures`: Directory where photos and other still images would typically be stored.
- `videos`: Directory where videos would typically be stored.

### Distinguishing the "purpose" of different file picker invocations.

Currently the Chrome implementation of the File System Access API remembers the
last used directory on a per-origin basis. To allow websites to remember
different directories for file pickers that are used for different purposes
(i.e. opening documents, exporting to other formats, inserting images) we
propose adding an `id` option to the file and directory picker methods. If an
`id` is specified the file picker implementation will remember a separate last
used directory for pickers with that same `id`:

```javascript
const file_ref1 = await self.showSaveFilePicker({
  id: 'fruit'
});

const file_ref2 = await self.showSaveFilePicker({
  id: 'flower'
});
```

## Detailed design discussion

### Interaction of `suggestedName` and accepted file types

It is possible for `suggestedName` to be inconssitent with the file types a
website declared as being the accepted file types for a file picker. There are
a couple of cases worth highlighting here:

1. If `suggestedName` ends in a suffix that is specified for one of the accepted
   file types, the file picker should default to that file type.

2. Otherwise, if `excludeAcceptAllOption` is false (or no explicit accepted file
   types are provided), the "all files" options should be the default selected
   file type in the file picker.

3. Finally, if neither of these are the case (i.e. the suggested file name does
   not match any of the file types accepted by the dialog), the implementation
   should behave as if `excludeAcceptAllOption` was set to false, and default
   to that option.

#### Considered alternatives

An alternative to 3. would be to reject if no accepted file types match the
suggested file name.

Another alternative to 3. would be to append a suffix from the first accepted
file type to the suggested file name (or replace the extension from the
suggested file name with one that is accepted). This seems less desirable than 3
because this would mean that specifying an extension in the `suggestedName` is
optional as long as `excludeAcceptAllOption` is set to true, but then later
changing `excludeAcceptAllOption` to false would suddenly change behavior and
possibly break existing API usage.

### Interaction between `startIn`, `startInParentOf` and `id`

All these attributes influence what directory the file picker should start with,
as such it isn't immediately obvious what should happen if all are provided.

Our proposal is for it to be an error to provide both `startIn` and
`startInParentOf`. It should not be an error to provide both `startIn*` and
`id`. If both are provided, `startIn*` will specify what directory the file
picker should start out it, and the ultimately picked directory will be recorded
as the last selected directory for the given `id`, such that future invocations
with that `id` but no `startIn*` value will use the directory.

#### Considered alternatives

We could reject if both a `startIn` option and `id` are provided, to ensure only
one option controls which directory to start in. There don't see to be many
downsides to allowing both though, as recording the last used directory for
future invocations of a file picker without a `startIn` option still seems
useful.

We could also not reject if both `startIn` and `startInParentOf` are provided.
This might have benefits if browsers only implement one of them. Websites
could still get some of the behavior they want without having to do tricky
feature detection to figure out which feature is and isn't supported. On the
other hand, allowing both would mean we'd have to specify a ordering between
them, and unless the ordering we specify happens to match the fallback behavior
a website wants, websites would still need to feature detect. Only allowing one
is simpler to reason about.

We could allow file handles to be passed to `startIn` as well. This could behave
as if the file handle was passed to `startInParentOf`, while the name of the
file was passed as `suggestedName`. This would perhaps make the Save As use
cases somewhat simpler, but is also redundant with what is already possible with
the proposed API. Furthermore we would then have to specify what happens if both
a `startIn` file, and an explicit `suggestedName` is specified.

Finally, rather than having both `startIn` and `startInParentOf`, we could have
a single option that accepts both a file or a directory handle. If a file is
passed in, it behaves as the above described `startInParentOf`, but if a
directory is passed in it behaves as `startIn`. This means we would lose the
ability to start a picker in the parent directory of a given directory handle,
but that is also one of the more edge use cases, and only having a single option
to deal with otherwise simplifies the API somewhat.

## Stakeholder Feedback / Opposition

* Chrome : Positive, authoring this explainer
* Gecko : No signals
* WebKit : No signals
* Web developers : Positive, frequently requested

## References & acknowledgements

Many thanks for valuable feedback and advice from:

Austin Sullivan
