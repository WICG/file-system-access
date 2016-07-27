# Example code

```javascript
// Open a db instance to save file references for later sessions
var db;
var request = indexedDB.open("WritableFilesDemo");
request.onerror = function(e) { console.log(e); }
request.onsuccess = function(e) { db = e.target.result; }

// Open the doc for the first time. Shows browser filepicker UI.
var doc;
File.open( { "writable": true } ).then( function(fh) {
    // Save the reference to open the file later
    var transaction = db.transaction(["filerefs"], "readwrite");
    var request = transaction.objectStore("filerefs").add( fh.ref );
    request.onsuccess = function(e) { console.log(e); }
    
    doc = fh;
}, function(e) {
    // Error. Maybe try to open the file readonly?
});

// Do useful things with the opened file.

// Save the file. Overwrites the current contents of doc.file
// to the local file
doc.save().then( function(e) {
    // Update UI to show successful save
}, function(e) {
    // Something went wrong. Check the event type.
    if( e.error == "FileModified" ) {
        // The file was changed somewhere else, force an overwrite
        // Could also offer more complex diff management.
        doc.save( {"force": true} );
    }
    else {
    	// Handle other kinds of errors
    }
}

// Retrieve a file you've opened before. Show's no filepicker UI.
// The browser can choose when to allow or not allow this open.
var file_id = "123"; // Some logic to determine which file you'd like to open
var transaction = db.transaction(["filerefs"], "readonly");
var request = transaction.objectStore("filerefs").get(file_id);
request.onsuccess = function(e) { 
    var ref = e.result;
    ref.open( { "writable": true } ).then( function(fh) {
        doc = fh;
    }, function(e) {
        // Handle errors with opening
    }
}
```

# Interface
This proposal adds a `File.open(options)` method. `options` is a JSON object specifying parameters about opening the file:

```webidl
dictionary FileOpenOptions {
    boolean writable = false;
    boolean multiple = false;
}

dictionary FileSaveOptions {
    boolean force = false;
}
```

`File.open()` returns a promise that resolves to a `FileHandle`: 

```webidl
interface FileHandle {
    readonly attribute FileReference ref;
    readonly attribute boolean writable;
    attribute File file;
    
    Promise<boolean> save( FileSaveOptions options ); 
}

interface FileReference {
    Promise<FileHandle> open( FileOpenOptions options );
}
```

The `FileReference` is used to reopen the file later without using browser file picker UI.

# Proposed security models
The spec has hooks for the browser to customize the security model:

- When to allow files to be opened as `writable`
- When to allow files to be re opened
- When to allow files to be saved

__Native app like__<br>
One proposed security model is to allow sites to always open files as `writable`. Files could be reopened on later visits to the site. If the file had been modified since the last time the site opened it, the browser could show non blocking UI notifying the user:

<div align="center">
    <img src="img/read-warning.png" alt="Warning text when reading a file" width="300px"></img>
</div>

When the file is written, the browser could show another non blocking UI notifying the user:

<div align="center">
    <img src="img/write-warning.png" alt="Warning text when writing a file" width="300px"></img>
</div>

Alternatively, the browser could choose not to show any UI. This would match the model that users expect from native apps, while restricting the site's actual access only to files that the user has explicitly granted access for.

__More restictive__<br>
Alternatively, the site could choose to only allow files to be `writable` if the user accepts some additional permission (such as a "Allow modifications" checkbox on the filepicker). The non blocking UI mentioned above could be a blocking UI that the user has to accept. The UI could be shown for *any* read, rather than just reads on modified files.

# More details
## Writable

A file can be opened as `readonly` or `readwrite`. This is specified using the `writable` attribute in the `options` parameter. User agents can decide when to allow files to be writable and when to have the promise reject. If a `FileHandle` object is `readonly`, it can be upgraded to `readwrite` by using the `ref.open()` method with `writable = true`:

```javascript
var writable_file;
readable_file.ref.open( { "writable": true } ).then( function(fh) {
    writable_file = fh;
});
```

## Reopening files
Sites can save the `ref` attribute of a `FileHandle` and use it to reopen the file later. Browsers can decide under what circumstances to allow a `FileReference` to reopen a `FileHandle`. For example, a user agent may decide to reject `ref.open()` if the file has been modified outside the domain.

## Save
The `save` method will write the current data in the `FileHandle`'s `data` attribute back to the local file. If `writable` is false, the promise will reject. If the file has been modified since the file was opened, `save()` will return a `FileModified` error. Calling `save( { "force": true } )` will suppress the `FileModified` error and overwrite the file.

## Backing file system
While this proposal is generally geared towards accessing native device files, it could theoretically provide access to files hosted in different mediums. Using the [ballista](https://github.com/chromium/ballista) API, writable-files could be used to interact with files housed in cloud based file systems.

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
