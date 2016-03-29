## Site Usage
### Read
Whenever a site is open, it can issue a request to `open()` a file. If `open` is passed without an argument, the browser will display its open UI. When the user selects a file, the browser returns the file name and a unique key to the site. The site can later call `open(key)` to open a file using the key issued on the initial open.

After opening a file, a site can read from the file any time the site is open. These subsequent reads do not require user interaction.

### Write
Whenever a site is open, it can issue a request to `write()`. Sites can save data to any file that they have the correct key for. If the site would like to change the name of a file, it must do so through the browser “save as…” dialog. If the site wants to create a new file, it must also use the browser “save as…” dialog

Whenever a site writes to a file, the browser should notify the user.

## Browser UI
No additional browser permission will be required. There will be browser UI that allows the user to open a file and to “save as…” a file. There will be a non modal warning shown by the browser whenever a site writes a file:

![Warning text when writing a file](img/write-warning.png?raw=true)

There will also be a non modal warning shown by the browser whenever a site reads from a file that has changed since the last time the site read the file:

![Warning text when reading a file](img/read-warning.png?raw=true)

The browser should also display all files a domain has access to in the domain's settings. Users will be allowed to revoke access to any of these files.

## Examples
__todo__ add examples

## Edge Cases
### File changes before save
There are a few models for native apps:

1. The file you are editing has changed. Please copy changes, close, then reopen.
2. Save anyways (i.e. overwrite)
3. Save as…
4. Merge changes

The browser doesn’t need to support these use cases explicitly, but the API should allow sites to provide all of these options. Write could return a `FileChanged` error whenever the site tried to write to a file that had changed in the background. This would enable use cases 1, 2, and 4. The `write()` method could also have a `force = True` option to allow option 3.

Alternatively the browser could offer a `FileChanged` event that included the key for what file changed. Sites could use this to listen for any open files that change. The browser would only fire the event for the site if it had access to the file in question.

__open issue__ do we want to do this? *Gut says no*
__open issue__ if we do this, should the event be fired for any file? Only ones opened that session?

### File renamed
- __open issue__ can the browser hold onto the file reference?
    - If yes, should the site be allowed to? *Gut says no*
	- If no, we should throw a `FileNotFound` exception
 
### File moved
Throw a `FileNotFound` exception

## Known weirdness
- Sites could communicate with each other via a shared file
- Could write a super cookie. __open issue__ Should we clear file access when CBD? *Talk to Mike West, ben wells, mek@, arichibald*
- Sites could bill “save as…” like “download…” but then be able to retain access to it *Could exacerbate super cookie issue from above*