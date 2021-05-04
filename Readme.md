# asar-flex

This library aims to make reading & writing asar archives as easy and flexible as possible, and concerns itself only with the specifics of the asar format, rather than delving into fs crawling, etc. This means that you can source files from anywhere, and stream them to anywhere, be it the local filesystem or a remote server.

Practical applications include pulling individual files out of asar archives stored on an HTTP server, streaming files straight from your build system into the archive, eliminating the need for tmpdirs, etc.

# Readers

## Constructors

### `AsarStreams`

`AsarStreams` is the recommended parser, because it not only supports buffer-based implementations, but will also attempt to stream data from them if requested by constructing many consecutive requests. This provides a greater degree of flexibility in the sourcing of data without changing the API.

```typescript
const ar = new AsarStreams((offset, length) => {
	return fs.createReadStream("/path/to/file.asar", {
		start: offset,
		end: offset + length - 1,
	});
});
```
The getter function can also return a promise resolving to the stream.
This constructor will also accept `AsarArchive`'s buffer-based getter functions, and request individual chunks to emulate a readable stream when using API calls that return a stream.

If you experience performance issues when using buffer-based getter functions, either adapt your function to return a stream instead, or avoid using streaming requests. This can be enforced by using the buffer-based `AsarArchive` implementation, on which this reader was built.

### `AsarArchive`

This is the buffer-based asar reader implementation. The constructor takes a getter function that can be used to fetch a single chunk of data from the source, given a byte offset and length.

```typescript
fs.open("/path/to/file.asar", (fd) => {
	const ar = AsarArchive((offset, length) => new Promise((res, rej) => {
		fs.read(fd, {
			buffer: Buffer.allocUnsafe(length),
			position: offset,
			length,
		}, (err, bytesRead, buf) => {
			if(err) rej(err);
			if(bytesRead !== length) rej(new Error("Incomplete data"));
			res(buf);
		});
	}));
});
```
The getter can also return the Buffer directly instead of a promise.


## Methods

### `AsarArchive`

#### `fetchIndex`
```typescript
await ar.fetchIndex();
```
Fetches the asar archive's file index, caching it for later use. This function must be run before anything else can be done.

#### `getFileIndex`
```typescript
const index: AsarIndex | AsarFile = ar.getFileIndex("/path/to/file");
```
Gets the metadata for a file/folder

#### `isFolder`
```typescript
const isFolder: boolean = ar.isFolder("/path/to/file/or/folder");
```
Determines whether a path points to a file or folder

#### `readdir`
```typescript
const children: string[] = ar.readdir("/path/to/folder");
```
Lists the names of file/folders within the requested folder

#### `readFile`
```typescript
const contents: Buffer = await ar.readFile("/path/to/file");
```
Reads the contents of a file asynchronously

### `AsarStreams extends AsarArchive`

#### `createReadStream`
```typescript
const fileStream: Readable = await ar.createReadStream("/path/to/file");
fileStream.pipe(process.stdout);
// or
fileStream.on("data", (chunk) => process.stdout.write(chunk));
fileStream.on("end", () => console.log("Data finished"));
```
Creates a readable stream through which you can retrieve the file's contents.

# Writer

## Constructor
```typescript
const asarBuilder = new AsarWriter();
```

## Methods

### `addFile`

Writes a file into the archive. Creates parent directories automatically

Files can only be written once. Subsequent writes to the same file will throw an error. It is recommended to buffer inserts and dedupe prior to using this interface.

```typescript
asarBuilder.addFile({
	// Path to the file within the asar archive. Uses unix-style "/".
	path: "path/to/new/file.txt",
	// Readable stream of the file's contents. This can be sourced from anywhere.
	// It can also be a buffer, but this is not recommended, and is only used internally for the asar header.
	stream: fs.createReadStream("file.txt"),
	// Size of the file. Streams will be trimmed to this length, and a value that is too large will yield undefined behavior
	size: 20,
	// Optional attributes for the file, to be injected into the asar index
	attributes: {
		// Whether or not to mark the file as executable
		executable: false,
	},
});
```

### `mkdir`

Creates a new directory within the archive. Also creates parent directories.

```typescript
asarBuilder.mkdir("path/to/new/directory");
```

### `createAsarStream`

Returns a stream for the asar file's data. All streams from `addFile` calls will be consumed at this point, and cannot be used again. Attempting to call any other methods on the writer will throw an error, since it may no longer have access to the resources needed to generate another file.
