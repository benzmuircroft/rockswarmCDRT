# Hypermerge ("hyperswarm": "^2.3.1")

## TODO:
- use rollup to convert ts to js in a single file
- test with old / latest Hyperswarm over 2 devices
- create two versions (old automerge / new automerge using node-fs instead of Hypercore)
## OR:
- use autobase
- create a Hyperswarm messaging network for the updates to propigate
- save with fs or levelDB

# Goal: Only use Hyperswarm to share and fs to store the autobase!

[](https://www.percona.com/blog/wp-content/uploads/2015/10/12.jpg)
## RocksDB
```js
const Automerge = require('automerge');
const Hyperswarm = require('hyperswarm');
const RocksDB = require('rocksdb');
const crypto = require('crypto');
const fs = require('fs');

// ==== CONFIGURATION ====
// Use RocksDB for persistence
const db = new RocksDB('./automerge-db'); // Open a RocksDB instance

// Hard-coded topic hash for peer discovery
const topic = Buffer.from('aabbccddeeff00112233445566778899aabbccddeeff00112233445566778899', 'hex');

// Initialize Hyperswarm for peer-to-peer communication
const swarm = new Hyperswarm();
swarm.join(topic, { lookup: true, announce: true });

let doc = Automerge.init(); // Initialize an empty Automerge document

// ==== LOAD OR CREATE DOCUMENT ====
// Try to load existing state from RocksDB, otherwise start fresh
async function loadDocument() {
    try {
        db.get('automerge-doc', (err, savedState) => {
            if (err) {
                console.log('No saved state found, starting fresh.');
            } else {
                doc = Automerge.load(savedState.toString());
                console.log('Loaded document:', doc);
            }
        });
    } catch (err) {
        console.error('Error loading document:', err);
    }
}

// ==== SAVE DOCUMENT TO ROCKSDB ====
// Periodically save the document state to RocksDB
async function saveDocument() {
    try {
        const docState = Automerge.save(doc);
        db.put('automerge-doc', docState, (err) => {
            if (err) console.error('Error saving document:', err);
        });
    } catch (err) {
        console.error('Error saving document:', err);
    }
}

// ==== HANDLE INCOMING PEER CONNECTIONS ====
// When peers connect, synchronize changes
swarm.on('connection', (socket) => {
    console.log('Connected to a peer');

    socket.on('data', (data) => {
        try {
            const changes = JSON.parse(data.toString());
            const newDoc = Automerge.applyChanges(doc, changes);
            
            // Check if changes need to be saved
            const docChanges = Automerge.getChanges(doc, newDoc);
            if (docChanges.length > 0) {
                doc = newDoc;
                saveDocument();
                console.log('Updated document:', doc);
            }
        } catch (err) {
            console.error('Error applying changes:', err);
        }
    });
});

// ==== APPEND NEW DATA ====
// Modify the document and broadcast changes
async function updateDocument(update) {
    let newDoc = Automerge.change(doc, (d) => {
        d.entries = d.entries || [];
        d.entries.push(update);
    });

    // Check for changes before saving
    const changes = Automerge.getChanges(doc, newDoc);
    if (changes.length > 0) {
        doc = newDoc;
        await saveDocument();

        // Broadcast changes to all connected peers
        const msg = JSON.stringify(changes);
        swarm.connections.forEach(conn => {
            conn.write(msg);  // Broadcast to all peers
        });

        console.log('Appended update:', update);
    }
}

// ==== TESTING ====
// Load document, then append a test entry
(async () => {
    await loadDocument();
    setTimeout(() => updateDocument({ user: 'Ben', action: 'Testing', timestamp: Date.now() }), 2000);
})();

```

## LevelDB
```js
const Automerge = require('automerge');
const Hyperswarm = require('hyperswarm');
const Level = require('level');
const crypto = require('crypto');
const fs = require('fs');

// ==== CONFIGURATION ====
// Use LevelDB for persistence
const db = new Level('./automerge-db', { valueEncoding: 'json' });

// Hard-coded topic hash for peer discovery
// const topic = crypto.createHash('sha256').update('automerge-shared-doc').digest(); // soft-coded
const topic = Buffer.from('aabbccddeeff00112233445566778899aabbccddeeff00112233445566778899', 'hex');

// Initialize Hyperswarm for peer-to-peer communication
const swarm = new Hyperswarm();
swarm.join(topic, { lookup: true, announce: true });

let doc = Automerge.init(); // Initialize an empty Automerge document

// ==== LOAD OR CREATE DOCUMENT ====
// Try to load existing state from LevelDB, otherwise start fresh
async function loadDocument() {
    try {
        const savedState = await db.get('automerge-doc'); // Load the saved state from LevelDB
        doc = Automerge.load(savedState);
        console.log('Loaded document:', doc);
    } catch (err) {
        console.log('No saved state found, starting fresh.');
    }
}

// ==== SAVE DOCUMENT TO LEVELDB ====
// Periodically save the document state to LevelDB
async function saveDocument() {
    await db.put('automerge-doc', Automerge.save(doc));
}

// ==== HANDLE INCOMING PEER CONNECTIONS ====
// When peers connect, synchronize changes
swarm.on('connection', (socket) => {
    console.log('Connected to a peer');

    socket.on('data', (data) => {
        try {
            const changes = JSON.parse(data.toString());
            const newDoc = Automerge.applyChanges(doc, changes);
            
            // Check if changes need to be saved
            const docChanges = Automerge.getChanges(doc, newDoc);
            if (docChanges.length > 0) {
                doc = newDoc;
                saveDocument();
                console.log('Updated document:', doc);
            }
        } catch (err) {
            console.error('Error applying changes:', err);
        }
    });
});

// ==== APPEND NEW DATA ====
// Modify the document and broadcast changes
async function updateDocument(update) {
    let newDoc = Automerge.change(doc, (d) => {
        d.entries = d.entries || [];
        d.entries.push(update);
    });

    // Check for changes before saving
    const changes = Automerge.getChanges(doc, newDoc);
    if (changes.length > 0) {
        doc = newDoc;
        await saveDocument();

        // Broadcast changes to all connected peers
        const msg = JSON.stringify(changes);
        swarm.connections.forEach(conn => {
            conn.write(msg);  // Broadcast to all peers
        });

        console.log('Appended update:', update);
    }
}

// ==== TESTING ====
// Load document, then append a test entry
(async () => {
    await loadDocument();
    setTimeout(() => updateDocument({ user: 'Ben', action: 'Testing', timestamp: Date.now() }), 2000);
})();


```

> **Warning**
> Hypermerge is deprecated.
> This library is no longer maintained and uses an ancient and slow version of Automerge.
> We strongly recommend you adopt https://github.com/automerge/automerge-repo instead.

Hypermerge is a Node.js library for building p2p collaborative applications
without any server infrastructure. It combines [Automerge](https://github.com/automerge/automerge), 
a CRDT, with [hypercore](https://github.com/mafintosh/hypercore), a distributed append-only log.

This project provides a way to have apps data sets that are
conflict free and offline first (thanks to CRDT's) and serverless (thanks to
hypercore/DAT).

While the DAT community has done a lot of work to secure their tool set, zero
effort has been made with hypermerge itself to deal with security and privacy
concerns. Due to the secure nature of the tools its built upon a properly
audited and secure version of this library would be possible in the future.

## How it works

Hypermerge stores an automerge document as a set of separate Hypercores, one per actor ID. Each
actor ID in the document is the discovery key of a Hypercore, which allows recursive lookup.

## Critique

This strategy works for a small number of documents, but is very slow to synchronize due 
to the vast number of Hypercores required for a large collection. It also doesn't work well with
the new Automerge file format which consolidates sequential changes during storage: each change
has its own Hypercore entry with all the attendant metadata. To put this in context, a single
keystroke in a textfield will result in hundreds of bytes of data if you use Hypermerge.

## Examples

There are several example repos in the `/examples` directory, including a very simple two-repo 
code demo and a simple CLI-based chat application.

The best demonstration of Hypermerge is PushPin, which shows Hypermerge in "full flight", including 
taking advantage of splitting the fast, simple front-end from the more expensive, slower back-end. 

## Concepts

The base object you make with hypermerge is a Repo. A repo is responsible for
managing documents and replicating to peers.

### Basic Setup (Serverless or with a Server)

```ts
import { Repo } from 'hypermerge'
import Hyperswarm from 'hyperswarm'

const path = '.data'

const repo = new Repo({ path })

repo.setSwarm(Hyperswarm())
```

### Create / Edit / View / Delete a document

```ts
const url = repo.create({ hello: 'world' })

repo.doc<any>(url, (doc) => {
  console.log(doc) // { hello: "world" }
})

// this is an automerge change function - see automerge for more info
// basically you get to treat the state as a plain old javacript object
// operations you make will be added to an internal append only log and
// replicated to peers

repo.change(url, (state: any) => {
  state.foo = 'bar'
})

repo.doc<any>(url, (doc) => {
  console.log(doc) // { hello: "world", foo: "bar" }
})

// to watch a document that changes over time ...
const handle = repo.watch(url, (doc: any) => {
  console.log(doc)
  if (doc.foo === 'bar') {
    handle.close()
  }
})
```

_NOTE_: If you're familiar with Automerge: the `change` function in Hypermerge
is asynchronous, while the `Automerge.change` function is synchronous. What this
means is that although `Automerge.change` returns an object representing the new
state of your document, `repo.change` (or `handle.change`) does NOT. So:

```ts
// ok in Automerge!
doc1 = Automerge.change(doc1, 'msg', (doc) => {
  doc.foo = 'bar'
})

// NOT ok in Hypermerge!
doc1 = repo.change(url1, (doc) => {
  doc.foo = 'bar'
})
```

Instead, you should expect to get document state updates via `repo.watch`
(or `handle.subscribe`) as shown in the example above.

### Two repos on different machines

```ts
const docUrl = repoA.create({ numbers: [2, 3, 4] })
// this will block until the state has replicated to machine B

repoA.watch<MyDoc>(docUrl, (state) => {
  console.log('RepoA', state)
  // { numbers: [2,3,4] }
  // { numbers: [2,3,4,5], foo: "bar" }
  // { numbers: [2,3,4,5], foo: "bar" } // (local changes repeat)
  // { numbers: [1,2,3,4,5], foo: "bar", bar: "foo" }
})

repoB.watch<MyDoc>(docUrl, (state) => {
  console.log('RepoB', state)
  // { numbers: [1,2,3,4,5], foo: "bar", bar: "foo" }
})

repoA.change<MyDoc>(docUrl, (state) => {
  state.numbers.push(5)
  state.foo = 'bar'
})

repoB.change<MyDoc>(docUrl, (state) => {
  state.numbers.unshift(1)
  state.bar = 'foo'
})
```

### Accessing Files

Hypermerge supports a special kind of core called a hyperfile. Hyperfiles are
unchanging, written-once hypercores that store binary data.

Here's a simple example of reading and writing files.

```ts
// Write an hyperfile
const fileStream = fs.createReadStream('image.png')
const { url } = await repo.files.write(fileStream, 'image/png')

// Read an hyperfile
const fileStream = fs.createWriteStream('image.png')
const hyperfileStream = await repo.files.read(url)

hyperfileStream.pipe(fileStream)
```

Note that hyperfiles are conveniently well-suited to treatment as a native
protocol for Electron applications. This allows you to refer to them throughout
your application directly as though they were regular files for images and other
assets without any special treatment. Here's how to register that:

```js
protocol.registerStreamProtocol(
  'hyperfile',
  (request, callback) => {
    try {
      const stream = await repo.files.read(request.url)
      callback(stream)
    } catch (e) {
      log(e)
    }
  },
  (error) => {
    if (error) {
      log('Failed to register protocol')
    }
  }
)
```

### Splitting the Front-end and Back-end

Both Hypermerge and Automerge supports separating the front-end (where materialized documents live and changes are made) from the backend (where CRDT computations are handled as well as networking and compression.) This is useful for maintaining application performance by moving expensive computation operations off of the render thread to another location where they don't block user input.

The communication between front-end and back-end is all done via simple Javascript objects and can be serialized/deserialized through JSON if required.

```js
  // establish a back-end
  const back = new RepoBackend({ path: HYPERMERGE_PATH, port: 0 })
  const swarm = Hyperswarm({ /* your config here */ })
  back.setSwarm(swarm)

  // elsewhere, create a front-end (you'll probably want to do this in different threads)
  const front = new RepoFrontend()

  // the `subscribe` method sends a message stream, the `receive` receives it
  // for demonstration here we simply output JSON and parse it back in the same location
  // note that front-end and back-end must each subscribe to each other's streams
  back.subscribe((msg) => front.receive(JSON.parse(JSON.stringify(msg))))
  front.subscribe((msg) => back.receive(JSON.parse(JSON.stringify(msg))))

}
```

_Note: each back-end only supports a single front-end today._

### Related libraries

[automerge]: https://github.com/automerge/automerge
[hypercore]: https://github.com/mafintosh/hypercore
