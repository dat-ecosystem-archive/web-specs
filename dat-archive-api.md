# The `DatArchive` API

The `DatArchive` API is an interface for reading, writing, and watching Dat archives from Web applications.

## Behaviors

### Permissions

By default, any `dat://` app can read other Dat sites via HTML embeds, Ajax, or the `DatArchive` read commands. By default, a Dat app is given permission to write to sites that it created. The user will be prompted to give permission to:

 1. Create a new Dat site
 2. Modify a Dat site created by another site

The user must be the owner of a Dat site to modify it.

### Disk usage and quotas

Dat sites are either permanent or temporary. Sites that are created by the user, or which are explicitly saved, are permanent. All other sites (downloaded by browsing, HTML embeds, or Ajax) are temporary and may be automatically deleted.

By default, sites are limited to 100MB of storage. In the future, this will be expandable by user action. When the limit is reached, all writes will fail.

### Special files

The `dat.json` file is a special manifest file that includes metadata and configuration. It cannot be written directly by the application.

## Toplevel methods

### new DatArchive(url)

Create an archive instance from an existing archive.

- `url` String. The URL of the archive to instantiate.

```js
var archive = new DatArchive(datUrl)
```

### DatArchive.create(opts?)

Create a new Dat archive, and return the archive instance.
The archive is sandboxed to only access to files within it.

The user will be prompted to confirm or deny the archive creation.

- `opts.title` String. The title of the new archive.
- `opts.description` String. The description of the new archive.
- returns `Promise<DatArchive>`

```js
var archive = await DatArchive.create({
  title: 'Hello, world!',
  description: 'My new site'
})
```

### DatArchive.fork(url, opts?)

Fork an existing archive to create a new Dat archive, and return the archive instance.
The archive is sandboxed to only access to files within it.

