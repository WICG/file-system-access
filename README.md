# File System Access API
View proposals in the [EXPLAINER](EXPLAINER.md) and the [spec](https://wicg.github.io/file-system-access/).

See [changes.md](changes.md) for a list of changes that were made to the API surface between what was available as an Origin Trial in Chrome 83 through Chrome 85, and what will be shipped in Chrome 86.

## Problem
Today, if a web site wants to create experiences involving local files (document editor, image compressor, etc.) they are at a disadvantage to native apps. A web site must ask the user to reopen a file every time they want to edit it. After opening, the site can only save changes by re downloading the file to the Downloads folder. A native app, by comparison, can maintain a most recently used list, auto save, and save files anywhere the user wants.

## Use cases
- Open local file to read
- Open local file to edit and then save
- Open local file to edit with auto save
- Create and save a new file
- Delete an existing file
- Read meta data about files

## Workarounds
- [FileSaver.js](https://github.com/eligrey/FileSaver.js/) polyfills `saveAs()` from the [W3C File API](https://www.w3.org/TR/FileAPI/), but files open in a new window instead of downloading on Safari 6.1+ and iOS.
- In Edge, Firefox, and Chrome developers can:
	- Create a fake anchor element (`var a = document.createElement('a')`)
	- Set `download` to the desired filename (`a.download = 'file.txt'`)
	- Set `href` to a data URI or Blob URL (`a.href = URL.createObjectURL(blob)`)
	- Fake a click on the anchor element (`a.click()`)
	- Clean up if necessary (`URL.revokeObjectURL(a.href)`)

  This is also the approach taken in the
  [browser-fs-access](https://github.com/GoogleChromeLabs/browser-fs-access)
  support library.
- Setting `window.location` to `'data:application/octet-stream' + data_stream`.
- Hidden Flash controls to display a “save as” dialog.

These methods are clunky and only support “save as” (and depending on the UA may automatically appear in Downloads without prompting the user for location). They do not support most recently used lists, auto save, save, or deleting a file.
