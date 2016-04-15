# Opening a file
This proposal adds a `File.open(options)` method. `options` is a JSON object specifying parameters about opening the file:

```webidl
dictionary FileOpenOptions {
    boolean writable = false;
    boolean multiple = false;
}
```

`File.open()` returns a promise that resolves to a `FileHandle`: 

```webidl
interface FileHandle {
    readonly attribute FileReference ref;
    readonly attribute boolean writable;
    attribute File data;
    
    Promise<boolean> save(); 
}

interface FileReference {
    Promise<FileHandle> open();
}
```

The `FileReference` is used to reopen the file later without using browser file picker UI.

# Writable

A file can be opened as `readonly` or `readwrite`. This is specified using the `writable` attribute in the `options` parameter. User agents can decide when to allow files to be writable and when to have the promise reject.

# Reopening files
Sites can save the `ref` attribute of a `FileHandle` and use it to reopen the file later. Browsers can decide what circumstances to allow a `FileReference` to reopen a `FileHandle`. For example, a user agent may decide to reject `ref.open()` if the file has been modified outside the domain.

# Save
The `save` method will write the current data in the `FileHandle`'s `data` attribute back to the local file. If `writable` is false, the promise will reject.

# Example code

```javascript
File.open( { "writable": true } ).then( function(fh) {
    // Save the reference to open the file later
    db.put( fh.ref );
    
    // Do other things with fh.data
    
    // Save any changes back to the file
    fh.save().then( function(e) {
        console.log("saved!");
    }).catch( function(e) {
    	console.log("failed to save.");
    });
}).catch( function(e) {
    // Could attempt opening the file readonly
});
```

# Browser UI
No additional browser permission / UI is required beyond the "File select" and "Save as.." UIs. User agents can choose to show additional UI as desired. For example, there could be a non modal warning shown by the browser whenever a site writes a file:

<div align="center">
    <img src="img/write-warning.png" alt="Warning text when writing a file" width="300px"></img>
</div>

There could also be a non modal warning shown by the browser whenever a site reads from a file that has changed since the last time the site read the file:

<div align="center">
    <img src="img/read-warning.png" alt="Warning text when reading a file" width="300px"></img>
</div>

The browser should display all files a domain has access to in the domain's settings. Users will be allowed to revoke access to any of these files.

# Edge Cases
## File changes before save
There are a few models for native apps:

1. The file you are editing has changed. Please copy changes, close, then reopen.
2. Save anyways (i.e. overwrite)
3. Save as…
4. Merge changes

The browser doesn’t need to support these use cases explicitly, but the API should allow sites to provide all of these options. Write could return a `FileChanged` error whenever the site tried to write to a file that had changed in the background. This would enable use cases 1, 2, and 4. The `write()` method could also have a `force = True` option to allow option 3.

Alternatively the browser could offer a `FileChanged` event that included the key for what file changed. Sites could use this to listen for any open files that change. The browser would only fire the event for the site if it had access to the file in question.

__open issue__ do we want to do this? *Gut says no*
__open issue__ if we do this, should the event be fired for any file? Only ones opened that session?

## File renamed
- __open issue__ can the browser hold onto the file reference?
    - If yes, should the site be allowed to? *Gut says no*
	- If no, we should throw a `FileNotFound` exception
 
## File moved
Throw a `FileNotFound` exception

# Known weirdness
- Sites could communicate with each other via a shared file
- Could write a super cookie. __open issue__ Should we clear file access when CBD? *Talk to Mike West, ben wells, mek@, arichibald*
- Sites could bill “save as…” like “download…” but then be able to retain access to it *Could exacerbate super cookie issue from above*
