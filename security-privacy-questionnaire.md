https://www.w3.org/TR/2019/NOTE-security-privacy-questionnaire-20190523/

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature exposes files and directories the user explicitly selects to share with web sites with those web sites. This feature doesn't expose any more information than is already exposed via `<input type=file>` and `<input type=file webkitdirectory>` today.

However this feature does make it possible for browsers to extend the time web sites have access to this information. I.e. permission grants and handles received through this API could be persisted giving web sites access to the same files/directories on disk when a user later returns to the website (or when a service worker for the origin is processing an event). At least for the Chrome implementation, we're only planning on having these grants persist for installed PWAs. On the drive-by web, access will only be as long as the web site is open, requiring re-prompting in subsequent visits.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes, we're only exposing files and directories explicitly selected by the user. Of course, web sites could ask for access to a directory when all it needs is access to some files, but the same is already true today. At least as far as exposing information to web sites is concerned, this API doesn't expose any more information than existing APIs do today.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

No data is exposed without the user explicitly choosing what files or directories to expose to the web site.

### 2.4. How does this specification deal with sensitive information?

No data is exposed without the user explicitly choosing what files or directories to expose to the web site, so only sensitive data that the user explicitly decides to share via this API will be shared. Furthermore, this API is only exposed in secure contexts, and third-party iframes (i.e. iframes that are cross origin from the top-level frame) won't be able to show pickers or permission prompts and can only access data they were already granted access to from a top-level same origin frame.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

Yes, this specification lets websites store handles they've gotten access to (via a file or directory picker) in IndexedDB. User agents could also persist the permission grants that go with these handles, but at least in the Chrome implementation, these permission grants will only be persistent for installed PWAs. The drive-by web will only have enough state to allow it to re-prompt for access, but the access itself won't be persistent.

Furthermore, the user will be able to clear storage (storage is file handles in IndexedDB) and/or revoke permissions to clear the state that was persisted, similarly to how other permissions work.

Websites can also store any state they like in files they get write access to via this API. Since files written to using this API are considered to be data owned by the user, not by the application, this state would not be cleared when clearing browser data. However access to this state would be removed. If a user later picks the same files or directories again to give the website access to them, the websites will regain access to whatever state they persisted.

Additionally, user agents could also choose to persist the last directory a file was picked from using this API on a per origin (and per purpose via the `FilePickerOption.id` option) basis. This state will not be exposed to the website, it only changes the UI that is presented to the user. A website will have no way of telling if a user picked a file in a certain directory because of this state or because the user manually navigated to the directory.

The `getUniqueId()` method will require a user agent to persist information (e.g. a salt) to provide unique identifiers for handles which are stable across browsing sessions, but which are invalidated once the user clears storage for the site. This state will not be exposed to the website.

The `getCloudIdentifiers()` method will request identifiers for a given file/directory handle from a cloud storage provider's sync client (usually an external service/application) and forward these to the requesting website. These identifiers may be stable and cannot be invalidated as part of this API.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

Anything that exists on disk in files could be exposed by the user to the web. However, user agents are encouraged to maintain a block list of certain directories with particularly sensitive files, and thus somewhat restrict which files and directories the user is allowed to select. For example, things like Chrome's "Profile" directory, and other platform configuration data directories are likely going to be on this block list.

The `getCloudIdentifiers()` method will request identifiers for a given file/directory handle from a cloud storage provider's sync client (usually an external service/application) and forward these to the requesting website. 
Therefore, the requesting website can enumerate all those sync clients present on the user's machine that sync a file/directory the website has a handle to.

### 2.7. Does this specification allow an origin access to sensors on a user’s device

No, unless a device exposes such sensors as files or directories. User agents are encouraged to block access to such files or directories (for example `/dev` on linux like systems).

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

The data this specification lets a user expose to an origin is identical to the data exposed via `<input type=file>` and `<input type=file webkitdirectory>`. The differences are in giving the origins the ability to re-prompt for access to files they previously had access to (if handles were stored in IndexedDB), and in the ability for origins to write back to the files (after explicit permission is granted for that).

### 2.9. Does this specification enable new script execution/loading mechanisms?

No.

### 2.10. Does this specification allow an origin to access other devices?

Not really. The exception would be devices that are exposed as files or directories by the platform. I.e. network shares or cloud storage sync clients could expose data on other devices in a way that looks like regular files or directories. The user agent could let the user pick these files or directories, thereby giving an origin implicit access to this other device. This API doesn't have any functionality to let a website enumerate all network shares on the local network, only explicitly selected files or directories can be accessed by an origin.

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?

The origin can pop up native file or directory pickers, and have some control over what appears inside that native UI (e.g. accepted file types, starting directory and suggested file names), but that control is very limited. The spec does put limitations on what is allowed as an accepted file type and suggested file name to limit the security impact of allowing websites to control the native UI. User agents are expected to employ similar mechanisms to sanitize the sugessted file names as are used to sanitize for example suggested file names in `<a download="foo.ln">` today.

### 2.12. What temporary identifiers might this this specification create or expose to the web?

The `getUniqueId()` method will create a temporary unique identifier for a given handle. This ID will become invalid if the user clears storage for the site.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

It is expected that user agents do not allow third-party contexts to prompt for any kind of access using this API. I.e. third-party contexts can potentially access files or directories that their origin was already granted access to in a first-party context (by sharing handles via IndexedDB or postMessage), but can't trigger any new file/directory pickers or permission requests.

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

The feature will work mostly the same as in regular mode, except no handles or permission grants will be persistent. Web sites can use this API to store data to disk even in private browsing mode, but to later be able to read this data again (either from private browsing or regular mode), the user would have to explicitly re-pick the same file or directory.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes.

### 2.16. Does this specification allow downgrading default security characteristics?

No.

### 2.17. What should this questionnaire have asked?

Perhaps something about how a feature might impact privacy and security in a bigger picture. Particularly this questionnaire focuses on all the ways it might make privacy or security worse. And while that is important, and while adding new capabilities like this looks scary, from a higher level perspective we do believe that this actually makes things better. Adding these capabilities to the web will lead to Web replacements for one-off native apps, resulting in a net benefit for user security & privacy.
