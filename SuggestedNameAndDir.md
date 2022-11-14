# Suggested file name and location

## Authors:

* Marijn Kruisselbrink

## Participate

* [Issue tracker](https://github.com/WICG/file-system-access/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals and Use Cases](#goals-and-use-cases)
  - [Save an existing file in a different directory](#save-an-existing-file-in-a-different-directory)
  - [Save a file using a different name in the same directory](#save-a-file-using-a-different-name-in-the-same-directory)
  - [Open a file from the same directory as a currently open file or project](#open-a-file-from-the-same-directory-as-a-currently-open-file-or-project)
  - [Prompt a user to open a image (or video, or audio file)](#prompt-a-user-to-open-a-image-or-video-or-audio-file)
  - [Remember different last-used directories for different purposes](#remember-different-last-used-directories-for-different-purposes)
- [Non-goals](#non-goals)
- [Proposed API](#proposed-api)
  - [Specifying suggested file name to save as](#specifying-suggested-file-name-to-save-as)
  - [Specifying starting directory based on an existing handle.](#specifying-starting-directory-based-on-an-existing-handle)
  - [Specifying a well-known starting directory](#specifying-a-well-known-starting-directory)
  - [Distinguishing the "purpose" of different file picker invocations.](#distinguishing-the-purpose-of-different-file-picker-invocations)
- [Detailed design discussion](#detailed-design-discussion)
  - [Interaction of `suggestedName` and accepted file types](#interaction-of-suggestedname-and-accepted-file-types)
    - [Considered alternatives](#considered-alternatives)
  - [Interaction between `startIn` and `id`](#interaction-between-startin-and-id)
    - [Considered alternatives](#considered-alternatives-1)
  - [Start in directory vs start in parent of directory](#start-in-directory-vs-start-in-parent-of-directory)
  - [Security considerations](#security-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

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

### Open a file from the same directory as a currently open file or project

1. User is editing a file from their local file system.
2. User then picks the "Open" option from the web application to open another file.
3. User can pick files from the same directory the currently open file is in
   without having to re-navigate to that directory.

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

We propose adding a `startIn` option, that can be either a file or a directory
handle. If the passed in handle is a file handle, the picker will start out in
the parent directory of the file, while if the passed in handle is a directory
handle the picker will start out in the passed in directory itself.

Additionally, in a save file picker, if `startIn` is a file, and no explicit
`suggestedName` is also passed in, the name from the passed in file handle
will be treated as if it was passed as `suggestedName`.

This lets you use `startIn` for typical "Save as" UI flows:

```javascript
async function saveFileAs(file_handle) {
  return await self.showSaveFilePicker({
    startIn: file_handle
  });
}
```

And is also useful for cases where you a user is likely to want to open a file
from the same directory as a currently open file or directory:

```javascript
async function openFileFromDirectory(project_dir) {
  return await self.showOpenFilePicker({
    startIn: project_dir
  });
}

// Used when prompting the user to open a new file, starting out in the
// directory containing a currently open file.
async function openFileFromDirectoryContainingFile(open_file) {
  return await self.showOpenFilePicker({
    startIn: open_file
  });
}

// Used for example in a flow where a website wants to prompt the user to open
// the directory containing the currently opened file (for example for file
// formats that contain relative paths to other files in the same directory).
async function openDirectoryContainingFile(open_file) {
  return await self.showDirectoryPicker({
    startIn: open_file
  });
}
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
- `music`: Directory where audio files would typically be stored.
- `pictures`: Directory where photos and other still images would typically be stored.
- `videos`: Directory where videos/movies would typically be stored.

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

// Picker should start out in the directory `file_ref1` was picked from.
const file_ref3 = await self.showOpenFilePicker({
  id: 'fruit'
});

```

## Detailed design discussion

### Interaction of `suggestedName` and accepted file types

It is possible for `suggestedName` to be inconsistent with the file types a
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

### Interaction between `startIn` and `id`

Both these attributes influence what directory the file picker should start with,
as such it isn't immediately obvious what should happen if all are provided.

Our proposal is for it to not be an error to provide both `startIn` and
`id`. If a well-known directory is provide for `startIn`, `id` will take
precedence. That means that if a previously recorded path for the same `id`
exists, it will be used as starting directory, and only if no such path is
known will `startIn` be used.

On the other hand if `startIn` is a file or directory handle, it will take
precedence over `id`. In this case `startIn` specifies what directory the
file picker should start out in, and the ultimately picked directory will be
recorded as the last selected directory for the given `id`, such that future
invocations with that `id` but no `startIn` will use the directory.

#### Considered alternatives

We could reject if both a `startIn` option and `id` are provided, to ensure only
one option controls which directory to start in. There don't see to be many
downsides to allowing both though, as recording the last used directory for
future invocations of a file picker without a `startIn` option still seems
useful.

We could also have either `id` or `startIn` always take precedence over the
other. In some ways this might be less confusing, as it would always be clear
which one takes precedence, without having to know the type of what is passed to
`startIn`. We've chosen not to do so though. Using a well-known directory only
as a fallback mechanism for the `id` based directory matches what other similar
APIs do. Simultanously if a website explicitly specifies a concrete directory to
open the picker in by passing a file or directory handle to `startIn` we also
want to respect that. Hence the precedence depending on the type of the `startIn`
option.

### Start in directory vs start in parent of directory

An earlier version of this document had separate `startIn` and `startInParentOf`
options. This would enable websites to not only open file or directory pickers
in the same directory as a passed in directory handle, but also in the parent
directory of such a directory (without needing to have access to a handle for
the parent directory).

Having a single `startIn` option results in a simpler and easier to explain and
understand API, whil still supporting all the major use cases. Websites will be
able to start file or directory pickers in any directory they have a handle to,
as well as any directory for which they have a handle to a file in said
directory. The only hypothetical use case that isn't covered by this is for
cases where the website wants the user to select a sibling directory to a
previously selected directory, but we're not aware of any concrete use cases
where that would be beneficial.

### Security considerations

As mentioned in the non-goals section, all these attributes should be considered
suggestions or hints. This is particularly relevant for the `suggestedName`
option. The same concerns around writing to files ending in `.lnk` or `.local`
as mentioned in https://wicg.github.io/file-system-access/#privacy-wide-access
also apply to letting the website suggest a user save files with these names.
User agents should do similar sanitization as to how the `download` attribute
of `<a>` tags is processed in https://html.spec.whatwg.org/multipage/links.html#as-a-download.

## Stakeholder Feedback / Opposition

* Chrome : Positive, authoring this explainer
* Gecko : No signals
* WebKit : No signals
* Web developers : Positive, frequently requested (See #85, #144, #94 and #80.)

## References & acknowledgements

Many thanks for valuable feedback and advice from:

Austin Sullivan, Thomas Steiner