The fork will contain only the files in the target archive that have been completely downloaded.
Use [`download()`](#download) to ensure the intended files are ready.

The user will be prompted to confirm or deny the archive creation.

- `url` String. The URL of the archive to fork.
- `opts.title` String. The title of the new archive.
- `opts.description` String. The description of the new archive.
- returns `Promise<DatArchive>`

```js
var archive = await DatArchive.fork(otherURL, {
  title: 'Hello, world 2!',
  description: 'My new fork'
})
```

### DatArchive.selectArchive(opts?)

This method creates a modal interface for the user to select a Dat archive from the library, or create a new Dat, and return the archive instance.

- `opts.title` String. The title of the select modal.
- `opts.buttonLabel` String. The label on the primary action button.
- `opts.filters.isOwner` Boolean. If true, only show archives that the user owns and can modify.
- returns `Promise<DatArchive>`

```js
var archive = await DatArchive.selectArchive({
  title: 'Hello, world!',
  buttonLabel: 'My new site'
})
```

### DatArchive.resolveName(url)

Resolves a Dat shortname to its "raw" public-key URL using
[DNS-over-TLS](https://github.com/beakerbrowser/beaker/wiki/Authenticated-Dat-URLs-and-HTTPS-to-Dat-Discovery).

- `url` String. The URL of the archive to resolve.
- returns `Promise<string>`

```js
var rawDatUrl = await DatArchive.resolveName(datUrl)
```

## DatArchive instance attributes and methods

### url

The URL of the archive.

```js
var archive = new DatArchive(datUrl)
archive.url
```

### getInfo(opts?)

Fetches information about the archive.

- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- returns `Promise<Object>`

Return object:

```plain
{
  url: string (the archive URL)
  version: number (the archive's current revision number)
  peers: number (the number of active connections)
  isOwner: boolean (is the local user the owner of this archive?)
  title: string (the archive title)
  description: string (the archive description)
  mtime: number (the walltime of the last received update)
}
```

Example:

```js
var archive = new DatArchive(datUrl)
await archive.getInfo()
```

### stat(path, opts?)

Fetches information about the file or directory at `path`.
The promise will reject if no file or directory is found.

- `path` The path of the file/directory to stat.
- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- returns `Promise<Object>`

Return object:

```plain
Stat {
  size: number (bytes)
  blocks: number (number of data blocks in the metadata)
  downloaded: number (number of blocks downloaded, if a remote archive)
  mtime: Date (last modified time; not reliable)
  ctime: Date (creation time; not reliable)
  isDirectory(): boolean
  isFile(): boolean
}
```

Example:

```js
var archive = new DatArchive(datUrl)
await archive.stat('/hello.txt')
```

signature: readFile(path, opts?)

Reads the file’s contents.

- `path` The path of the file to read.
- `opts.encoding`
  - `'utf8'` / `'utf-8'` (default) Return value will be a string.
  - `'base64'` Return value will be a string.
  - `'hex'` Return value will be a string.
  - `'binary'` Return value will be an ArrayBuffer.
  - If `opts` is a string, it is specifying the encoding.
- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- returns `Promise<string|ArrayBuffer>`

```js
var archive = new DatArchive(datUrl)

var buf = await archive.readFile('/picture.png', 'binary')
var blob = new Blob([buf], {type: 'image/png'})
document.querySelector('img').src = URL.createObjectURL(blob)

var str = await archive.readFile('/picture.png', 'base64')
document.querySelector('img').src = 'data:image/png;base64,'+str
```

### readdir(path, opts?)

Reads the contents of a directory.
Returns an array listing the files and folders in the directory, excluding `'.'` and `'..'`.

- `path` The path of the directory to read.
- `opts.recursive` Boolean. Recurse the listing into the subdirectories?
- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- `opts.stat` Boolean. Run stat() on every entry and return with `{name:, stat:}`
- returns `Promise<Array<String>>`

```js
var archive = new DatArchive(datUrl)
var topFiles = await archive.readdir('/')
var allFiles = await archive.readdir('/', {recursive: true})
var stats = await archive.readdir('/', {stat: true})
```

### writeFile(path, data, opts?)

Replaces the file’  s contents with `data` in the staging area.
The promise will reject if there is already a directory at `path`, or if the containing directory-tree does not yet exist.

The archive must be committed to make the file update permanent.

- `path` The path of the file to write.
- `data` The data to be written.
- `opts.encoding`
  - `'utf8'` / `'utf-8'` Data must be a string. (This is the default value if data is a string.)
  - `'base64'` Data must be a string.
  - `'hex'` Data must be a string.
  - `'binary'` Data must be an ArrayBuffer. (This is the default value if data is an ArrayBuffer.)
  - If `opts` is a string, it is specifying the encoding.
- returns `Promise<void>`

```js
var archive = new DatArchive(datUrl)
await archive.writeFile('/hello.txt', 'world')
```

### mkdir(path)

Creates a new directory at `path` in the staging area.
The promise will reject if there is already a file or directory at `url`, or if the containing directory-tree does not yet exist.

The archive must be committed to make the new directory permanent.

- `path` The path of the directory to create.
- returns `Promise<void>`

```js
var archive = new DatArchive(datUrl)
await archive.mkdir('/subdir')
```

### unlink(path)

Deletes the file at `path`.
The promise will reject if no file is found, or if the target path is a directory.

The archive must be committed to make the deletion permanent.

- `path` The path of the file to delete.
- returns `Promise<void>`

```js
var archive = new DatArchive(datUrl)
await archive.unlink('/hello.txt')
```

### rmdir(path, opts?)

Deletes the directory at `path`.
The promise will reject if there is not a directory at `path`, or if the directory is not empty.

The archive must be committed to make the deletion permanent.

- `path` The path of the directory to delete.
- `opts.recursive` Boolean. If not empty, delete all files and folders inside the folder?
- returns `Promise<void>`

```js
var archive = new DatArchive(datUrl)
await archive.rmdir('/subdir')
await archive.rmdir('/', {recursive: true})
```

### diff(opts?)

Compare the current staging area to the published archive, and provide a list of the differences.

- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- `opts.shallow` Boolean. If true, diff() will not recurse into folders that need to be added or removed.
- returns `Promise<Array<Object>>`

Returns an array of objects:

```plain
[
  {
    change: string (the operation, 'add' or 'mod' or 'del')
    type: string (the typeof of entry, 'dir' or 'file')
    path: string (path of the file)
  },
  ...
]
```

Example:

```js
var archive = new DatArchive(datUrl)
await archive.diff()
```

### commit()

Publish all changes in the staging area to the archive, and provide a list of the applied changes.

- returns `Promise<Array<Object>>`

Returns an array of objects:

```plain
[
  {
    change: string (the operation, 'add' or 'mod' or 'del')
    type: string (the typeof of entry, 'dir' or 'file')
    path: string (path of the file)
  },
  ...
]
```

Example:

```js
var archive = new DatArchive(datUrl)
await archive.commit()
```

### revert()

Undo all changes in the staging area, and provide a list of the reversions.

- returns `Promise<Array<Object>>`

Returns an array of objects:

```plain
[
  {
    change: string (the operation, 'add' or 'mod' or 'del')
    type: string (the typeof of entry, 'dir' or 'file')
    path: string (path of the file)
  },
  ...
]
```

Example:

```js
var archive = new DatArchive(datUrl)
await archive.revert()
```

### history(opts?)

Fetch the history of changes to this archive.

- `opts.start` Number. Where in the history to start reading.
- `opts.end` Number. Where in the history to stop reading.
- `opts.reverse` Boolean. Reverse the order given?
- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- returns `Promise<Array<Object>>`

Returns an array of objects:

```plain
[
  {
    path: string (the path of the entry, eg '/index.html')
    version: number (the revision number that represents the change)
    type: string (the operation, 'put' or 'del')
  },
  ...
]
```

Example:

```js
var archive = new DatArchive(datUrl)
await archive.history({start: 0, end: 100, reverse: true})
```

### download(path, opts?)

Download the given file or folder, and return when finished.

- `opts.timeout` Number. How long, in ms, to wait for a response? Default 5000.
- returns `Promise<void>`

```js
var archive = new DatArchive(datUrl)
await archive.download('/foo.txt')
await archive.download('/') // download everything
```

### createFileActivityStream(pattern?)

Watch the given path or path-pattern for file events.

- `pattern`
  - If a string, the path to watch.
  - If an array of strings, an [anymatch](https://npm.im/anymatch) pattern of paths to watch.

Provides an `EventTarget` with a `.close()` method and the following events:

- `'changed' ({path: string})` The contents of the file has changed, either by a local write or a remote write. The new content will be ready when this event is emitted.
  - `path` is the path-string of the file.
- `'invalidated' ({path: string})` The contents of the file has changed remotely, but it hasn't been downloaded yet.
  - `path` is the path-string of the file.

```js
var archive = new DatArchive(datUrl)

var evts = archive.createFileActivityStream() // or...
var evts = archive.createFileActivityStream('foo.txt') // or...
var evts = archive.createFileActivityStream(['**/*.txt', '**/*.md'])

evts.addEventListener('invalidated', ({path}) => {
  console.log(path, 'has been invalidated, downloading the update')
  archive.download(path)
})
evts.addEventListener('changed', ({path}) => {
  console.log(path, 'has been updated!')
})

// later:
evts.close()
```

### createNetworkActivityStream()

Watch for network events. Provides an `EventTarget` with a `.close()` method and the following events:

- `'network-changed' ({connections})` The number of connections has changed.
  - `connections` is a number.
- `'download' ({feed,block,bytes})` A block has been downloaded.
  - `feed` will either be `"metadata"` or `"content"`.
  - `block` is the index of data downloaded.
  - `bytes` is the number of bytes in the block.
- `'upload' ({feed,block,bytes})` A block has been uploaded.
  - `feed` will either be `"metadata"` or `"content"`.
  - `block` is the index of data downloaded.
  - `bytes` is the number of bytes in the block.
- `'sync' ({feed})` All known blocks have been downloaded.
  - `feed` will either be `"metadata"` or `"content"`.

```js
var archive = new DatArchive(datUrl)
var evts = archive.createNetworkActivityStream()

evts.addEventListener('network-changed', ({connections}) => {
  console.log(connections, 'current peers')
})

evts.addEventListener('download', ({feed, block, bytes}) => {
  console.log('Downloaded a block in the', feed, {block, bytes})
})

evts.addEventListener('upload', ({feed, block, bytes}) => {
  console.log('Uploaded a block in the', feed, {block, bytes})
})

evts.addEventListener('sync', ({feed}) => {
  console.log('Downloaded everything currently published in the', feed)
})

// later:
evts.close()
```