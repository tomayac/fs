## Site Usage
### Opening / reading
Whenever a site is open, it can issue a request to open a file. If no file is specified to open, the browser will display file selection UI. When the user selects a file, the browser returns the file name and a unique key to the site. The site can later reference this key to open the file without showing file select UI.

### Writing
Whenever a site is open, it can write to a file it has the correct key for. If the site would like to change the name of a file, it must do so through the browser “save as…” dialog. If the site wants to create a new file, it must also use the browser “save as…” dialog.

### Writability of opened files
When a file is opened, it can be `readonly` or `readwrite`. There are two proposals for supporting this:

__Using a `readonly` / `readwrite` attribute__
There is a single open method. Returned file objects have a `writable` attribute that the site can inspect. User agents can always set `writable = true` or only allow this on some heuristic / user action.

This option is less API surface area and allows for easier introspection. 

```html
<input type="file" onchange="processFiles(this.files)">
```

```javascript
function processFiles(file_list) {
    var num_files = file_list.length;
    var file_keys = [];
    for( var i = 0; i < num_files; i++ ) {
        file_keys.push( file_list[i].key );
    }

    // Save file_keys somewhere
    // Do other useful things with the files
}

function openFile(key) {
    navigator.openFile(key).then( function (f) {
        // f is a File
        console.log(f.name);

        // Using a read mode attribute
        if( !f.writable ) {
            alert("Can't write to file foo.txt!");
        }

        var data = new FileReader().readAsText(f);
    }).catch( function(e) {
        console.log(e);
    });
}
```

__Using separate method names__
There are two methods - one to open `readonly` and one to open `readwrite`. The user agent can always allow the `readwrite` method, or can throw an error unless some heuristic / user action succeeds.

This approach allows the site to know it won't be able to write to a file before it makes the user go throught the process of selecting one. It also allows the site to decide without having to open the file.

```html
<input type="file" writable onchange="processFiles(this.files)" onDeny="alert('Need write access')">
```

```javascript
function processFiles(file_list) {
    var num_files = file_list.length;
    var file_keys = [];
    for( var i = 0; i < num_files; i++ ) {
        file_keys.push( file_list[i].key );
    }

    // Save file_keys somewhere
    // Do other useful things with the files
}

// Using separate apis
function openWritableFile(key) {
    navigator.openWritableFile(key).then( function(f) {
        console.log(f.name);
        var data = new FileReader().readAsText(f);
    }).catch( function(e) {
        alert("Can't write to file foo.txt!")
    });
}
```

## Access persistence
When a file is opened (`readonly` or `readwrite`), the site should retain access to that file across browser sessions. If the file is moved, access should be lost.

Each file that a site has access to will have a `read-dirty` flag. If the file has changed since the last time the site accessed it, it is only allowed to access it now if `read-dirty == true`. User agents can choose to always mark `read-dirty` as `true`, always `false`, or provide some mix of heuristics / user actions to set the bit.

## Browser UI
No additional browser permission / UI is required beyond the "File select" and "Save as.." UIs. User agents can choose to show additional UI as desired. For example, there could be a non modal warning shown by the browser whenever a site writes a file:

<div align="center">
    <img src="img/write-warning.png" alt="Warning text when writing a file" width="300px"></img>
</div>

There could also be a non modal warning shown by the browser whenever a site reads from a file that has changed since the last time the site read the file:

<div align="center">
    <img src="img/read-warning.png" alt="Warning text when reading a file" width="300px"></img>
</div>

The browser should display all files a domain has access to in the domain's settings. Users will be allowed to revoke access to any of these files.

## Examples
__Using a `readonly` / `readwrite` attribute__
```html
<input type="file" onchange="processFiles(this.files)">
```

```javascript
function processFiles(file_list) {
    var num_files = file_list.length;
    var file_keys = [];
    for( var i = 0; i < num_files; i++ ) {
        file_keys.push( file_list[i].key );
    }

    // Save file_keys somewhere
    // Do other useful things with the files
}

function openFile(key) {
    navigator.openFile(key).then( function (f) {
        // f is a File
        console.log(f.name);

        // Using a read mode attribute
        if( !f.writable ) {
            alert("Can't write to file foo.txt!");
        }

        var data = new FileReader().readAsText(f);
    }).catch( function(e) {
        console.log(e);
    });
}
```

__Using separate API surfaces__
```html
<input type="file" writable onchange="processFiles(this.files)" onDeny="alert('Need write access')">
```

```javascript
function processFiles(file_list) {
    var num_files = file_list.length;
    var file_keys = [];
    for( var i = 0; i < num_files; i++ ) {
        file_keys.push( file_list[i].key );
    }

    // Save file_keys somewhere
    // Do other useful things with the files
}

// Using separate apis
function openWritableFile(key) {
    navigator.openWritableFile(key).then( function(f) {
        console.log(f.name);
        var data = new FileReader().readAsText(f);
    }).catch( function(e) {
        alert("Can't write to file foo.txt!")
    });
}
```

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