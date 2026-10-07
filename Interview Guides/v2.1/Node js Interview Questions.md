# Node.js Interview Questions - Prioritized by Importance

---

## 🏆 TOP 10 MOST IMPORTANT QUESTIONS

---

### Q1: How does Node.js work? Explain its architecture.

**Answer (from given info):**
Node.js works on a single-threaded, event-driven architecture using the V8 JavaScript engine. V8 compiles JavaScript into fast machine code. The Event Loop handles asynchronous tasks (I/O, timers, requests) without blocking the main thread. Libuv library provides a thread pool and handles background tasks like file system operations and networking. Non-blocking I/O lets Node.js process thousands of concurrent requests efficiently without creating multiple threads.

**Polished Answer:**
Node.js operates on a single-threaded, event-driven, non-blocking I/O architecture built on top of Chrome's V8 JavaScript engine. Here's the breakdown:

1. **V8 Engine**: Compiles JavaScript directly into machine code for fast execution
2. **Event Loop**: The heart of Node.js that continuously monitors the call stack and callback queue, executing callbacks when the stack is empty
3. **Libuv**: A C library that provides the thread pool (default 4 threads) for handling expensive operations like file I/O, DNS lookups, and crypto operations
4. **Non-blocking I/O**: Instead of waiting for operations to complete, Node.js registers callbacks and moves on to handle other requests

This architecture allows Node.js to handle thousands of concurrent connections efficiently without the overhead of thread-per-connection models used in traditional backend frameworks.

**TL;DR:** Single-threaded event loop + V8 engine + libuv thread pool = non-blocking I/O for high concurrency.

**Keyword/Key mappings:** Event Loop, V8 Engine, Libuv, Single-threaded, Non-blocking I/O, Asynchronous, Callback Queue

---

### Q2: If Node.js is single-threaded, how does it handle concurrency?

**Answer (from given info):**
Node.js is single-threaded, but it can handle concurrency efficiently through its event-driven, non-blocking I/O model. Node.js uses a single-threaded event loop to handle requests. I/O operations run asynchronously, so other tasks continue without blocking. Completed tasks add callbacks to a queue, which the event loop executes.

**Polished Answer:**
Node.js achieves concurrency despite being single-threaded through its **event-driven, non-blocking I/O model**:

1. **Main Thread (Event Loop)**: Executes JavaScript code synchronously
2. **Thread Pool (libuv)**: Handles expensive operations (file I/O, DNS, crypto) using background threads
3. **Callback Queue**: Completed operations push their callbacks to the queue
4. **Event Loop Phases**: The loop iterates through phases (timers → pending callbacks → poll → check → close callbacks) executing queued callbacks

The key insight: While JavaScript execution is single-threaded, the actual I/O work is delegated to the operating system or thread pool. The event loop continuously checks for completed operations and executes their callbacks without ever blocking.

**Practical example**: When handling 1000 concurrent HTTP requests that query a database, Node.js fires off all 1000 database queries asynchronously. As each query completes, its response callback is queued and executed by the event loop.

**TL;DR:** Single thread for JS execution + async I/O delegation to OS/libuv + event loop callback processing = concurrency without multiple threads.

**Keyword/Key mappings:** Concurrency, Event Loop, Thread Pool, Callback Queue, Non-blocking, Libuv, Asynchronous I/O

---

### Q3: What is the Event Loop in Node.js? Explain with an example.

**Answer (from given info):**
The event loop in Node.js is a mechanism that allows it to handle multiple asynchronous tasks concurrently within a single thread. It continuously listens for events and executes associated callback functions.

Example:
```javascript
console.log("Start");

setTimeout(() => {
  console.log("Timeout callback");
}, 0);

console.log("End");
```
Output: "Start", "End", "Timeout callback"

**Polished Answer:**
The **Event Loop** is the central mechanism that enables Node.js's non-blocking I/O model. It's an infinite loop that:

1. **Checks the call stack** - If empty, it pulls from the callback queue
2. **Processes phases in order**: Timers → Pending callbacks → Idle/Prepare → Poll (I/O) → Check (setImmediate) → Close callbacks
3. **Executes callbacks** associated with completed async operations
4. **Handles nextTick queue** - process.nextTick callbacks run before moving to the next phase

**Execution order example:**
```javascript
console.log("Start");        // 1st: Synchronous, executes immediately

setTimeout(() => {
  console.log("Timeout");    // 4th: Timer phase callback
}, 0);

Promise.resolve().then(() => {
  console.log("Promise");    // 3rd: Microtask, runs after current operation
});

console.log("End");          // 2nd: Synchronous

// Output: Start → End → Promise → Timeout
```

The event loop prioritizes microtasks (Promises, nextTick) over macrotasks (setTimeout, setImmediate).

**TL;DR:** Event Loop = infinite loop checking call stack and callback queue, enabling async execution on a single thread.

**Keyword/Key mappings:** Event Loop, Call Stack, Callback Queue, Microtasks, Macrotasks, Phases, nextTick

---

### Q4: What is the difference between Synchronous and Asynchronous functions?

**Answer (from given info):**
Synchronous functions block execution until the task completes. Asynchronous functions do not block execution and allow other tasks to proceed concurrently. Synchronous executes tasks sequentially; asynchronous initiates tasks and proceeds with other operations while waiting for completion. Synchronous returns result immediately; asynchronous typically returns a promise, callback, or uses event handling.

**Polished Answer:**
| Aspect | Synchronous | Asynchronous |
|--------|------------|--------------|
| **Blocking** | Blocks execution until complete | Non-blocking, continues execution |
| **Execution Order** | Sequential (one after another) | Concurrent (tasks overlap) |
| **Return Type** | Returns value directly | Returns Promise, uses callback, or event |
| **Error Handling** | try-catch works naturally | Requires .catch(), async/await, or error-first callbacks |
| **Use Case** | Simple, CPU-bound operations | I/O operations, network requests, DB queries |

**Example:**
```javascript
// Synchronous
const data = fs.readFileSync('file.txt'); // Blocks until read completes
console.log(data);

// Asynchronous
fs.readFile('file.txt', (err, data) => {  // Non-blocking
    if (err) throw err;
    console.log(data);
});
console.log("This runs before file read completes!");
```

**TL;DR:** Sync = blocking, sequential, immediate result. Async = non-blocking, concurrent, callback/promise-based.

**Keyword/Key mappings:** Blocking, Non-blocking, Sequential, Concurrent, Callbacks, Promises, I/O Operations

---

### Q5: What are Promises and how do they help avoid Callback Hell?

**Answer (from given info):**
A Promise is an object representing the eventual completion or failure of an asynchronous operation. It helps handle asynchronous operations more efficiently compared to callbacks, avoiding callback hell through better chaining using .then() and .catch() methods. The three methods to avoid callback hell are: using async/await(), using promises, and using generators.

**Polished Answer:**
**Callback Hell** (also called "Pyramid of Doom") occurs when multiple async operations are nested:
```javascript
getUser(id, (user) => {
    getPosts(user.id, (posts) => {
        getComments(posts[0].id, (comments) => {
            // Deeply nested - hard to read and maintain
        });
    });
});
```

**Promises** provide a cleaner solution:
```javascript
getUser(id)
    .then(user => getPosts(user.id))
    .then(posts => getComments(posts[0].id))
    .then(comments => console.log(comments))
    .catch(err => console.error(err));
```

**Three ways to avoid Callback Hell:**
1. **Promises**: Chain operations with .then() and .catch()
2. **Async/Await**: Write async code that looks synchronous
3. **Generators**: Pause and resume function execution

**Async/Await example:**
```javascript
async function getUserData(id) {
    try {
        const user = await getUser(id);
        const posts = await getPosts(user.id);
        const comments = await getComments(posts[0].id);
        return comments;
    } catch (err) {
        console.error(err);
    }
}
```

**TL;DR:** Promises = cleaner async handling. Avoid callback hell via Promises, async/await, or generators.

**Keyword/Key mappings:** Callback Hell, Promises, Async/Await, Generators, .then(), .catch(), Pyramid of Doom

---

### Q6: What is Express.js and how does middleware work?

**Answer (from given info):**
Express.js is a minimal and flexible web framework built on top of Node.js. It provides a structured way to handle HTTP requests, routing, and middleware. Middleware in Express is a function that executes between receiving a request and sending a response. The next() function passes control to the next middleware or route handler.

**Polished Answer:**
**Express.js** is the most popular web framework for Node.js, providing:
- **Routing**: Handle HTTP methods (GET, POST, PUT, DELETE) with path matching
- **Middleware**: Functions that process requests before reaching route handlers
- **Template engines**: Support for EJS, Pug, Handlebars
- **Static file serving**: Built-in express.static() middleware

**Middleware functions** have access to:
- `req` (request object)
- `res` (response object)
- `next()` (function to pass control to next middleware)

**Types of Middleware:**
1. **Application-level**: `app.use()` - runs for all requests or specific paths
2. **Router-level**: `router.use()` - scoped to specific routers
3. **Error-handling**: `(err, req, res, next)` - handles errors
4. **Built-in**: `express.json()`, `express.static()`
5. **Third-party**: `cors`, `helmet`, `body-parser`

**Example:**
```javascript
const express = require('express');
const app = express();

// Application-level middleware
app.use((req, res, next) => {
    console.log(`${req.method} ${req.url}`);
    next(); // Must call next() or request hangs!
});

// Route handler
app.get('/users', (req, res) => {
    res.json({ users: [] });
});

app.listen(3000);
```

**TL;DR:** Express = web framework for Node.js. Middleware = functions in request-response cycle, call next() to proceed.

**Keyword/Key mappings:** Express.js, Middleware, Routing, next(), Request-Response Cycle, Web Framework, REST API

---

### Q7: What is NPM and how do you manage packages in a Node.js project?

**Answer (from given info):**
NPM (Node Package Manager) is used to install and manage packages for JavaScript applications. It uses package.json to manage project dependencies and metadata. Package management is handled through npm, allowing installation of third-party packages, creation and publishing of packages.

**Polished Answer:**
**NPM (Node Package Manager)** is the default package manager for Node.js with:
- **Registry**: Largest software registry (over 1 million packages)
- **CLI**: Command-line interface for package management
- **package.json**: Project metadata and dependency manifest

**Essential Commands:**
```bash
# Initialize a project
npm init -y

# Install dependencies
npm install express          # Install and save to dependencies
npm install --save-dev jest  # Install to devDependencies
npm install -g nodemon       # Install globally

# Update dependencies
npm update express
npm update                   # Update all

# Remove dependencies
npm uninstall express

# Install specific version
npm install express@4.18.2

# Install from package.json
npm install                  # Installs all dependencies
npm ci                       # Clean install from package-lock.json
```

**package.json structure:**
```json
{
    "name": "my-app",
    "version": "1.0.0",
    "scripts": {
        "start": "node index.js",
        "dev": "nodemon index.js"
    },
    "dependencies": {
        "express": "^4.18.2"     // Production dependencies
    },
    "devDependencies": {
        "jest": "^29.0.0"        // Development-only
    }
}
```

**Semantic Versioning (SemVer):**
- `^4.18.2`: Compatible with 4.x.x (minor updates allowed)
- `~4.18.2`: Compatible with 4.18.x (patch updates only)
- `4.18.2`: Exact version

**TL;DR:** NPM = package manager. Use npm install/uninstall/update, track in package.json, lock versions in package-lock.json.

**Keyword/Key mappings:** NPM, package.json, Dependencies, devDependencies, Semantic Versioning, npm install, Registry

---

### Q8: How do you handle errors in Express? Explain error-handling middleware.

**Answer (from given info):**
Express handles errors using error-handling middleware with the signature (err, req, res, next). Error-handling middleware must be defined after all routes. Calling next(err) forwards errors to the error handler. Synchronous errors are caught automatically, while asynchronous errors must be passed explicitly using next(err).

**Polished Answer:**
**Error handling in Express** requires understanding the difference between sync and async errors:

**1. Synchronous errors** - caught automatically:
```javascript
app.get('/users', (req, res) => {
    throw new Error('Something went wrong'); // Express catches this
});
```

**2. Asynchronous errors** - must be passed to next():
```javascript
app.get('/users', async (req, res, next) => {
    try {
        const users = await db.getUsers();
        res.json(users);
    } catch (err) {
        next(err); // Must explicitly pass error
    }
});
```

**3. Error-handling middleware** (4 parameters required):
```javascript
// Must be defined AFTER all routes
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(err.status || 500).json({
        message: err.message,
        stack: process.env.NODE_ENV === 'production' ? null : err.stack
    });
});
```

**Best Practices:**
- Always define error middleware last
- Use custom error classes with status codes
- Log errors centrally
- Don't leak stack traces in production
- Use async wrapper for cleaner async error handling:
```javascript
const asyncHandler = (fn) => (req, res, next) =>
    Promise.resolve(fn(req, res, next)).catch(next);
```

**TL;DR:** Sync errors auto-caught; async errors need next(err). Error middleware has 4 params, defined last.

**Keyword/Key mappings:** Error Handling, Error Middleware, next(err), Async Errors, Synchronous Errors, Express.js

---

### Q9: What are Streams in Node.js? Explain the different types.

**Answer (from given info):**
Streams are a powerful way to handle data in chunks rather than loading the entire data into memory. There are four types: Readable Streams (read data, e.g., fs.createReadStream()), Writable Streams (write data, e.g., fs.createWriteStream()), Duplex Streams (both readable and writable, e.g., TCP socket), Transform Streams (duplex streams where data is transformed, e.g., zlib compression).

**Polished Answer:**
**Streams** enable efficient processing of large data by handling it in chunks rather than loading everything into memory.

**Types of Streams:**

| Type | Description | Example |
|------|-------------|---------|
| **Readable** | Data can be read from | `fs.createReadStream()`, HTTP request |
| **Writable** | Data can be written to | `fs.createWriteStream()`, HTTP response |
| **Duplex** | Both readable and writable | `net.Socket`, TCP connection |
| **Transform** | Duplex that modifies data | `zlib.createGzip()`, crypto streams |

**Key Benefits:**
- **Memory efficient**: Process data without loading entirely into memory
- **Time efficient**: Start processing as soon as first chunk arrives
- **Backpressure**: Built-in mechanism to handle fast producer/slow consumer

**Example - Piping:**
```javascript
const fs = require('fs');
const zlib = require('zlib');

// Read file → Compress → Write compressed file
fs.createReadStream('large-file.txt')
    .pipe(zlib.createGzip())
    .pipe(fs.createWriteStream('large-file.txt.gz'));
```

**Backpressure handling:**
```javascript
const readStream = fs.createReadStream('large-file.txt');
const writeStream = fs.createWriteStream('copy.txt');

readStream.on('data', (chunk) => {
    // If writeStream buffer is full, pause reading
    if (!writeStream.write(chunk)) {
        readStream.pause();
    }
});

writeStream.on('drain', () => {
    readStream.resume(); // Resume when buffer drained
});
```

**TL;DR:** Streams = chunked data processing. 4 types: Readable, Writable, Duplex, Transform. Use pipe() for efficient data flow.

**Keyword/Key mappings:** Streams, Readable, Writable, Duplex, Transform, Pipe, Backpressure, Chunked Data

---

### Q10: What is the difference between setImmediate(), setTimeout(), and process.nextTick()?

**Answer (from given info):**
setImmediate() executes callback in the check phase of the event loop, runs after I/O events. process.nextTick() executes callback in the next tick queue, runs before I/O events, has higher priority. setTimeout() executes callback after specified delay, in the timer queue. setImmediate has no timer, designed for immediate execution.

**Polished Answer:**
These three functions control **when** callbacks execute in the event loop, with different priorities:

**Priority Order (highest to lowest):**
1. `process.nextTick()` - Executes immediately after current operation, before event loop continues
2. `setTimeout(cb, 0)` - Executes in timer phase of next iteration
3. `setImmediate()` - Executes in check phase of next iteration

**Detailed Comparison:**
| Function | Phase | Priority | Use Case |
|----------|-------|----------|----------|
| `process.nextTick()` | nextTick queue | Highest | Run before any I/O, catch errors |
| `setTimeout(cb, 0)` | Timer phase | Medium | Delay execution to next loop iteration |
| `setImmediate()` | Check phase | Lower | Run after I/O callbacks in current loop |

**Execution order example:**
```javascript
console.log('Start');

process.nextTick(() => console.log('nextTick'));

setTimeout(() => console.log('setTimeout'), 0);

setImmediate(() => console.log('setImmediate'));

console.log('End');

// Output: Start → End → nextTick → setTimeout → setImmediate
```

**Key insight**: `process.nextTick()` can starve I/O if overused because it always runs before returning to the event loop. Use it sparingly.

**When to use:**
- `process.nextTick()`: Critical cleanup, error propagation before next operation
- `setImmediate()`: Defer execution until after pending I/O
- `setTimeout()`: Actual delays or debouncing

**TL;DR:** Priority: nextTick > setTimeout > setImmediate. nextTick can block I/O if overused.

**Keyword/Key mappings:** process.nextTick, setImmediate, setTimeout, Event Loop Phases, Timer Queue, Check Phase, Priority

---

## 🔥 TOP 25 - IMPORTANT QUESTIONS (11-25)

---

### Q11: What is the purpose of package.json and how do you use environment variables?

**Answer (from given info):**
package.json is a metadata file containing project-specific information such as dependencies, scripts, version, author details. Environment variables are handled using process.env. Install dotenv package to use .env file.

**Polished Answer:**
**package.json** serves as the project manifest:
```json
{
    "name": "my-app",
    "version": "1.0.0",
    "main": "index.js",
    "scripts": {
        "start": "node index.js",
        "test": "jest",
        "dev": "nodemon index.js"
    },
    "dependencies": {
        "express": "^4.18.2"
    },
    "devDependencies": {
        "jest": "^29.0.0"
    }
}
```

**Environment Variables:**
```javascript
// Using dotenv package
require('dotenv').config();

const port = process.env.PORT || 3000;
const dbUrl = process.env.DATABASE_URL;
const nodeEnv = process.env.NODE_ENV;

// .env file
PORT=3000
DATABASE_URL=mongodb://localhost:27017/myapp
NODE_ENV=development
```

**NODE_ENV purpose**: Distinguishes environment (development, testing, production) to customize behavior:
- Enable debugging in development
- Disable stack traces in production
- Use different databases per environment

**TL;DR:** package.json = project manifest. Use process.env + dotenv for configuration. NODE_ENV for environment-specific behavior.

**Keyword/Key mappings:** package.json, process.env, dotenv, NODE_ENV, Environment Variables, Configuration

---

### Q12: Explain the EventEmitter and event-driven programming in Node.js.

**Answer (from given info):**
Event Emitter is a class that allows objects to emit events and register listeners. Part of the events module, used to handle asynchronous events and implement observer pattern. Event-driven programming uses callback functions (event handlers) called when events trigger, and an event loop that listens for triggers.

**Polished Answer:**
**EventEmitter** is the foundation of Node.js's event-driven architecture. It implements the **Observer Pattern** where objects emit events and listeners respond.

**Core methods:**
- `.on(event, listener)`: Register a listener
- `.emit(event, args)`: Trigger an event
- `.once(event, listener)`: Register one-time listener
- `.removeListener(event, listener)`: Remove listener
- `.removeAllListeners()`: Remove all listeners

**Example:**
```javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();

// Register listeners
myEmitter.on('user-login', (user) => {
    console.log(`${user} logged in`);
});

myEmitter.once('server-start', () => {
    console.log('Server started once');
});

// Emit events
myEmitter.emit('user-login', 'Alice');
myEmitter.emit('server-start');
myEmitter.emit('server-start'); // Won't fire (once)

// Built-in events
process.on('exit', (code) => {
    console.log(`Process exiting with code: ${code}`);
});
```

**Common use cases:**
- Handling HTTP requests/responses
- File system operations
- Database connection events
- Custom application events

**TL;DR:** EventEmitter = Observer Pattern. Use .on() to listen, .emit() to trigger. Foundation of Node.js async model.

**Keyword/Key mappings:** EventEmitter, Observer Pattern, Events, Listeners, emit(), on(), Event-driven

---

### Q13: What are Buffers in Node.js and when would you use them?

**Answer (from given info):**
Buffer class is used to perform operations on raw binary data. Buffers refer to particular memory location. Buffers only deal with binary data and cannot be resizable. Each integer in a buffer represents a byte.

**Polished Answer:**
**Buffers** are Node.js's way of handling **raw binary data** directly in memory. They're essential because JavaScript strings are UTF-16 encoded and can't efficiently handle binary data.

**Creating Buffers:**
```javascript
// Create buffer with size
const buf1 = Buffer.alloc(10); // 10 bytes, filled with zeros
const buf2 = Buffer.alloc(10, 'a'); // Filled with 'a'

// Create from string
const buf3 = Buffer.from('Hello World', 'utf8');

// Create from array
const buf4 = Buffer.from([72, 101, 108, 108, 111]); // "Hello"
```

**Common operations:**
```javascript
// Write to buffer
buf1.write('Hello');

// Read from buffer
console.log(buf1.toString()); // "Hello"

// Concatenate buffers
const combined = Buffer.concat([buf3, buf4]);

// Compare buffers
const isEqual = buf3.equals(buf4);

// Slice (returns view, not copy)
const slice = buf3.slice(0, 5);
```

**Use cases:**
- Reading/writing files (fs module)
- Network operations (TCP/UDP)
- Image/video processing
- Binary data manipulation
- Stream processing (chunks are buffers)

**TL;DR:** Buffers = raw binary data handling. Fixed size, non-resizable, used in streams, files, networks.

**Keyword/Key mappings:** Buffer, Binary Data, Memory Allocation, Streams, File I/O, Network Operations

---

### Q14: How do you connect Node.js to MongoDB and handle databases?

**Answer (from given info):**
To connect to MongoDB, install Mongoose package and use mongoose.connect() with connection string and options like useNewUrlParser and useUnifiedTopology. Libraries like Mongoose handle database connection and query execution.

**Polished Answer:**
**Connecting to databases** in Node.js involves using appropriate drivers or ORMs/ODMs:

**MongoDB with Mongoose:**
```javascript
const mongoose = require('mongoose');

mongoose.connect('mongodb://localhost:27017/myapp', {
    useNewUrlParser: true,
    useUnifiedTopology: true
})
.then(() => console.log('Connected to MongoDB'))
.catch(err => console.error('Connection failed:', err));

// Define schema
const userSchema = new mongoose.Schema({
    name: { type: String, required: true },
    email: { type: String, unique: true },
    age: Number
});

// Create model
const User = mongoose.model('User', userSchema);

// CRUD operations
const newUser = new User({ name: 'Alice', email: 'alice@example.com' });
await newUser.save(); // Create

const users = await User.find({ age: { $gt: 18 } }); // Read
await User.updateOne({ _id: id }, { $set: { age: 30 } }); // Update
await User.deleteOne({ _id: id }); // Delete
```

**MySQL with mysql2:**
```javascript
const mysql = require('mysql2');

const connection = mysql.createConnection({
    host: 'localhost',
    user: 'root',
    password: 'password',
    database: 'myapp'
});

connection.query('SELECT * FROM users WHERE age > ?', [18], (err, results) => {
    if (err) throw err;
    console.log(results);
});
```

**Connection pooling (production best practice):**
```javascript
const pool = mysql.createPool({
    connectionLimit: 10,
    host: 'localhost',
    user: 'root',
    database: 'myapp'
});

pool.query('SELECT * FROM users', (err, results) => {
    // Connection automatically released
});
```

**TL;DR:** Use Mongoose for MongoDB, mysql2/pg for SQL. Always use connection pooling in production.

**Keyword/Key mappings:** MongoDB, Mongoose, MySQL, Connection Pooling, CRUD, ODM, Database Connection

---

### Q15: What are Child Processes in Node.js? Explain spawn(), fork(), and exec().

**Answer (from given info):**
Child processes allow running external processes. child_process module spawns child processes that communicate using built-in messaging. Fork creates new instances of the V8 engine. spawn() runs system commands; fork() creates new Node.js processes with IPC channel.

**Polished Answer:**
**Child Processes** allow Node.js to run external commands or other Node.js scripts in separate processes, useful for CPU-intensive tasks.

**Comparison of methods:**

| Method | Description | Communication | Use Case |
|--------|-------------|---------------|----------|
| `spawn()` | Runs any command | Streams (stdout/stderr) | Large data, streaming output |
| `exec()` | Runs command, buffers output | Callback with buffer | Small output, simple commands |
| `fork()` | Creates Node.js child | IPC channel (messaging) | Node-to-Node communication |
| `execFile()` | Runs executable without shell | Callback with buffer | Direct executable execution |

**Examples:**
```javascript
const { spawn, exec, fork } = require('child_process');

// spawn - streaming output
const ls = spawn('ls', ['-la']);
ls.stdout.on('data', (data) => console.log(`stdout: ${data}`));
ls.stderr.on('data', (data) => console.error(`stderr: ${data}`));

// exec - buffered output
exec('ls -la', (err, stdout, stderr) => {
    if (err) throw err;
    console.log(stdout);
});

// fork - Node.js process with IPC
const child = fork('./worker.js');
child.send({ task: 'process-data', data: largeDataset });
child.on('message', (result) => {
    console.log('Result:', result);
});
```

**Worker.js (child process):**
```javascript
process.on('message', (message) => {
    const result = processData(message.data);
    process.send(result);
});
```

**TL;DR:** Child processes = separate processes for heavy tasks. spawn() for streaming, exec() for buffered, fork() for Node.js IPC.

**Keyword/Key mappings:** Child Processes, spawn(), exec(), fork(), IPC, Messaging, Process Management

---

### Q16: What is Cluster in Node.js and how does it improve performance?

**Answer (from given info):**
Cluster modules create child processes that run simultaneously with a single parent process, taking advantage of multi-core systems. Methods include fork(), isWorker, process, send(), kill(). Cluster mode starts multiple Node.js processes with multiple instances of event loop.

**Polished Answer:**
**Cluster module** enables Node.js to take advantage of **multi-core systems** by creating multiple worker processes, each with its own event loop.

**How it works:**
```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
    console.log(`Master ${process.pid} is running`);

    // Fork workers equal to CPU cores
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }

    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        cluster.fork(); // Restart dead workers
    });
} else {
    // Workers share the same TCP connection
    http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Handled by worker ${process.pid}\n`);
    }).listen(8000);

    console.log(`Worker ${process.pid} started`);
}
```

**Key Cluster Methods:**
- `cluster.fork()`: Create new worker
- `cluster.isMaster`: Check if current is master
- `cluster.isWorker`: Check if current is worker
- `worker.send()`: Send message to worker
- `worker.kill()`: Terminate worker

**Cluster vs Worker Threads:**
| Aspect | Cluster | Worker Threads |
|--------|---------|----------------|
| Process/Thread | Multiple processes | Multiple threads in one process |
| Memory | Separate memory space | Shared memory |
| Use Case | Scale HTTP requests | CPU-intensive tasks |
| Communication | IPC | SharedArrayBuffer, postMessage |

**TL;DR:** Cluster = multiple processes for multi-core utilization. Each worker has own event loop. Use for scaling HTTP servers.

**Keyword/Key mappings:** Cluster, Multi-core, Workers, Master Process, Fork, Load Balancing, IPC

---

### Q17: What is the crypto module and how do you use it for encryption?

**Answer (from given info):**
Crypto module is used for encrypting, decrypting, or hashing data. It secures data by adding authentication layer. Converts plain readable text to encrypted format and decrypts when required.

**Polished Answer:**
**Crypto module** provides cryptographic functionality including encryption, hashing, and secure random generation.

**Hashing (one-way):**
```javascript
const crypto = require('crypto');

// SHA-256 hashing
const hash = crypto.createHash('sha256')
    .update('password123')
    .digest('hex');
console.log(hash); // Fixed-length hash

// HMAC (keyed hash)
const hmac = crypto.createHmac('sha256', 'secret-key')
    .update('data-to-sign')
    .digest('hex');
```

**Encryption (two-way):**
```javascript
// Symmetric encryption (AES)
const algorithm = 'aes-256-cbc';
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

function encrypt(text) {
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
}

function decrypt(encrypted) {
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
}

// Password hashing with salt (recommended)
const salt = crypto.randomBytes(16).toString('hex');
crypto.scrypt('password123', salt, 64, (err, derivedKey) => {
    if (err) throw err;
    console.log(derivedKey.toString('hex'));
});
```

**Random generation:**
```javascript
// Secure random bytes
const randomBytes = crypto.randomBytes(32);

// Random UUID
const uuid = crypto.randomUUID();
```

**Use cases:**
- Password hashing (bcrypt, scrypt, PBKDF2)
- Data encryption for sensitive info
- Digital signatures
- Secure token generation

**TL;DR:** Crypto = encryption, hashing, random generation. Use bcrypt/scrypt for passwords, AES for encryption.

**Keyword/Key mappings:** Crypto, Encryption, Decryption, Hashing, SHA-256, AES, HMAC, Salt

---

### Q18: How do you implement authentication and authorization in Node.js?

**Answer (from given info):**
Authentication verifies user identity; authorization determines access rights. Implement using Passport for strategies (OAuth, Google, GitHub) and JWT (jsonwebtoken) for token-based authentication and role-based authorization.

**Polished Answer:**
**Authentication vs Authorization:**
- **Authentication**: Who are you? (login, verify identity)
- **Authorization**: What can you do? (permissions, roles)

**JWT (JSON Web Token) implementation:**
```javascript
const jwt = require('jsonwebtoken');

// 1. User login - create token
app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    
    // Verify credentials against database
    const user = await User.findOne({ username });
    const isValid = await bcrypt.compare(password, user.password);
    
    if (!isValid) return res.status(401).json({ error: 'Invalid credentials' });
    
    // Create JWT token
    const token = jwt.sign(
        { id: user.id, username: user.username, role: user.role },
        process.env.JWT_SECRET,
        { expiresIn: '1h' }
    );
    
    res.json({ token });
});

// 2. Authentication middleware
const authenticate = (req, res, next) => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader) return res.status(401).json({ error: 'No token' });
    
    const token = authHeader.split(' ')[1];
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (err) {
        return res.status(403).json({ error: 'Invalid token' });
    }
};

// 3. Authorization middleware
const authorize = (...roles) => {
    return (req, res, next) => {
        if (!roles.includes(req.user.role)) {
            return res.status(403).json({ error: 'Insufficient permissions' });
        }
        next();
    };
};

// 4. Protected routes
app.get('/admin/dashboard', 
    authenticate, 
    authorize('admin'), 
    (req, res) => {
        res.json({ message: 'Admin access granted' });
    }
);
```

**Passport.js strategies:**
```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
}, (accessToken, refreshToken, profile, done) => {
    // Find or create user
    User.findOrCreate({ googleId: profile.id }, (err, user) => {
        return done(err, user);
    });
}));
```

**Security best practices:**
- Hash passwords (bcrypt, argon2)
- Use HTTPS
- Store tokens securely
- Implement refresh tokens
- Set token expiry

**TL;DR:** Auth = verify identity (JWT, Passport, sessions). Authorization = check permissions (roles, middleware).

**Keyword/Key mappings:** Authentication, Authorization, JWT, Passport, OAuth, Middleware, Token, Sessions

---

### Q19: Explain the concept of REST API and different HTTP methods.

**Answer (from given info):**
REST API stands for REpresentational State Transfer. Allows communication between systems over internet, typically in JSON format. Uses HTTP methods: GET (retrieve), POST (create), PUT (update entire), PATCH (partial update), DELETE (remove). Methods align with CRUD operations.

**Polished Answer:**
**REST API** is an architectural style for building APIs that use HTTP methods to perform CRUD operations on resources.

**HTTP Methods mapped to CRUD:**

| HTTP Method | CRUD Operation | Description | Example |
|-------------|---------------|-------------|---------|
| **GET** | Read | Retrieve resource(s) | `GET /users/123` |
| **POST** | Create | Create new resource | `POST /users` |
| **PUT** | Update (full) | Replace entire resource | `PUT /users/123` |
| **PATCH** | Update (partial) | Update specific fields | `PATCH /users/123` |
| **DELETE** | Delete | Remove resource | `DELETE /users/123` |

**RESTful API Design:**
```javascript
const express = require('express');
const app = express();
app.use(express.json());

// GET - Retrieve all users
app.get('/api/users', (req, res) => {
    res.json(users);
});

// GET - Retrieve specific user
app.get('/api/users/:id', (req, res) => {
    const user = users.find(u => u.id === parseInt(req.params.id));
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
});

// POST - Create user
app.post('/api/users', (req, res) => {
    const newUser = { id: users.length + 1, ...req.body };
    users.push(newUser);
    res.status(201).json(newUser);
});

// PUT - Update entire user
app.put('/api/users/:id', (req, res) => {
    const index = users.findIndex(u => u.id === parseInt(req.params.id));
    users[index] = { id: users[index].id, ...req.body };
    res.json(users[index]);
});

// PATCH - Partial update
app.patch('/api/users/:id', (req, res) => {
    const user = users.find(u => u.id === parseInt(req.params.id));
    Object.assign(user, req.body);
    res.json(user);
});

// DELETE - Remove user
app.delete('/api/users/:id', (req, res) => {
    users = users.filter(u => u.id !== parseInt(req.params.id));
    res.status(204).send();
});
```

**Status codes:**
- `200`: Success
- `201`: Created
- `204`: No content
- `400`: Bad request
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Not found
- `500`: Server error

**TL;DR:** REST API = HTTP methods for CRUD. GET/read, POST/create, PUT/PATCH/update, DELETE/delete.

**Keyword/Key mappings:** REST API, HTTP Methods, CRUD, GET, POST, PUT, PATCH, DELETE, Status Codes

---

### Q20: What is CORS and how do you handle it in Node.js?

**Answer (from given info):**
CORS stands for Cross-Origin Resource Sharing. HTTP-header based mechanism allowing servers to indicate origins permitted to access resources. Use cors package from npm to handle CORS errors.

**Polished Answer:**
**CORS (Cross-Origin Resource Sharing)** is a security mechanism implemented by browsers to prevent unauthorized cross-origin requests.

**Why CORS exists:**
- Same-Origin Policy restricts requests to the same origin
- CORS allows controlled cross-origin access
- Prevents CSRF attacks

**Implementation with cors package:**
```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// Option 1: Allow all origins
app.use(cors());

// Option 2: Allow specific origin
app.use(cors({
    origin: 'https://example.com',
    methods: ['GET', 'POST'],
    allowedHeaders: ['Content-Type', 'Authorization']
}));

// Option 3: Multiple origins
const allowedOrigins = ['https://app1.com', 'https://app2.com'];
app.use(cors({
    origin: (origin, callback) => {
        if (allowedOrigins.indexOf(origin) !== -1 || !origin) {
            callback(null, true);
        } else {
            callback(new Error('Not allowed by CORS'));
        }
    }
}));

// Option 4: Route-specific CORS
app.get('/public-data', cors(), (req, res) => {
    res.json({ data: 'Public' });
});

app.get('/private-data', cors({
    origin: 'https://trusted.com'
}), (req, res) => {
    res.json({ data: 'Private' });
});
```

**Manual CORS headers:**
```javascript
app.use((req, res, next) => {
    res.header('Access-Control-Allow-Origin', 'https://example.com');
    res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.header('Access-Control-Allow-Credentials', 'true');
    
    // Handle preflight
    if (req.method === 'OPTIONS') {
        return res.sendStatus(204);
    }
    next();
});
```

**TL;DR:** CORS = browser security for cross-origin requests. Use cors package, configure origins, methods, headers.

**Keyword/Key mappings:** CORS, Cross-Origin, Same-Origin Policy, Preflight, Access-Control-Headers

---

### Q21: Explain the util module and other core Node.js modules (os, path, dns, net).

**Answer (from given info):**
Util module provides utility functions. OS module provides operating system utilities. Path module transforms and handles file paths. DNS module enables OS name resolution. Net module creates TCP clients and servers.

**Polished Answer:**
**Core Node.js modules** provide essential functionality without installation:

**1. util module:**
```javascript
const util = require('util');

// Promisify callback-based functions
const readFile = util.promisify(fs.readFile);
const data = await readFile('file.txt', 'utf8');

// Format strings
util.format('%s is %d years old', 'Alice', 30);

// Inspect objects
console.log(util.inspect(obj, { depth: 2, colors: true }));

// Type checking
util.types.isDate(new Date()); // true
util.types.isPromise(Promise.resolve()); // true
```

**2. os module:**
```javascript
const os = require('os');

os.cpus();        // CPU info
os.totalmem();    // Total memory
os.freemem();     // Free memory
os.platform();    // 'linux', 'darwin', 'win32'
os.hostname();    // System hostname
os.userInfo();    // Current user info
```

**3. path module:**
```javascript
const path = require('path');

path.join('/users', 'alice', 'file.txt');  // Normalized path
path.resolve('folder', 'file.txt');         // Absolute path
path.dirname('/users/alice/file.txt');      // '/users/alice'
path.basename('/users/alice/file.txt');     // 'file.txt'
path.extname('file.txt');                    // '.txt'
path.parse('/users/alice/file.txt');         // { root, dir, base, ext, name }
```

**4. dns module:**
```javascript
const dns = require('dns');

// Resolve domain to IP
dns.lookup('example.com', (err, address) => {
    console.log('IP:', address);
});

// Reverse lookup
dns.reverse('93.184.216.34', (err, hostnames) => {
    console.log('Hostnames:', hostnames);
});
```

**5. net module (TCP):**
```javascript
const net = require('net');

// Create TCP server
const server = net.createServer((socket) => {
    console.log('Client connected');
    
    socket.on('data', (data) => {
        console.log('Received:', data.toString());
        socket.write('Echo: ' + data);
    });
    
    socket.on('end', () => console.log('Client disconnected'));
});

server.listen(3000, () => console.log('TCP server on port 3000'));
```

**TL;DR:** util = utilities, os = system info, path = file paths, dns = name resolution, net = TCP networking.

**Keyword/Key mappings:** util, os, path, dns, net, Core Modules, promisify, TCP, File Paths

---

### Q22: What are WebSockets and how do they differ from HTTP?

**Answer (from given info):**
Web Socket is a protocol providing full-duplex communication, allowing communication in both directions simultaneously. Continuous connection between client and server. Both can send messages at any point in time.

**Polished Answer:**
**WebSockets** enable **real-time, bidirectional communication** between client and server, unlike HTTP's request-response model.

**HTTP vs WebSocket:**
| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| Communication | Request-response | Full-duplex |
| Connection | New per request | Persistent |
| Server push | Not possible (polling) | Yes, anytime |
| Overhead | High (headers each time) | Low (after handshake) |
| Use case | REST APIs, static content | Chat, gaming, live updates |

**Implementation with ws library:**
```javascript
const WebSocket = require('ws');
const server = new WebSocket.Server({ port: 8080 });

server.on('connection', (socket) => {
    console.log('Client connected');
    
    // Send message to client
    socket.send('Welcome!');
    
    // Receive message from client
    socket.on('message', (message) => {
        console.log('Received:', message.toString());
        
        // Broadcast to all clients
        server.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(message);
            }
        });
    });
    
    socket.on('close', () => console.log('Client disconnected'));
});
```

**Socket.io (more features):**
```javascript
const io = require('socket.io')(server);

io.on('connection', (socket) => {
    console.log('User connected');
    
    socket.on('chat-message', (message) => {
        io.emit('chat-message', message); // Broadcast
    });
    
    socket.on('disconnect', () => {
        console.log('User disconnected');
    });
});
```

**Use cases:**
- Real-time chat applications
- Live sports scores
- Collaborative editing
- Online gaming
- Stock market updates
- IoT device communication

**TL;DR:** WebSocket = persistent, bidirectional communication. Use for real-time apps. socket.io adds features.

**Keyword/Key mappings:** WebSocket, Full-duplex, Real-time, Bidirectional, socket.io, Persistent Connection

---

### Q23: How do you validate data and handle file uploads in Node.js?

**Answer (from given info):**
Validation can be done using express-validator module. File uploading uses Multer middleware for handling multipart/form-data. Other modules include hapi/joi.

**Polished Answer:**
**Data validation** and **file uploads** are critical for security and data integrity.

**1. Data Validation with express-validator:**
```javascript
const { body, validationResult } = require('express-validator');

app.post('/users',
    body('email').isEmail().normalizeEmail(),
    body('password').isLength({ min: 8 }),
    body('age').isInt({ min: 18, max: 100 }),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        
        // Process valid data
        const { email, password, age } = req.body;
        res.json({ message: 'Valid data' });
    }
);
```

**2. Validation with Joi:**
```javascript
const Joi = require('joi');

const schema = Joi.object({
    email: Joi.string().email().required(),
    password: Joi.string().min(8).required(),
    age: Joi.number().integer().min(18).max(100)
});

const { error, value } = schema.validate(req.body);
if (error) {
    return res.status(400).json({ error: error.details[0].message });
}
```

**3. File Uploads with Multer:**
```javascript
const multer = require('multer');
const path = require('path');

// Configure storage
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
        cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
    }
});

// File filter for security
const fileFilter = (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (allowedTypes.includes(file.mimetype)) {
        cb(null, true);
    } else {
        cb(new Error('Invalid file type'), false);
    }
};

// Configure upload
const upload = multer({
    storage: storage,
    fileFilter: fileFilter,
    limits: { fileSize: 5 * 1024 * 1024 } // 5MB limit
});

// Single file upload
app.post('/upload', upload.single('file'), (req, res) => {
    res.json({ 
        message: 'File uploaded', 
        file: req.file 
    });
});

// Multiple files
app.post('/upload-multiple', upload.array('files', 10), (req, res) => {
    res.json({ 
        message: 'Files uploaded', 
        files: req.files 
    });
});
```

**Security best practices:**
- Validate file type (MIME + extension)
- Limit file size
- Generate unique filenames
- Store outside web root
- Scan for malware

**TL;DR:** Use express-validator/Joi for data validation. Use Multer for file uploads. Always validate and limit.

**Keyword/Key mappings:** Validation, express-validator, Joi, Multer, File Upload, Security, multipart/form-data

---

### Q24: What is REPL and what tools do you use for code consistency?

**Answer (from given info):**
REPL stands for Read, Evaluate, Print, Loop. Computer environment for writing and debugging code. ESLint is used for consistent code style, written using NodeJS for fast runtime and easy installation via npm.

**Polished Answer:**
**REPL (Read-Eval-Print-Loop)** is an interactive shell for testing JavaScript code:

**Features of REPL:**
```bash
$ node
> console.log('Hello')  # Read + Evaluate + Print
Hello
undefined
>                       # Loop back for more input

# Multi-line input
> function add(a, b) {
... return a + b;
... }
undefined
> add(5, 3)
8

# Special commands
> .help    # Show all commands
> .clear   # Clear context
> .exit    # Exit REPL
> .save file.js  # Save session
> .load file.js  # Load file
```

**ESLint for code consistency:**
```javascript
// .eslintrc.json
{
    "env": {
        "node": true,
        "es2021": true
    },
    "extends": "eslint:recommended",
    "rules": {
        "semi": ["error", "always"],
        "quotes": ["error", "single"],
        "indent": ["error", 4],
        "no-unused-vars": "warn",
        "no-console": "off"
    }
}

// Package.json scripts
{
    "scripts": {
        "lint": "eslint .",
        "lint:fix": "eslint . --fix"
    }
}
```

**Other code quality tools:**
- **Prettier**: Code formatter
- **Husky**: Git hooks for linting
- **lint-staged**: Run linters on staged files
- **EditorConfig**: Editor-agnostic settings

**TL;DR:** REPL = interactive JS shell. ESLint = code consistency. Add Prettier for formatting.

**Keyword/Key mappings:** REPL, ESLint, Code Consistency, Interactive Shell, Debugging, Prettier

---

### Q25: How do you implement session management and what is the passport module?

**Answer (from given info):**
Session management done using express-session module. Saves data in key-value form. Session data not saved in cookie, just session ID. Passport module implements authentication measures for sign-in operations.

**Polished Answer:**
**Session management** maintains state between HTTP requests. **Passport** provides authentication middleware with multiple strategies.

**1. Express Sessions:**
```javascript
const session = require('express-session');
const MongoStore = require('connect-mongo');

app.use(session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: { 
        secure: process.env.NODE_ENV === 'production',
        maxAge: 24 * 60 * 60 * 1000, // 24 hours
        httpOnly: true 
    },
    store: MongoStore.create({
        mongoUrl: process.env.MONGODB_URI
    })
}));

// Set session
app.post('/login', (req, res) => {
    req.session.user = { id: user.id, username: user.username };
    res.json({ message: 'Logged in' });
});

// Get session
app.get('/profile', (req, res) => {
    if (req.session.user) {
        res.json(req.session.user);
    } else {
        res.status(401).json({ error: 'Not logged in' });
    }
});

// Destroy session
app.post('/logout', (req, res) => {
    req.session.destroy((err) => {
        res.json({ message: 'Logged out' });
    });
});
```

**2. Passport Authentication:**
```javascript
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const bcrypt = require('bcrypt');

// Configure strategy
passport.use(new LocalStrategy(
    { usernameField: 'email' },
    async (email, password, done) => {
        try {
            const user = await User.findOne({ email });
            if (!user) return done(null, false, { message: 'User not found' });
            
            const isValid = await bcrypt.compare(password, user.password);
            if (!isValid) return done(null, false, { message: 'Wrong password' });
            
            return done(null, user);
        } catch (err) {
            return done(err);
        }
    }
));

// Serialize user to session
passport.serializeUser((user, done) => {
    done(null, user.id);
});

// Deserialize user from session
passport.deserializeUser(async (id, done) => {
    try {
        const user = await User.findById(id);
        done(null, user);
    } catch (err) {
        done(err);
    }
});

// Initialize passport
app.use(passport.initialize());
app.use(passport.session());

// Login route
app.post('/login', 
    passport.authenticate('local', { 
        successRedirect: '/dashboard',
        failureRedirect: '/login',
        failureFlash: true 
    })
);

// OAuth strategies
passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
    const user = await User.findOrCreate({ googleId: profile.id });
    return done(null, user);
}));
```

**Session vs JWT:**
| Aspect | Sessions | JWT |
|--------|----------|-----|
| Storage | Server-side | Client-side |
| Scalability | Requires shared store | Stateless |
| Revocation | Easy (destroy session) | Hard (blacklist) |
| Payload size | Small (just ID) | Larger (data in token) |

**TL;DR:** Sessions store state server-side, client gets session ID. Passport provides auth strategies (local, OAuth).

**Keyword/Key mappings:** Sessions, express-session, Passport, Authentication, serializeUser, deserializeUser, OAuth

---

## 📚 TOP 50 - IMPORTANT QUESTIONS (26-50)

---

### Q26: Explain the concept of Control Flow in Node.js.

**Answer (from given info):**
Control flow refers to the order in which asynchronous operations are executed and how their results are handled. Tasks don't always finish in the order they start. The order of statement execution: execution and queue handling, collection of data and storing, handling concurrency, executing next lines of code.

**Polished Answer:**
**Control flow** in Node.js manages the execution order of asynchronous operations. Since async operations complete at different times, control flow ensures proper sequencing.

**Common patterns:**
```javascript
// 1. Sequential execution
async function sequential() {
    const user = await getUser(1);
    const posts = await getPosts(user.id);
    const comments = await getComments(posts[0].id);
    return comments;
}

// 2. Parallel execution
async function parallel() {
    const [user, posts, comments] = await Promise.all([
        getUser(1),
        getPosts(1),
        getComments(1)
    ]);
    return { user, posts, comments };
}

// 3. Conditional flow
async function conditional() {
    const user = await getUser(1);
    if (user.isAdmin) {
        const reports = await getAdminReports();
        return reports;
    }
    return user;
}
```

**Control flow libraries:**
- **async.js**: Provides waterfall, parallel, series
- **Bluebird**: Promise utilities
- **Native Promises**: Promise.all, Promise.race

**TL;DR:** Control flow = managing async execution order. Use Promises, async/await for clean flow.

**Keyword/Key mappings:** Control Flow, Asynchronous, Sequential, Parallel, Async.js, Promise.all

---

### Q27: What are global objects in Node.js?

**Answer (from given info):**
Global objects are available in all modules without importing. Provide built-in functionalities and information about NodeJS environment. Similar to window object in browsers but for server-side.

**Polished Answer:**
**Global objects** are accessible everywhere without requiring imports:

**Key global objects:**
```javascript
// 1. process - runtime information
process.pid          // Process ID
process.env          // Environment variables
process.argv         // Command line arguments
process.exit()       // Exit process
process.memoryUsage() // Memory info

// 2. __dirname - current directory
console.log(__dirname);

// 3. __filename - current file path
console.log(__filename);

// 4. console - logging
console.log('Info');
console.error('Error');
console.warn('Warning');

// 5. setTimeout/setInterval/setImmediate
const timer = setTimeout(() => {}, 1000);
clearTimeout(timer);

// 6. Buffer - binary data
const buf = Buffer.from('Hello');

// 7. global - global namespace
global.myVariable = 'accessible everywhere';

// 8. module/exports/require
module.exports = {};
const fs = require('fs');
```

**Reading command line arguments:**
```javascript
// index.js
const args = process.argv;
console.log(args);
// node index.js hello world
// Output: ['node', 'index.js', 'hello', 'world']

// Useful parsing
const [nodePath, scriptPath, ...rest] = process.argv;
```

**TL;DR:** Global objects = always available: process, console, Buffer, __dirname, timers, global.

**Keyword/Key mappings:** Global Objects, process, console, Buffer, __dirname, setTimeout, require

---

### Q28: What is the Test Pyramid and how do you approach testing in Node.js?

**Answer (from given info):**
Test Pyramid is a strategy for structuring tests. Three levels: Unit Tests (base, test individual components), Integration Tests (middle, test interactions), End-to-End Tests (top, test entire application flow).

**Polished Answer:**
**Test Pyramid** provides a framework for balanced testing strategy:

```
        /\
       /E2E\        Few, slow
      /------\
     /  Int. \      Medium
    /----------\
   /   Unit     \   Many, fast
  /--------------\
```

**Testing levels:**

**1. Unit Tests (Most numerous):**
```javascript
// math.test.js
const { add, multiply } = require('./math');

describe('Math functions', () => {
    test('adds two numbers', () => {
        expect(add(2, 3)).toBe(5);
    });
    
    test('multiplies two numbers', () => {
        expect(multiply(2, 3)).toBe(6);
    });
});
```

**2. Integration Tests:**
```javascript
// api.test.js
const request = require('supertest');
const app = require('./app');
const db = require('./db');

describe('API Integration', () => {
    beforeAll(async () => {
        await db.connect();
    });
    
    test('GET /users returns array', async () => {
        const res = await request(app).get('/users');
        expect(res.status).toBe(200);
        expect(Array.isArray(res.body)).toBe(true);
    });
    
    test('POST /users creates user', async () => {
        const res = await request(app)
            .post('/users')
            .send({ name: 'Alice' });
        expect(res.status).toBe(201);
    });
});
```

**3. End-to-End Tests (Fewest):**
```javascript
// e2e.test.js (using Cypress or Playwright)
describe('User Flow', () => {
    it('completes signup process', () => {
        cy.visit('/signup');
        cy.get('input[name="email"]').type('test@example.com');
        cy.get('input[name="password"]').type('password123');
        cy.get('button[type="submit"]').click();
        cy.url().should('include', '/dashboard');
    });
});
```

**Testing frameworks:**
- **Jest**: Unit + integration testing
- **Mocha + Chai**: Flexible testing
- **Supertest**: HTTP assertions
- **Sinon**: Spies, stubs, mocks
- **Cypress/Playwright**: E2E testing

**Stubbing example:**
```javascript
const sinon = require('sinon');
const request = require('request');
const getPhotos = require('./index');

describe('getPhotos', () => {
    before(() => {
        sinon.stub(request, 'get')
            .yields(null, null, JSON.stringify([{ id: 1, title: 'Photo' }]));
    });
    
    after(() => {
        request.get.restore();
    });
    
    it('returns photos', async () => {
        const photos = await getPhotos(1);
        expect(photos).to.have.length(1);
    });
});
```

**TL;DR:** Test pyramid = many unit tests, some integration, few E2E. Use Jest, Mocha, Supertest, Sinon.

**Keyword/Key mappings:** Testing, Unit Tests, Integration Tests, E2E Tests, Jest, Mocha, Supertest, Sinon, Stubs

---

### Q29: What is the TLS module and how do you secure Node.js applications?

**Answer (from given info):**
TLS module provides implementation of Transport Layer Security and Secure Socket Layer protocols built on OpenSSL. Helps establish secure connections.

**Polished Answer:**
**TLS/SSL** ensures encrypted communication. The `tls` module provides cryptographic protocols for secure networking.

**Creating HTTPS server:**
```javascript
const https = require('https');
const fs = require('fs');

const options = {
    key: fs.readFileSync('private-key.pem'),
    cert: fs.readFileSync('certificate.pem'),
    ca: fs.readFileSync('ca-certificate.pem'), // Optional
    requestCert: true, // Request client cert
    rejectUnauthorized: true // Reject invalid certs
};

https.createServer(options, (req, res) => {
    res.writeHead(200);
    res.end('Secure connection established\n');
}).listen(443);
```

**Using Express with HTTPS:**
```javascript
const express = require('express');
const https = require('https');
const fs = require('fs');

const app = express();
app.get('/', (req, res) => res.send('Secure!'));

https.createServer({
    key: fs.readFileSync('server.key'),
    cert: fs.readFileSync('server.cert')
}, app).listen(443);
```

**Security best practices:**
```javascript
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

// Set security headers
app.use(helmet());

// Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // Limit each IP to 100 requests
});
app.use(limiter);

// Input validation
app.use(express.json({ limit: '10kb' }));

// Prevent parameter pollution
const hpp = require('hpp');
app.use(hpp());
```

**Common security measures:**
- Use HTTPS (TLS/SSL)
- Set security headers (helmet)
- Rate limiting
- Input validation
- SQL injection prevention
- XSS protection
- CSRF tokens
- Secure session management

**TL;DR:** TLS = encrypted communication. Use helmet, rate limiting, validation for security.

**Keyword/Key mappings:** TLS, SSL, HTTPS, Security, Helmet, Rate Limiting, Encryption

---

### Q30: What is piping in Node.js?

**Answer (from given info):**
Piping refers to passing output of one stream directly into another stream. Allows data flow through multiple streams without storing in memory. Common pattern in file handling, HTTP requests, I/O operations.

**Polished Answer:**
**Piping** connects streams together, automatically handling data flow and backpressure:

**Basic piping:**
```javascript
const fs = require('fs');

// Read from file, pipe to write file
const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');

readStream.pipe(writeStream);

// Handle completion
writeStream.on('finish', () => {
    console.log('File copied successfully');
});
```

**Chaining pipes:**
```javascript
const zlib = require('zlib');

// Read → Compress → Write
fs.createReadStream('large-file.txt')
    .pipe(zlib.createGzip())
    .pipe(fs.createWriteStream('large-file.txt.gz'));

// Read → Decompress → Write
fs.createReadStream('large-file.txt.gz')
    .pipe(zlib.createGunzip())
    .pipe(fs.createWriteStream('large-file.txt'));
```

**HTTP streaming:**
```javascript
const http = require('http');
const fs = require('fs');

http.createServer((req, res) => {
    const fileStream = fs.createReadStream('video.mp4');
    res.writeHead(200, { 'Content-Type': 'video/mp4' });
    fileStream.pipe(res);
}).listen(3000);
```

**Benefits:**
- Memory efficient (no full file in memory)
- Automatic backpressure handling
- Clean, readable code
- Chainable operations

**TL;DR:** Piping = connecting streams. stream.pipe(targetStream) for efficient data flow.

**Keyword/Key mappings:** Piping, Streams, pipe(), Backpressure, File I/O, Data Flow

---

### Q31: Explain the Reactor Pattern in Node.js.

**Answer (from given info):**
Reactor Pattern avoids blocking of I/O operations. Provides handler associated with I/O operations. Requests submitted to demultiplexer which handles concurrency, collects requests as events, queues them.

**Polished Answer:**
**Reactor Pattern** is the foundation of Node.js's event-driven architecture:

**Components:**
1. **Reactor (Event Loop)**: Dispatches I/O events to handlers
2. **Demultiplexer**: Monitors multiple I/O sources, notifies when ready
3. **Handler**: Processes the event
4. **Event Queue**: Holds pending events

**How it works:**
```
I/O Request → Demultiplexer → Event Queue → Event Loop → Handler
                ↑                                        |
                └────────────────────────────────────────┘
                        (non-blocking)
```

**Node.js implementation:**
```javascript
// Reactor pattern in action
const fs = require('fs');

// 1. I/O request submitted to demultiplexer
fs.readFile('file.txt', (err, data) => {
    // 4. Handler executes when data ready
    if (err) throw err;
    console.log(data);
});

// 2. Event loop continues processing
console.log('Doing other work...');

// 3. When file read completes, event queued
// 4. Event loop dequeues and executes callback
```

**Benefits:**
- Non-blocking I/O
- Efficient resource utilization
- Handles thousands of concurrent operations
- Single-threaded simplicity

**TL;DR:** Reactor pattern = demultiplexer monitors I/O, event loop dispatches to handlers.

**Keyword/Key mappings:** Reactor Pattern, Event Loop, Demultiplexer, I/O, Non-blocking, Event Queue

---

### Q32: What is the difference between spawn() and fork()?

**Answer (from given info):**
spawn() runs any system command, executes external programs, no communication channel by default. fork() creates new Node.js processes specifically, executes JavaScript files, creates built-in IPC channel for messaging.

**Polished Answer:**
**Comparison table:**

| Aspect | spawn() | fork() |
|--------|---------|--------|
| **Purpose** | Run system commands | Create Node.js child processes |
| **Process type** | Any executable | Node.js only |
| **Communication** | Streams (stdout/stderr) | IPC channel (send/receive) |
| **Use case** | External programs, shell commands | Node-to-Node communication |
| **Memory** | Separate process | Separate V8 instance |

**Example:**
```javascript
const { spawn, fork } = require('child_process');

// spawn - Run system command
const ls = spawn('ls', ['-la', '/tmp']);
ls.stdout.on('data', (data) => {
    console.log(`Output: ${data}`);
});

// fork - Run Node.js script with messaging
const child = fork('worker.js');

child.send({ task: 'process', data: [1, 2, 3] });

child.on('message', (result) => {
    console.log('Result:', result);
});

// worker.js
process.on('message', (message) => {
    const sum = message.data.reduce((a, b) => a + b, 0);
    process.send({ sum });
});
```

**TL;DR:** spawn = external commands, stream output. fork = Node.js processes, IPC messaging.

**Keyword/Key mappings:** spawn, fork, Child Processes, IPC, Streams, System Commands

---

### Q33: How do you read command line arguments?

**Answer (from given info):**
Command-line arguments are strings passed to program when running through command line interface. Read using process.argv global object.

**Polished Answer:**
**Reading CLI arguments** using `process.argv`:

```javascript
// index.js
console.log(process.argv);

// Run: node index.js hello world --port=3000
// Output:
// [
//   '/usr/bin/node',      // Node.js executable path
//   '/path/to/index.js',  // Script path
//   'hello',              // Argument 1
//   'world',              // Argument 2
//   '--port=3000'         // Argument 3
// ]
```

**Parsing arguments:**
```javascript
// Simple parsing
const args = process.argv.slice(2);
console.log('Arguments:', args);

// Parse named arguments
function parseArgs() {
    const args = {};
    process.argv.slice(2).forEach((arg) => {
        if (arg.startsWith('--')) {
            const [key, value] = arg.slice(2).split('=');
            args[key] = value || true;
        } else {
            args.positional = args.positional || [];
            args.positional.push(arg);
        }
    });
    return args;
}

// Using popular libraries
const yargs = require('yargs');
const argv = yargs
    .option('port', { alias: 'p', default: 3000 })
    .option('host', { alias: 'h', default: 'localhost' })
    .argv;

console.log(`Server on ${argv.host}:${argv.port}`);
```

**TL;DR:** Use process.argv for CLI args. Libraries like yargs/commander for complex parsing.

**Keyword/Key mappings:** CLI Arguments, process.argv, Command Line, yargs, Argument Parsing

---

### Q34: What is the purpose of module.exports and require()?

**Answer (from given info):**
module.exports exposes functions of a module to be used elsewhere. require() includes and imports modules into application. Supports encapsulation and project structure.

**Polished Answer:**
**Module system** enables code organization and reuse through exports and imports:

**Exporting (module.exports):**
```javascript
// utils.js
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;
const PI = 3.14159;

// Export as object
module.exports = { add, multiply, PI };

// Or export individually
exports.add = add;
exports.multiply = multiply;
exports.PI = PI;
```

**Importing (require):**
```javascript
// app.js
const utils = require('./utils');
console.log(utils.add(5, 3)); // 8

// Destructure imports
const { add, PI } = require('./utils');
console.log(add(2, 3)); // 5

// Import built-in modules
const fs = require('fs');
const http = require('http');

// Import npm packages
const express = require('express');

// Import from node_modules
const lodash = require('lodash');
```

**ES Modules alternative:**
```javascript
// export.js (using ESM)
export const add = (a, b) => a + b;
export default function multiply(a, b) { return a * b; }

// import.js (using ESM)
import { add } from './export.js';
import multiply from './export.js';
```

**CommonJS vs ES Modules:**
| Aspect | CommonJS | ES Modules |
|--------|----------|------------|
| Syntax | require/module.exports | import/export |
| Loading | Synchronous | Asynchronous |
| Default | Yes (Node.js default) | Requires .mjs or config |
| Use | Legacy | Modern |

**TL;DR:** module.exports = share code. require() = import code. Encapsulation for clean structure.

**Keyword/Key mappings:** module.exports, require, CommonJS, ES Modules, Import/Export, Encapsulation

---

### Q35: What are exit codes in Node.js?

**Answer (from given info):**
Exit codes indicate how a process terminated. Codes: Uncaught fatal exception (code 1), Unused (code 2), Fatal Error (code 5), Internal Exception handler failure (code 7), Internal JavaScript Evaluation Failure (code 4).

**Polished Answer:**
**Exit codes** communicate process termination status:

**Common exit codes:**
| Code | Meaning |
|------|---------|
| 0 | Success (normal exit) |
| 1 | Uncaught fatal exception |
| 2 | Unused (reserved by bash) |
| 3 | Internal JavaScript parse error |
| 4 | Internal JavaScript evaluation failure |
| 5 | Fatal error in V8 |
| 6 | Non-function internal exception handler |
| 7 | Internal exception handler runtime failure |
| 8 | Unused |
| 9 | Invalid argument |
| 10 | Internal JavaScript runtime failure |

**Usage:**
```javascript
// Successful exit
process.exit(0);

// Failure with code
process.exit(1);

// Exit handlers
process.on('exit', (code) => {
    console.log(`Process exiting with code: ${code}`);
});

// Check exit code in shell
// $ node script.js && echo "Success" || echo "Failed with $?"
```

**Graceful shutdown:**
```javascript
function gracefulShutdown(signal) {
    console.log(`${signal} received`);
    server.close(() => {
        console.log('Server closed');
        process.exit(0);
    });
    
    // Force exit after timeout
    setTimeout(() => {
        console.error('Forced shutdown');
        process.exit(1);
    }, 10000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

**TL;DR:** Exit codes show process termination reason. 0 = success, 1 = error, 5 = V8 fatal.

**Keyword/Key mappings:** Exit Codes, process.exit, Termination, Graceful Shutdown, SIGTERM, SIGINT

---

### Q36: What is the difference between process.nextTick() and setImmediate()?

**Answer (from given info):**
process.nextTick() executes in next tick queue, runs before I/O events, higher priority, can block I/O if overused. setImmediate() executes in check phase, runs after I/O events, lower priority.

**Polished Answer:**
**Key differences:**

| Aspect | process.nextTick() | setImmediate() |
|--------|-------------------|----------------|
| **Queue** | nextTick queue | Check queue |
| **Timing** | Before event loop continues | After I/O events |
| **Priority** | Higher | Lower |
| **I/O impact** | Can block I/O | Doesn't block I/O |
| **Use case** | Immediate cleanup, errors | Defer to next iteration |

**Execution order demonstration:**
```javascript
const fs = require('fs');

console.log('Start');

fs.readFile('file.txt', () => {
    console.log('I/O callback');
    
    process.nextTick(() => {
        console.log('nextTick inside I/O');
    });
    
    setImmediate(() => {
        console.log('setImmediate inside I/O');
    });
});

process.nextTick(() => {
    console.log('nextTick');
});

setImmediate(() => {
    console.log('setImmediate');
});

console.log('End');

// Output:
// Start
// End
// nextTick
// setImmediate
// I/O callback
// nextTick inside I/O
// setImmediate inside I/O
```

**When to use what:**
- `process.nextTick()`: Critical operations that must run before event loop continues
- `setImmediate()`: Defer execution until after pending I/O completes

**TL;DR:** nextTick runs before I/O, higher priority. setImmediate runs after I/O in check phase.

**Keyword/Key mappings:** process.nextTick, setImmediate, Priority, Event Loop, I/O, Check Phase

---

### Q37: What is the purpose of the url module?

**Answer (from given info):**
URL module splits URLs into readable parts for use in different application parts. parse() method separates URL into components.

**Polished Answer:**
**URL module** parses and constructs URLs:

**Legacy API (url.parse):**
```javascript
const url = require('url');

const urlString = 'https://user:pass@example.com:8080/path?query=string#hash';

const parsed = url.parse(urlString, true);
console.log(parsed);
// {
//   protocol: 'https:',
//   host: 'example.com:8080',
//   auth: 'user:pass',
//   hostname: 'example.com',
//   port: '8080',
//   pathname: '/path',
//   query: { query: 'string' },
//   hash: '#hash'
// }
```

**Modern API (URL class):**
```javascript
const { URL } = require('url');

const myURL = new URL('https://example.com/path?name=value');

console.log(myURL.hostname);    // 'example.com'
console.log(myURL.pathname);    // '/path'
console.log(myURL.searchParams.get('name')); // 'value'

// Manipulate URL
myURL.pathname = '/new-path';
myURL.searchParams.append('key', 'value');
console.log(myURL.toString());
```

**Express route URL parsing:**
```javascript
app.get('/users/:id', (req, res) => {
    console.log(req.params.id);    // Route parameter
    console.log(req.query.page);   // Query parameter
    console.log(req.path);         // URL path
    console.log(req.originalUrl);  // Full URL
});
```

**TL;DR:** url module = parse and construct URLs. Use URL class for modern approach.

**Keyword/Key mappings:** URL Module, parse(), URL Class, Query Parameters, Path

---

### Q38: What is body-parser and how do you parse request bodies?

**Answer (from given info):**
Body-parser is Node.js body-parsing middleware. Parses incoming request bodies before handling. NPM module that processes data sent in HTTP requests.

**Polished Answer:**
**Body parsing** extracts data from HTTP request bodies. Express has built-in body parsers (previously body-parser package).

**Types of body parsing:**
```javascript
const express = require('express');
const app = express();

// 1. JSON bodies
app.use(express.json());

// 2. URL-encoded bodies (forms)
app.use(express.urlencoded({ extended: true }));

// 3. Raw bodies
app.use(express.raw());

// 4. Text bodies
app.use(express.text());

// 5. Multipart (file uploads) - requires multer
const multer = require('multer');
const upload = multer();
app.use(upload.none());
```

**Accessing parsed data:**
```javascript
// POST with JSON body
app.post('/api/users', (req, res) => {
    console.log(req.body); // Parsed JSON object
    res.json(req.body);
});

// POST with form data
app.post('/api/login', (req, res) => {
    const { username, password } = req.body;
    res.json({ message: 'Login attempted' });
});
```

**Security considerations:**
```javascript
// Limit body size to prevent abuse
app.use(express.json({ limit: '10kb' }));

// Handle malformed JSON
app.use((err, req, res, next) => {
    if (err instanceof SyntaxError && err.status === 400) {
        return res.status(400).json({ error: 'Invalid JSON' });
    }
    next(err);
});
```

**TL;DR:** body-parser = request body parsing middleware. Use express.json(), express.urlencoded().

**Keyword/Key mappings:** body-parser, express.json, Request Body, Middleware, Parsing, multipart

---

### Q39: What is tracing in Node.js?

**Answer (from given info):**
Tracing Objects used for set of categories to enable/disable tracing. Created by calling tracing.enable() method, categories added to set of enabled traces, accessed by tracing.categories.

**Polished Answer:**
**Tracing** in Node.js provides insights into asynchronous operations for debugging and performance monitoring:

**Using async_hooks:**
```javascript
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook({
    init(asyncId, type, triggerAsyncId) {
        console.log(`Init: ${asyncId}, Type: ${type}, Trigger: ${triggerAsyncId}`);
    },
    before(asyncId) {
        console.log(`Before: ${asyncId}`);
    },
    after(asyncId) {
        console.log(`After: ${asyncId}`);
    },
    destroy(asyncId) {
        console.log(`Destroy: ${asyncId}`);
    }
});

hook.enable();
```

**Measuring async duration:**
```javascript
const { performance, PerformanceObserver } = require('perf_hooks');
const async_hooks = require('async_hooks');

const marks = new Map();

const hook = async_hooks.createHook({
    init(asyncId, type) {
        if (type === 'Timeout') {
            performance.mark(`Timeout-${asyncId}-Init`);
            marks.set(asyncId, type);
        }
    },
    destroy(asyncId) {
        if (marks.has(asyncId)) {
            performance.mark(`Timeout-${asyncId}-Destroy`);
            performance.measure(`Timeout-${asyncId}`,
                `Timeout-${asyncId}-Init`,
                `Timeout-${asyncId}-Destroy`);
        }
    }
});

hook.enable();

setTimeout(() => {}, 1000);
```

**Diagnostic tools:**
- `--trace-event-categories` CLI flag
- Chrome DevTools integration
- Performance observer
- async_hooks for async context

**TL;DR:** Tracing = monitoring async operations. Use async_hooks, perf_hooks for diagnostics.

**Keyword/Key mappings:** Tracing, async_hooks, perf_hooks, Performance, Diagnostics

---

### Q40: What is WASI and why was it introduced?

**Answer (from given info):**
WASI (WebAssembly System Interface) implemented through WASI API in Node.js using WASI class. Allows using underlying OS via POSIX-like functions, enabling applications to use resources efficiently.

**Polished Answer:**
**WASI (WebAssembly System Interface)** enables WebAssembly modules to access system resources:

**Purpose:**
- Run WebAssembly outside browsers
- Access OS capabilities (file system, networking)
- Standardized system interface for WASM
- Sandboxed execution environment

**Usage in Node.js:**
```javascript
const fs = require('fs');
const { WASI } = require('wasi');

const wasi = new WASI({
    args: process.argv,
    env: process.env,
    preopens: {
        '/sandbox': '/path/to/directory'
    }
});

const wasm = await WebAssembly.compile(fs.readFileSync('module.wasm'));
const instance = await WebAssembly.instantiate(wasm, wasi.getImportObject());

wasi.start(instance);
```

**Benefits:**
- Cross-platform WASM execution
- Secure sandboxing
- Access to system resources
- Language-agnostic (compile from C, Rust, Go)

**TL;DR:** WASI = WebAssembly system interface. Allows WASM access to OS resources.

**Keyword/Key mappings:** WASI, WebAssembly, System Interface, POSIX, Sandbox

---

### Q41: What are the pros and cons of Node.js?

**Answer (from given info):**
Pros: Non-blocking async I/O, high performance (V8 engine), active community, npm ecosystem, unified codebase. Cons: Single-threaded limitations, callback hell, NoSQL preference, rapid API changes.

**Polished Answer:**
**Advantages:**
| Pro | Description |
|-----|-------------|
| **Non-blocking I/O** | Handles multiple requests simultaneously |
| **V8 Performance** | Fast JavaScript execution |
| **NPM Ecosystem** | Largest package registry |
| **Active Community** | Large, supportive community |
| **Unified Language** | JavaScript on frontend and backend |
| **Real-time Apps** | Perfect for WebSocket, chat, gaming |

**Disadvantages:**
| Con | Description |
|-----|-------------|
| **Single-threaded** | Limited for CPU-intensive tasks |
| **Callback Hell** | Nested callbacks (mitigated by Promises) |
| **Relational DB support** | Better with NoSQL, weaker with SQL |
| **Rapid Changes** | API instability between versions |
| **Immature tooling** | Some libraries less stable than Java/Python |

**When to use Node.js:**
- ✅ Real-time applications
- ✅ API servers (JSON data)
- ✅ Microservices
- ✅ I/O-bound operations
- ❌ CPU-intensive processing
- ❌ Complex transactions

**TL;DR:** Node.js excels at I/O-heavy, real-time apps. Not ideal for CPU-intensive tasks.

**Keyword/Key mappings:** Pros, Cons, Non-blocking, V8 Engine, npm, Single-threaded, Callback Hell

---

### Q42: What is the difference between Node.js and Angular?

**Answer (from given info):**
Node.js is server-side runtime environment, used for backend development. Angular is front-end framework for building dynamic user interfaces. Node.js runs on server; Angular runs in browser.

**Polished Answer:**
**Comparison:**

| Aspect | Node.js | Angular |
|--------|---------|---------|
| **Type** | Server-side runtime | Front-end framework |
| **Language** | JavaScript | TypeScript |
| **Runs on** | Server | Browser |
| **Purpose** | APIs, servers, databases | User interfaces, SPAs |
| **Architecture** | Event-driven | Component-based |

**They work together:**
```
Angular (Frontend) ──HTTP──→ Node.js (Backend) ──→ Database
     Browser                    Server              Storage
```

**Example integration:**
```javascript
// Node.js backend API
app.get('/api/users', (req, res) => {
    res.json({ users: [{ id: 1, name: 'Alice' }] });
});
```

```typescript
// Angular frontend
@Injectable()
export class UserService {
    constructor(private http: HttpClient) {}
    
    getUsers(): Observable<User[]> {
        return this.http.get<User[]>('/api/users');
    }
}
```

**TL;DR:** Node.js = backend runtime. Angular = frontend framework. They complement each other.

**Keyword/Key mappings:** Node.js, Angular, Backend, Frontend, TypeScript, SPA

---

### Q43: What is the difference between Node.js and Python for backend?

**Answer (from given info):**
Node.js built on JavaScript, async by nature, non-blocking event-driven I/O, performant for I/O-heavy tasks. Python synchronous by default, thread-based concurrency, suitable for CPU-heavy operations.

**Polished Answer:**
**Comparison:**

| Aspect | Node.js | Python |
|--------|---------|--------|
| **Language** | JavaScript | Python |
| **Concurrency** | Event-driven, non-blocking | Thread-based, async with asyncio |
| **Performance** | Fast for I/O | Better for CPU-intensive |
| **Learning curve** | Easy for JS developers | Easy for beginners |
| **Ecosystem** | npm (largest) | pip (large) |
| **Use cases** | Real-time, APIs | Data science, ML, scripting |

**When to choose Node.js:**
- Real-time applications
- JSON APIs
- Streaming data
- JavaScript familiarity

**When to choose Python:**
- Machine learning/AI
- Data processing
- Scientific computing
- Rapid prototyping

**TL;DR:** Node.js for I/O-heavy real-time apps. Python for CPU-heavy, data science.

**Keyword/Key mappings:** Node.js, Python, Backend, Performance, I/O-bound, CPU-bound

---

### Q44: How do you handle database connections efficiently?

**Answer (from given info):**
Use drivers like mongoose for MongoDB, handle connection pooling for efficiency. Libraries provide methods to connect and execute queries.

**Polished Answer:**
**Efficient database connections** require proper management:

**1. Connection Pooling:**
```javascript
// MongoDB with pooling
mongoose.connect(uri, {
    maxPoolSize: 10,
    minPoolSize: 2,
    maxIdleTimeMS: 30000
});

// MySQL pooling
const pool = mysql.createPool({
    connectionLimit: 10,
    host: 'localhost',
    user: 'root',
    database: 'myapp'
});
```

**2. Singleton pattern:**
```javascript
// db.js
let connection = null;

async function getConnection() {
    if (!connection) {
        connection = await mongoose.connect(uri);
    }
    return connection;
}
```

**3. Graceful shutdown:**
```javascript
process.on('SIGTERM', async () => {
    await mongoose.connection.close();
    process.exit(0);
});
```

**4. Reconnection handling:**
```javascript
mongoose.connection.on('disconnected', () => {
    console.log('DB disconnected, retrying...');
    setTimeout(() => mongoose.connect(uri), 5000);
});
```

**TL;DR:** Use connection pooling, singleton pattern, handle disconnections, graceful shutdown.

**Keyword/Key mappings:** Connection Pooling, Database, Mongoose, MySQL, Singleton, Graceful Shutdown

---

### Q45: What is the purpose of NODE_ENV?

**Answer (from given info):**
NODE_ENV specifies environment of Node.js application. Distinguishes between development, testing, production. Allows customization of behavior based on environment.

**Polished Answer:**
**NODE_ENV** controls application behavior per environment:

```javascript
// Config based on environment
const config = {
    development: {
        debug: true,
        dbUrl: 'mongodb://localhost:27017/dev',
        logging: 'verbose'
    },
    production: {
        debug: false,
        dbUrl: process.env.PROD_DB_URL,
        logging: 'error'
    },
    test: {
        debug: false,
        dbUrl: 'mongodb://localhost:27017/test',
        logging: 'none'
    }
};

const currentConfig = config[process.env.NODE_ENV || 'development'];

// Conditional behavior
if (process.env.NODE_ENV === 'production') {
    app.use(helmet());
    app.use(compression());
    // Don't show error details
    app.use((err, req, res, next) => {
        res.status(500).json({ error: 'Internal server error' });
    });
} else {
    // Development error details
    app.use((err, req, res, next) => {
        res.status(500).json({ error: err.message, stack: err.stack });
    });
}
```

**Setting NODE_ENV:**
```bash
# Linux/Mac
NODE_ENV=production node app.js

# Windows
set NODE_ENV=production && node app.js

# Using dotenv
NODE_ENV=production  # in .env file
```

**TL;DR:** NODE_ENV = environment indicator. Use for config, error handling, optimization per environment.

**Keyword/Key mappings:** NODE_ENV, Environment, Development, Production, Configuration

---

### Q46: What is the difference between Duplex and Transform streams?

**Answer (from given info):**
Duplex streams are both readable and writable, e.g., TCP socket. Transform streams are duplex where data is transformed as read/written, e.g., zlib compression.

**Polished Answer:**
**Both are readable + writable, but Transform modifies data:**

| Aspect | Duplex | Transform |
|--------|--------|-----------|
| **Data flow** | Independent read/write | Output connected to input |
| **Transformation** | No modification | Modifies data |
| **Example** | TCP socket | Compression, encryption |
| **Use case** | Two-way communication | Data processing pipeline |

**Duplex example:**
```javascript
const { Duplex } = require('stream');

const duplex = new Duplex({
    read(size) {
        this.push('Data from read');
        this.push(null);
    },
    write(chunk, encoding, callback) {
        console.log('Received:', chunk.toString());
        callback();
    }
});
```

**Transform example:**
```javascript
const { Transform } = require('stream');

const upperCaseTransform = new Transform({
    transform(chunk, encoding, callback) {
        this.push(chunk.toString().toUpperCase());
        callback();
    }
});

// Using transform
process.stdin
    .pipe(upperCaseTransform)
    .pipe(process.stdout);
// Input: "hello" → Output: "HELLO"
```

**TL;DR:** Duplex = independent read/write. Transform = modifies data between read/write.

**Keyword/Key mappings:** Duplex, Transform, Streams, Read/Write, Data Pipeline

---

### Q47: What is Redis and how do you integrate it with Node.js?

**Answer (from given info):**
Redis is an open-source data structure store. Used as database, cache, and message broker. Stores strings, hashes, sets, sorted sets. Reduces cache size, makes applications efficient.

**Polished Answer:**
**Redis** is an in-memory data store used for caching, sessions, pub/sub:

**Integration with Node.js:**
```javascript
const redis = require('redis');
const client = redis.createClient({
    url: 'redis://localhost:6379'
});

client.on('error', (err) => console.error('Redis error:', err));
client.on('connect', () => console.log('Connected to Redis'));

await client.connect();
```

**Common use cases:**

**1. Caching:**
```javascript
async function getCachedData(key) {
    const cached = await client.get(key);
    if (cached) return JSON.parse(cached);
    
    const data = await fetchFromDatabase();
    await client.set(key, JSON.stringify(data), {
        EX: 3600 // Expire in 1 hour
    });
    return data;
}
```

**2. Session store:**
```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
    store: new RedisStore({ client: redisClient }),
    secret: 'secret',
    resave: false,
    saveUninitialized: false
}));
```

**3. Pub/Sub:**
```javascript
const subscriber = redis.createClient();
const publisher = redis.createClient();

subscriber.subscribe('notifications', (message) => {
    console.log('Received:', message);
});

publisher.publish('notifications', 'New user registered');
```

**4. Rate limiting:**
```javascript
async function rateLimiter(userId, limit, window) {
    const key = `rate:${userId}`;
    const count = await client.incr(key);
    
    if (count === 1) {
        await client.expire(key, window);
    }
    
    return count <= limit;
}
```

**TL;DR:** Redis = in-memory data store. Use for caching, sessions, pub/sub, rate limiting.

**Keyword/Key mappings:** Redis, Caching, Sessions, Pub/Sub, In-memory Store, Rate Limiting

---

### Q48: What is the difference between process-level and thread-level concurrency?

**Answer (from given info):**
Process-level concurrency runs multiple Node.js processes (cluster), each with own event loop and memory. Thread-level concurrency uses worker threads within single process, threads share memory, best for CPU-intensive tasks.

**Polished Answer:**
**Comparison:**

| Aspect | Process-level (Cluster) | Thread-level (Worker Threads) |
|--------|------------------------|------------------------------|
| **Unit** | Multiple processes | Multiple threads in one process |
| **Memory** | Separate memory space | Shared memory |
| **Communication** | IPC (messaging) | SharedArrayBuffer, postMessage |
| **Isolation** | Strong isolation | Weaker isolation |
| **Overhead** | Higher (process creation) | Lower (thread creation) |
| **Use case** | Scaling HTTP servers | CPU-intensive tasks |

**Process-level (Cluster):**
```javascript
if (cluster.isMaster) {
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
} else {
    // Each process handles HTTP requests
    http.createServer(handler).listen(8000);
}
```

**Thread-level (Worker Threads):**
```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    const worker = new Worker(__filename);
    worker.on('message', (result) => console.log(result));
    worker.postMessage({ data: largeArray });
} else {
    parentPort.on('message', (message) => {
        const result = processData(message.data);
        parentPort.postMessage(result);
    });
}
```

**TL;DR:** Process = separate memory, for scaling requests. Threads = shared memory, for CPU tasks.

**Keyword/Key mappings:** Process Concurrency, Thread Concurrency, Cluster, Worker Threads, IPC, Shared Memory

---

### Q49: How do you prevent event loop blocking?

**Answer (from given info):**
Offload CPU-heavy tasks to worker threads, use asynchronous non-blocking APIs, move background work to queues, avoid synchronous loops and heavy processing in routes.

**Polished Answer:**
**Preventing event loop blocking** keeps Node.js responsive:

**Common causes and solutions:**

**1. CPU-intensive tasks → Worker threads:**
```javascript
// Bad: Blocks event loop
app.get('/compute', (req, res) => {
    let sum = 0;
    for (let i = 0; i < 1e9; i++) sum += i;
    res.json({ sum });
});

// Good: Offload to worker
app.get('/compute', (req, res) => {
    const worker = new Worker('./compute-worker.js');
    worker.on('message', (result) => res.json(result));
    worker.postMessage({ iterations: 1e9 });
});
```

**2. Synchronous I/O → Asynchronous:**
```javascript
// Bad: Blocking
const data = fs.readFileSync('large-file.txt');

// Good: Non-blocking
fs.readFile('large-file.txt', (err, data) => {});
```

**3. Large loops → Chunked processing:**
```javascript
// Bad: Blocks for entire loop
for (let i = 0; i < 1e6; i++) {
    processItem(i);
}

// Good: Process in chunks
async function processInChunks(items, chunkSize = 1000) {
    for (let i = 0; i < items.length; i += chunkSize) {
        const chunk = items.slice(i, i + chunkSize);
        chunk.forEach(processItem);
        await new Promise(resolve => setImmediate(resolve)); // Yield to event loop
    }
}
```

**4. Heavy JSON processing → Stream or delegate:**
```javascript
// Bad: Parse entire JSON
const data = JSON.parse(hugeString);

// Good: Stream parsing or delegate to worker
```

**Detection:**
- Monitor event loop delay
- Check response times under load
- Profile for slow functions

**TL;DR:** Offload CPU tasks to workers, use async I/O, chunk large operations, delegate heavy work.

**Keyword/Key mappings:** Event Loop Blocking, Worker Threads, Async I/O, Chunked Processing, Performance

---

### Q50: How do you implement graceful shutdown in Node.js?

**Answer (from given info):**
Listen for SIGTERM/SIGINT signals, stop server from accepting new connections, allow in-flight requests to complete within timeout, close database connections and resources cleanly.

**Polished Answer:**
**Graceful shutdown** ensures no requests are dropped during restarts:

```javascript
const server = app.listen(3000);

let connections = [];

// Track connections
server.on('connection', (connection) => {
    connections.push(connection);
    connection.on('close', () => {
        connections = connections.filter(c => c !== connection);
    });
});

async function gracefulShutdown(signal) {
    console.log(`${signal} received, shutting down gracefully`);
    
    // 1. Stop accepting new connections
    server.close(() => {
        console.log('Server closed');
    });
    
    // 2. Close existing connections
    connections.forEach(connection => connection.end());
    
    // 3. Close database connections
    await mongoose.connection.close();
    
    // 4. Close Redis
    await redisClient.quit();
    
    // 5. Force exit after timeout
    setTimeout(() => {
        console.error('Forced shutdown after timeout');
        process.exit(1);
    }, 10000);
    
    // 6. Exit successfully
    process.exit(0);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

**Key considerations:**
- Drain in-flight requests (10-15s timeout)
- Close database connections
- Stop accepting new connections
- Log shutdown process
- Handle force shutdown gracefully

**TL;DR:** Graceful shutdown = stop new connections, drain existing, close resources, exit cleanly.

**Keyword/Key mappings:** Graceful Shutdown, SIGTERM, SIGINT, Server Close, Drain, Zero-downtime

---

## 🚀 TOP 100 - ADDITIONAL QUESTIONS (51-100)

---

### Q51: What are the different types of HTTP requests and their purposes?

**Answer (from given info):**
GET retrieves data, POST creates new resources, PUT updates entire resources, PATCH partially updates resources, DELETE removes resources.

**Polished Answer:**
HTTP methods define operations on resources. GET is safe and idempotent for reading. POST creates new resources. PUT replaces entire resources and is idempotent. PATCH does partial updates. DELETE removes resources and is idempotent. HEAD retrieves only headers. OPTIONS describes communication options. Each method has specific semantics that APIs should respect.

**TL;DR:** GET/read, POST/create, PUT/full update, PATCH/partial update, DELETE/remove.

**Keyword/Key mappings:** HTTP Methods, GET, POST, PUT, PATCH, DELETE, CRUD

---

### Q52: How do you handle environment variables safely across environments?

**Answer (from given info):**
Use environment variables for all environment-specific settings, avoid hard-coding secrets, validate configuration at startup, use secret managers for sensitive data.

**Polished Answer:**
Follow 12-factor principles: store config in environment variables, never commit .env files, validate required variables at startup, use secret managers (AWS Secrets Manager, HashiCorp Vault) for production secrets. Create config module that centralizes all environment access. Different .env files for different environments (.env.development, .env.production) with proper gitignore rules.

**TL;DR:** Env vars for config, validate at startup, secret managers for production, never commit secrets.

**Keyword/Key mappings:** Environment Variables, 12-factor, Configuration, Secret Management, dotenv

---

### Q53: What is the purpose of the net module?

**Answer (from given info):**
Net module creates TCP client and server. Establishes connections, handles incoming requests, shares data over network. Provides asynchronous network wrapper.

**Polished Answer:**
The net module provides low-level TCP networking. Create TCP servers that handle raw socket connections, implement custom protocols, create TCP clients that connect to servers. It's the foundation for higher-level protocols like HTTP and WebSocket. Supports socket events (data, end, close, error), connection management, and bidirectional data streaming.

**TL;DR:** net = TCP networking. Create servers/clients, raw socket communication.

**Keyword/Key mappings:** net, TCP, Sockets, Networking, Client, Server

---

### Q54: What is the DNS module used for?

**Answer (from given info):**
DNS module enables underlying OS name resolution and actual DNS lookup. Converts domain/subdomain names to IP addresses.

**Polished Answer:**
The DNS module provides name resolution capabilities. dns.lookup() uses OS facilities (may use hosts file). dns.resolve() performs actual DNS queries using DNS servers. Methods include resolve4 (IPv4), resolve6 (IPv6), resolveMx (mail servers), resolveCname (canonical names). Useful for building DNS-related tools, validating domains, and network diagnostics.

**TL;DR:** DNS = domain to IP resolution. lookup() uses OS, resolve() queries DNS servers.

**Keyword/Key mappings:** DNS, Name Resolution, lookup, resolve, IP Address

---

### Q55: What is the OS module?

**Answer (from given info):**
OS module provides operating system-based utility modules. Provides OS information and utilities.

**Polished Answer:**
The OS module provides system information: cpus() returns CPU details, totalmem/freemem show memory, platform shows OS type, hostname shows system name, userInfo shows current user. Useful for monitoring, clustering decisions (number of cores), and system-specific behavior. Commonly used to determine optimal worker thread count or cluster size.

**TL;DR:** OS module = system info. CPUs, memory, platform, hostname.

**Keyword/Key mappings:** OS Module, System Info, CPUs, Memory, Platform

---

### Q56: What is the path module?

**Answer (from given info):**
Path module transforms and handles various file paths. Handles file path operations.

**Polished Answer:**
The path module handles file paths across different operating systems. join() combines paths with correct separators, resolve() creates absolute paths, dirname/basename extract components, extname gets file extension, parse splits paths into objects. Critical for cross-platform compatibility since Windows and Unix use different separators.

**TL;DR:** path = file path handling. join, resolve, dirname, basename, extname.

**Keyword/Key mappings:** path, File Paths, Cross-platform, join, resolve

---

### Q57: How do worker threads differ from clusters?

**Answer (from given info):**
Clusters: one process per CPU with IPC, multiple servers on single port, separate memory. Worker threads: one process with multiple threads, shared memory, for CPU-intensive tasks.

**Polished Answer:**
Clusters create multiple processes each with own Node.js instance and memory space. Best for scaling HTTP request handling across CPU cores. Worker threads create multiple threads within one process sharing memory (SharedArrayBuffer). Best for offloading CPU-intensive computation without process overhead. Clusters provide better isolation while workers provide better memory efficiency and communication speed.

**TL;DR:** Cluster = multiple processes for scaling. Workers = multiple threads for CPU tasks, shared memory.

**Keyword/Key mappings:** Worker Threads, Cluster, Shared Memory, Multi-core, IPC

---

### Q58: How do you measure performance of async operations?

**Answer (from given info):**
Performance API provides tools for performance metrics. Use perf_hooks and async_hooks for measurement.

**Polished Answer:**
Use performance.now() for high-resolution timestamps, performance.mark() to create named markers, performance.measure() to calculate duration between marks. PerformanceObserver watches for measurements. For async operations, use async_hooks to track async lifecycle. Tools like clinic.js provide visual analysis. Key metrics include event loop delay, GC pauses, and operation duration.

**TL;DR:** Use perf_hooks for timing, async_hooks for async tracking, performance.measure for duration.

**Keyword/Key mappings:** Performance, perf_hooks, performance.measure, async_hooks, Timing

---

### Q59: What is the difference between require() and import?

**Answer (from given info):**
require() is CommonJS, dynamic loading. import is ES Modules, static loading. Node.js supports both.

**Polished Answer:**
CommonJS (require) is Node.js's original module system, loads synchronously, can be called conditionally, supports dynamic paths. ES Modules (import) is the JavaScript standard, loads asynchronously, hoisted to top, supports static analysis and tree-shaking. ES Modules require .mjs extension or "type": "module" in package.json. Both systems coexist in modern Node.js.

**TL;DR:** require = CommonJS, synchronous, dynamic. import = ESM, asynchronous, static.

**Keyword/Key mappings:** require, import, CommonJS, ES Modules, Module System

---

### Q60: What are first-class functions in JavaScript?

**Answer (from given info):**
Functions treated like any other variable. Can be passed as parameters (callbacks) or returned from functions (higher-order functions).

**Polished Answer:**
First-class functions mean functions are values that can be assigned to variables, passed as arguments, returned from functions, and stored in data structures. This enables callbacks, higher-order functions (map, filter, reduce), closures, and functional programming patterns. It's the foundation of Node.js's callback-based async model and event handling.

**TL;DR:** Functions as values. Pass as arguments, return from functions, assign to variables.

**Keyword/Key mappings:** First-class Functions, Callbacks, Higher-order Functions, Closures

---

### Q61: What is the purpose of async.queue?

**Answer (from given info):**
async.queue takes task function and concurrency value as input. Manages queue of async tasks with limited concurrency.

**Polished Answer:**
async.queue manages concurrent execution of async tasks with a configurable concurrency limit. Takes a worker function and concurrency value. Provides methods like push() to add tasks, drain() callback when queue empties, and error handling. Useful for rate-limiting operations, processing batches, and controlling resource usage.

**TL;DR:** async.queue = task queue with concurrency limit. Prevents overwhelming resources.

**Keyword/Key mappings:** async.queue, Concurrency, Task Queue, Rate Limiting, Async.js

---

### Q62: How do you implement rate limiting?

**Answer (from given info):**
Rate limiting implemented as middleware tracking requests per client, enforcing limits over time windows, rejecting when limits exceeded. Often backed by Redis in production.

**Polished Answer:**
Implement rate limiting middleware that tracks request counts per client (IP, user, token) over a time window. Use Redis for distributed rate limiting across multiple instances. Configure different limits for authenticated vs unauthenticated users. Return 429 status with retry-after header when exceeded. Consider sliding window or token bucket algorithms for smoother limiting.

**TL;DR:** Rate limiting = middleware tracking requests, Redis for distributed, 429 when exceeded.

**Keyword/Key mappings:** Rate Limiting, Middleware, Redis, 429, Throttling

---

### Q63: What is the trust proxy setting in Express?

**Answer (from given info):**
trust proxy tells Express to trust headers from reverse proxy like X-Forwarded-For when determining client IP.

**Polished Answer:**
Behind load balancers or reverse proxies, Express sees proxy IP instead of client IP. trust proxy setting tells Express to trust X-Forwarded-For headers to correctly determine client IP. Set true when behind one proxy, specify IP/subnet for specific proxies, or number of proxies. Affects req.ip, req.ips, and secure cookie detection. Incorrect configuration can allow IP spoofing.

**TL;DR:** trust proxy = trust X-Forwarded-For for real client IP behind proxies.

**Keyword/Key mappings:** trust proxy, X-Forwarded-For, Reverse Proxy, Client IP, Load Balancer

---

### Q64: How do you implement file uploads securely?

**Answer (from given info):**
Use Multer for file uploads. Enforce size limits, validate file types, store outside web root, rename files, scan for malware.

**Polished Answer:**
Secure file uploads require: size limits to prevent DoS, file type validation (MIME + extension + magic bytes), unique filename generation to prevent collisions, store outside web root to prevent execution, proper permissions, malware scanning. Use Multer middleware with disk or memory storage. Consider cloud storage (S3) for scalability. Always validate before processing.

**TL;DR:** Secure uploads = validate type/size, unique names, store safely, scan for malware.

**Keyword/Key mappings:** File Upload, Multer, Security, Validation, Size Limits, Malware Scanning

---

### Q65: What are common causes of event loop blocking?

**Answer (from given info):**
CPU-intensive computations, large synchronous loops, heavy JSON parsing, expensive regex, blocking file system calls.

**Polished Answer:**
Event loop blocking occurs from synchronous operations on the main thread: CPU-heavy computations (crypto, compression), large array operations, JSON.stringify/parse on big objects, complex regex (catastrophic backtracking), synchronous file I/O (readFileSync), synchronous DB queries. Symptoms include increased latency, timeouts, and unresponsive server. Detect via event loop delay monitoring and profiling.

**TL;DR:** Blocking = sync operations on main thread. CPU tasks, sync I/O, large processing.

**Keyword/Key mappings:** Event Loop Blocking, Synchronous, CPU-intensive, Profiling, Latency

---

### Q66: How do you diagnose memory leaks?

**Answer (from given info):**
Monitor heap usage over time, compare heap snapshots, check for unbounded caches, global references, event listeners, closures.

**Polished Answer:**
Memory leak diagnosis: monitor heap usage (should stabilize after warm-up), take heap snapshots at intervals and compare, identify retained objects, check for unbounded caches without TTL, global variable accumulation, unreleased event listeners, closures holding references, database connections not released. Use tools: Chrome DevTools inspector, clinic doctor, heapdump module. Correlate with specific features or traffic patterns.

**TL;DR:** Memory leak = growing heap. Compare snapshots, find retained objects, check caches/listeners.

**Keyword/Key mappings:** Memory Leaks, Heap, Snapshots, GC, Retained Objects, Event Listeners

---

### Q67: What metrics should you monitor for Node.js services?

**Answer (from given info):**
Event loop delay, request latency (p50/p95/p99), throughput, error rates, GC frequency, thread pool usage, database latency.

**Polished Answer:**
Essential metrics: event loop delay (detects blocking), request latency percentiles (p50, p95, p99), throughput (requests/sec), error rate, GC frequency/duration (indicates memory pressure), thread pool utilization, database/external dependency latency, queue depth for background jobs. Use tools: Prometheus + Grafana, Datadog, New Relic, AppDynamics. Set up alerting on key metrics.

**TL;DR:** Monitor: event loop delay, latency, throughput, errors, GC, dependencies.

**Keyword/Key mappings:** Metrics, Monitoring, Event Loop Delay, Latency, Throughput, GC

---

### Q68: What is backpressure in streams?

**Answer (from given info):**
Backpressure prevents memory from growing when consumers can't keep up with producers. Respecting write() return values ensures safe data flow.

**Polished Answer:**
Backpressure occurs when a readable stream produces data faster than a writable stream can consume. Streams use internal buffers, but if not handled, memory grows unbounded. Implementation: writable.write() returns false when buffer is full, pause readable stream, resume on 'drain' event. pipe() handles this automatically. Manual implementation requires respecting return values and listening for drain events.

**TL;DR:** Backpressure = slow consumer, pause producer. Use pipe() or handle write() returns.

**Keyword/Key mappings:** Backpressure, Streams, Buffer, drain, pipe, Memory Management

---

### Q69: How do you design retries and timeouts?

**Answer (from given info):**
Set strict timeouts, limit retry attempts, exponential backoff with jitter, avoid retries for non-idempotent operations, combine with circuit breakers.

**Polished Answer:**
Retry design: always set timeouts on external calls (avoid hanging), limit retry attempts (2-3 max), exponential backoff (2^n) with jitter (random delay to prevent retry storms), only retry idempotent operations (GET, PUT - not POST), use circuit breakers to stop retrying failing dependencies. Implement with libraries like opossum or custom logic. Log retries for monitoring.

**TL;DR:** Retries = timeouts + limited attempts + backoff + jitter + circuit breaker. No retries for POST.

**Keyword/Key mappings:** Retries, Timeouts, Exponential Backoff, Jitter, Circuit Breaker, Idempotent

---

### Q70: What are circuit breakers and bulkheads?

**Answer (from given info):**
Circuit breakers stop calling failing dependencies after repeated errors. Bulkheads isolate resources so one failing component doesn't exhaust the system.

**Polished Answer:**
Circuit breaker pattern prevents cascading failures by monitoring dependency health: closed (normal operation), open (after threshold failures, stop calling), half-open (test after cooldown). Bulkhead pattern isolates resources (connection pools, threads, queues) per dependency so one failing service doesn't consume all resources. Implement with opossum library or manually with state tracking.

**TL;DR:** Circuit breaker = stop calling failing deps. Bulkhead = isolate resources per dependency.

**Keyword/Key mappings:** Circuit Breaker, Bulkhead, Resilience, Cascading Failures, opossum

---

### Q71: What is the difference between cache growth and memory leak?

**Answer (from given info):**
Cache growth stabilizes after warm-up with limits/eviction. Memory leak grows steadily without release. Cache has TTL/size limits, leak has unbounded retention.

**Polished Answer:**
Cache growth is intentional: memory increases until cache fills (bounded by max size/TTL), then stabilizes or cycles. Memory leak is unintentional: heap grows continuously without release. Distinguish by: checking if growth stabilizes, reviewing cache eviction policies, heap snapshots showing retained objects, GC behavior (frequent GC indicates pressure). Fix leaks by removing references, using WeakMap, clearing listeners.

**TL;DR:** Cache = bounded growth, stabilizes. Leak = unbounded growth, never stabilizes.

**Keyword/Key mappings:** Cache, Memory Leak, Heap Growth, GC, Eviction Policy

---

### Q72: How do you separate DB latency from CPU blocking?

**Answer (from given info):**
Review database query timings, check event loop delay, inspect network latency, compare response times with/without DB access.

**Polished Answer:**
Isolate performance bottlenecks by: checking DB query timings independently (logging, APM), measuring event loop delay (if high, CPU blocking; if low, DB/network), comparing response time with/without DB calls, inspecting connection pool metrics (exhausted pool indicates DB slowness), reviewing slow query logs, using distributed tracing. Separate middleware timings to isolate components.

**TL;DR:** Measure separately: event loop delay (CPU), query timings (DB), network latency.

**Keyword/Key mappings:** Performance, DB Latency, CPU Blocking, Event Loop Delay, Profiling

---

### Q73: What are exit codes 1, 5, and 7 in Node.js?

**Answer (from given info):**
Code 1: Uncaught fatal exception. Code 5: Fatal error in V8. Code 7: Internal exception handler runtime failure.

**Polished Answer:**
Exit code 1 indicates uncaught fatal exception (unhandled error crashed process). Code 5 indicates V8 engine fatal error with stderr output describing the error. Code 7 indicates exception occurred when internal exception handler was called (rare, indicates Node.js internal issue). Understanding these codes helps diagnose crashes. Code 0 is normal exit.

**TL;DR:** Exit 1 = uncaught exception, 5 = V8 fatal, 7 = internal handler failure, 0 = success.

**Keyword/Key mappings:** Exit Codes, process.exit, Fatal Error, V8, Uncaught Exception

---

### Q74: What is the purpose of the passport module?

**Answer (from given info):**
Passport module adds authentication features. Implements authentication measures for sign-in operations.

**Polished Answer:**
Passport is authentication middleware for Node.js providing 500+ strategies. Handles local authentication (username/password), OAuth (Google, Facebook, GitHub), JWT, SAML. Integrates with Express via middleware. Manages session serialization/deserialization. Separates authentication logic from application logic. Extensible via custom strategies.

**TL;DR:** Passport = authentication middleware with multiple strategies (local, OAuth, JWT).

**Keyword/Key mappings:** Passport, Authentication, OAuth, Strategies, Middleware

---

### Q75: How do you manage sessions in Node.js?

**Answer (from given info):**
Use express-session module. Saves data in key-value form. Session data not in cookie, just session ID.

**Polished Answer:**
Session management stores user state server-side with only session ID in cookie. Use express-session with configurable stores (memory for dev, Redis/MongoDB for production). Set secure cookies (httpOnly, secure in production, sameSite). Handle session expiry, regenerate on privilege change (login), destroy on logout. Scale sessions with shared stores across instances.

**TL;DR:** Sessions = server-side state, ID in cookie. Use Redis store for scalability.

**Keyword/Key mappings:** Sessions, express-session, Session ID, Cookie, Redis Store

---

### Q76: What is the difference between authentication and authorization?

**Answer (from given info):**
Authentication verifies user identity. Authorization determines access rights to resources.

**Polished Answer:**
Authentication answers "Who are you?" - verifying identity through credentials (password, token, biometrics). Authorization answers "What can you do?" - determining permissions after authentication (roles, access levels, resource permissions). Implement authentication with JWT/Passport/sessions. Implement authorization with middleware checking roles/permissions. Both essential for security, authentication precedes authorization.

**TL;DR:** AuthN = identity verification. AuthZ = permission checking. Both required.

**Keyword/Key mappings:** Authentication, Authorization, Identity, Permissions, Roles

---

### Q77: What are the different types of streams?

**Answer (from given info):**
Four types: Readable (read data), Writable (write data), Duplex (both), Transform (duplex with transformation).

**Polished Answer:**
Readable streams produce data (file read, HTTP request body). Writable streams consume data (file write, HTTP response). Duplex streams both produce and consume independently (TCP socket). Transform streams modify data between reading and writing (compression, encryption). All streams are EventEmitters. Use pipe() for connecting streams. Handle errors and backpressure.

**TL;DR:** 4 stream types: Readable, Writable, Duplex, Transform. All EventEmitters.

**Keyword/Key mappings:** Streams, Readable, Writable, Duplex, Transform, EventEmitter

---

### Q78: How do you handle large JSON payloads efficiently?

**Answer (from given info):**
Stream parsing instead of loading entire JSON, use worker threads for processing, limit payload size.

**Polished Answer:**
Efficient large JSON handling: use streaming parsers (JSONStream, stream-json) to process incrementally, avoid JSON.parse on entire payload, offload parsing to worker threads, set payload size limits (express.json({ limit: '1mb' })), consider alternative formats for very large data (protobuf, MessagePack), use database directly instead of JSON for large datasets.

**TL;DR:** Stream JSON parsing, limit size, use workers, consider binary formats.

**Keyword/Key mappings:** JSON, Streaming, Large Payloads, Size Limits, Worker Threads

---

### Q79: What is the role of libuv in Node.js?

**Answer (from given info):**
Libuv provides thread pool and handles background tasks like file system operations and networking.

**Polished Answer:**
Libuv is a C library that provides Node.js's cross-platform async I/O capabilities. Manages thread pool (default 4 threads) for expensive operations (file I/O, DNS, crypto). Implements event loop. Provides abstractions for networking, file system, process management across OSes. Handles asynchronous operations queuing and completion callbacks. Critical for non-blocking I/O implementation.

**TL;DR:** libuv = thread pool + event loop implementation + cross-platform async I/O.

**Keyword/Key mappings:** libuv, Thread Pool, Event Loop, Async I/O, Cross-platform

---

### Q80: What is the purpose of the process object?

**Answer (from given info):**
Process object provides information about current Node.js process. Global object with runtime information.

**Polished Answer:**
The process object is a global providing runtime information and control: process.env for environment variables, process.argv for CLI arguments, process.pid for process ID, process.exit() to terminate, process.on() for event handling, process.memoryUsage() for memory info, process.cwd() for working directory, process.nextTick() for immediate execution. Essential for environment configuration and process management.

**TL;DR:** process = runtime info and control. env, argv, exit, events, memory usage.

**Keyword/Key mappings:** process, Global Object, Environment, CLI Arguments, Process Control

---

### Q81: How do you implement logging in Node.js?

**Answer (from given info):**
Structured logs with adjustable levels, correlation IDs across services, centralized logging.

**Polished Answer:**
Implement structured logging with libraries like winston or pino. Use log levels (error, warn, info, debug). Include correlation IDs for request tracing. JSON format for machine parsing. Log to stdout (containers, cloud), aggregate with ELK stack or cloud services. Include context (user, request ID, timestamp). Avoid logging sensitive data. Sample logs in high-volume scenarios.

**TL;DR:** Use winston/pino, JSON format, correlation IDs, central aggregation.

**Keyword/Key mappings:** Logging, Structured Logging, winston, pino, Correlation IDs

---

### Q82: What are correlation IDs?

**Answer (from given info):**
Correlation IDs track single request across services by generating/propagating request ID at edge, attaching to logs and downstream calls.

**Polished Answer:**
Correlation IDs uniquely identify requests flowing through distributed systems. Generate or receive at edge, attach to all logs and downstream HTTP calls (via headers like X-Request-ID). Middleware generates/validates ID, adds to request context. All logs include ID for tracing request path through microservices. Essential for debugging distributed systems.

**TL;DR:** Correlation ID = request identifier propagated across services for tracing.

**Keyword/Key mappings:** Correlation ID, Request Tracing, Distributed Systems, Middleware

---

### Q83: How do you implement API versioning?

**Answer (from given info):**
URI versioning (/v1/users), header-based versioning, content negotiation with Accept header.

**Polished Answer:**
API versioning approaches: URI versioning (/api/v1/users) - simple, explicit, most common. Header-based (Accept-Version or custom header) - clean URLs, requires client handling. Content negotiation (Accept header with versioned media types) - flexible but complex. Choose URI versioning for simplicity, header for cleaner URLs. Maintain backward compatibility within major versions. Document deprecation policies.

**TL;DR:** Version APIs via URI (/v1/), header, or content negotiation. URI is simplest.

**Keyword/Key mappings:** API Versioning, URI Versioning, Header Versioning, Content Negotiation

---

### Q84: What is the purpose of the timers module?

**Answer (from given info):**
Timers module contains functions to execute code after set period: setTimeout, setImmediate, setInterval.

**Polished Answer:**
Timers module provides scheduling functions: setTimeout executes once after delay, setInterval executes repeatedly with fixed delay, setImmediate executes after current event loop iteration, clearTimeout/clearInterval cancel timers. Timers are global (no require needed). setTimeout with 0ms defers to next timer phase. setInterval can drift; use recursive setTimeout for accurate intervals.

**TL;DR:** Timers = setTimeout, setInterval, setImmediate, clear functions.

**Keyword/Key mappings:** Timers, setTimeout, setInterval, setImmediate, Scheduling

---

### Q85: How do you implement idempotency for POST endpoints?

**Answer (from given info):**
Require idempotency key from client, store key with request result, return same response for repeated requests.

**Polished Answer:**
Idempotency ensures repeated requests don't cause duplicate effects (critical for payments). Client sends Idempotency-Key header. Server stores key with result in database (Redis). On receiving same key, return stored result without reprocessing. Set TTL for keys. Implement as middleware before business logic. Generate response once, cache, replay on retries.

**TL;DR:** Idempotency = store key + result, replay on repeat, prevents duplicate operations.

**Keyword/Key mappings:** Idempotency, POST, Idempotency Key, Payment APIs, Retries

---

### Q86: What is the difference between spawn() and exec()?

**Answer (from given info):**
spawn() streams output, good for large data. exec() buffers output, good for small data.

**Polished Answer:**
spawn() creates child process with streaming stdout/stderr, handles large outputs efficiently, doesn't create shell by default. exec() runs command in shell, buffers entire output, callback receives buffer (limited to maxBuffer size), suitable for small outputs. Use spawn for large data streams, exec for simple commands with small output. exec supports shell syntax (pipes, redirects), spawn requires array arguments.

**TL;DR:** spawn = streaming, large data. exec = buffered, small data, shell syntax.

**Keyword/Key mappings:** spawn, exec, Child Processes, Streaming, Buffering

---

### Q87: How do you handle static files in Express?

**Answer (from given info):**
Use express.static middleware. Configure caching, correct paths.

**Polished Answer:**
Serve static files using express.static('public') middleware. Place before route handlers. Configure maxAge for caching. Handle multiple static directories. Serve with virtual path prefix. Security: don't serve sensitive files (source code, .env), use correct directory, configure caching headers. For production, serve via CDN or reverse proxy (nginx) for better performance.

**TL;DR:** express.static = static file serving. Configure cache, secure paths.

**Keyword/Key mappings:** Static Files, express.static, Middleware, Caching, CDN

---

### Q88: What is the difference between route params, query params, and body?

**Answer (from given info):**
Route params are in URL path (req.params). Query params in URL (?key=value). Body in request payload (req.body).

**Polished Answer:**
Route params (/users/:id) identify resources, accessed via req.params, mandatory in URL. Query params (/users?page=2) filter/sort/paginate, accessed via req.query, optional. Request body (POST/PUT data) contains data payload, accessed via req.body, requires parsing middleware. Use route params for resource identity, query params for options, body for data.

**TL;DR:** Route params = resource ID in path. Query = filters in URL. Body = data payload.

**Keyword/Key mappings:** Route Params, Query Params, Request Body, req.params, req.query, req.body

---

### Q89: What is express.Router()?

**Answer (from given info):**
express.Router() creates modular route handlers. Groups routes and middleware by feature.

**Polished Answer:**
express.Router() creates isolated router instances for modular routing. Group routes by feature/domain. Apply router-specific middleware. Mount with app.use('/users', userRouter). Benefits: code organization, reusable route modules, team ownership, cleaner main app file. Each router maintains its own middleware stack.

**TL;DR:** Router = modular routing. Group by feature, mount with app.use().

**Keyword/Key mappings:** express.Router, Modular Routing, Middleware, Route Organization

---

### Q90: How do you handle synchronous vs asynchronous errors?

**Answer (from given info):**
Sync errors thrown directly, caught automatically by Express. Async errors must be passed via next(err).

**Polished Answer:**
Synchronous errors in route handlers are caught automatically by Express and forwarded to error middleware. Asynchronous errors (inside promises/async functions) must be handled explicitly: use try-catch with next(err), use async wrapper functions, or use libraries that handle async errors. Express 5 will auto-catch rejected promises. Unhandled async errors cause unhandled promise rejections.

**TL;DR:** Sync errors auto-caught. Async errors need next(err) or wrapper.

**Keyword/Key mappings:** Error Handling, Synchronous, Asynchronous, next(err), async/await

---

### Q91: What is the test pyramid?

**Answer (from given info):**
Test pyramid structure: Unit tests at base (most numerous), Integration tests in middle, E2E tests at top (fewest).

**Polished Answer:**
Test pyramid guides testing strategy: many unit tests (fast, isolated), fewer integration tests (test component interactions), fewest E2E tests (slow, full system). Unit tests verify individual functions. Integration tests verify API routes with database. E2E tests verify user flows. More low-level tests = faster feedback. Balance coverage with execution time.

**TL;DR:** Many unit tests, some integration, few E2E. Faster feedback with more unit tests.

**Keyword/Key mappings:** Test Pyramid, Unit Testing, Integration Testing, E2E Testing

---

### Q92: How do you implement request validation?

**Answer (from given info):**
Validation as middleware before controllers. Validate params, query, body. Use express-validator or Joi.

**Polished Answer:**
Request validation ensures data integrity and security. Implement as middleware before business logic. Validate route params, query strings, and request bodies. Use express-validator (integrates with Express) or Joi (schema-based). Return consistent error responses (400 with details). Validate types, formats, required fields, ranges. Sanitize inputs to prevent injection. Keep validation rules reusable and versioned.

**TL;DR:** Validate in middleware with express-validator/Joi. Return 400 with error details.

**Keyword/Key mappings:** Validation, express-validator, Joi, Middleware, Schema Validation

---

### Q93: What is the purpose of NODE_ENV in Express apps?

**Answer (from given info):**
NODE_ENV specifies environment, customizes behavior, error handling, optimization.

**Polished Answer:**
NODE_ENV controls environment-specific behavior in Express apps. Development: verbose errors, hot reload, debug logging. Production: minimal errors, compression, security headers, caching. Test: mocking, test database. Use to conditionally enable middleware, configure error messages, set cache headers, choose database connections. Set via process.env or deployment configuration.

**TL;DR:** NODE_ENV = environment config. Different behavior per environment.

**Keyword/Key mappings:** NODE_ENV, Environment, Development, Production, Configuration

---

### Q94: How do you prevent SQL injection in Node.js?

**Answer (from given info):**
Use parameterized queries, validate inputs, sanitize data.

**Polished Answer:**
Prevent SQL injection by using parameterized queries (never string concatenation), ORM/query builders with built-in protection (Sequelize, Knex), input validation and sanitization, escape special characters, use least-privilege database users, stored procedures. Example: pool.query('SELECT * FROM users WHERE id = ?', [userId]). Never trust user input.

**TL;DR:** Parameterized queries, ORM, input validation. Never concatenate user input.

**Keyword/Key mappings:** SQL Injection, Parameterized Queries, Input Validation, Security

---

### Q95: What is the purpose of Helmet middleware?

**Answer (from given info):**
Helmet sets secure HTTP headers to protect against common attacks.

**Polished Answer:**
Helmet is middleware that sets security-related HTTP headers: Content-Security-Policy (XSS protection), X-Frame-Options (clickjacking), HSTS (HTTPS enforcement), X-Content-Type-Options (MIME sniffing), Referrer-Policy, X-Permitted-Cross-Domain-Policies. Configure per header or use defaults. Essential for Express apps in production. Reduces attack surface against common vulnerabilities.

**TL;DR:** Helmet = security headers middleware. Protects against XSS, clickjacking, sniffing.

**Keyword/Key mappings:** Helmet, Security Headers, XSS, Clickjacking, CSP, HSTS

---

### Q96: How do you handle file downloads and streaming responses?

**Answer (from given info):**
Stream responses using Node.js streams, pipe file to response, handle backpressure.

**Polished Answer:**
Streaming responses avoids loading entire file in memory. Use res.download() for file downloads with correct headers (Content-Disposition). Use fs.createReadStream().pipe(res) for large files. Handle backpressure automatically. Set appropriate headers (Content-Type, Content-Length, Content-Range). Support range requests for partial downloads (video streaming). Handle client disconnects and errors.

**TL;DR:** Stream files with createReadStream().pipe(res) or res.download().

**Keyword/Key mappings:** Streaming, Downloads, res.download, File Streaming, Content-Disposition

---

### Q97: What is the difference between clusters and worker threads for CPU tasks?

**Answer (from given info):**
Clusters: multiple processes, separate memory, for scaling HTTP. Workers: threads sharing memory, for CPU tasks.

**Polished Answer:**
For CPU-intensive tasks, worker threads are preferred: shared memory reduces overhead, faster communication, no IPC serialization. Clusters are better for scaling HTTP request handling across cores: better isolation, no shared state issues. Workers can use SharedArrayBuffer for fast data sharing. Clusters simpler for horizontally scaling stateless services. Choose based on task nature: workers for computation, clusters for request handling.

**TL;DR:** CPU tasks = worker threads (shared memory). HTTP scaling = clusters (isolation).

**Keyword/Key mappings:** Cluster, Worker Threads, CPU Tasks, Shared Memory, Scaling

---

### Q98: How do you implement caching in Express apps?

**Answer (from given info):**
Set cache headers (Cache-Control, ETag), implement Redis caching, invalidate on writes.

**Polished Answer:**
Implement caching at multiple levels: HTTP caching with Cache-Control headers (public/private, max-age), ETag for conditional requests, Redis for application-level caching, CDN for static assets. Cache key includes relevant dimensions (params, user). Invalidate cache on writes (purge, versioning, TTL). Don't cache user-specific or frequently changing data. Monitor cache hit rates.

**TL;DR:** Cache headers for HTTP, Redis for app data, CDN for static. Invalidate on writes.

**Keyword/Key mappings:** Caching, Cache-Control, ETag, Redis, CDN, Invalidation

---

### Q99: What is the purpose of the Performance Observer?

**Answer (from given info):**
PerformanceObserver watches for performance measurements. Used with perf_hooks for monitoring.

**Polished Answer:**
PerformanceObserver asynchronously observes performance measurements. Subscribe to entry types (mark, measure, resource). Callback receives performance entries. Use for monitoring operation durations, detecting performance regressions. Combined with performance.mark() and performance.measure() for custom timing. Buffer entries until observer ready. Useful for building performance monitoring tools.

**TL;DR:** PerformanceObserver = async monitoring of performance measurements.

**Keyword/Key mappings:** PerformanceObserver, perf_hooks, performance.measure, Monitoring

---

### Q100: How do you handle distributed transactions in microservices?

**Answer (from given info):**
Use saga pattern, event-driven architecture, idempotency, compensation transactions.

**Polished Answer:**
Distributed transactions in microservices use saga pattern: choreographed (event-driven, services react to events) or orchestrated (central coordinator manages steps). Each step has compensation action for rollback. Implement idempotency for all operations. Use message queues for reliable delivery. Handle partial failures with retries and compensations. Maintain transaction logs for recovery. Avoid distributed transactions where possible by designing around them.

**TL;DR:** Saga pattern for distributed transactions, compensation actions, idempotency.

**Keyword/Key mappings:** Microservices, Saga Pattern, Compensation, Idempotency, Event-driven

---

*End of Node.js Interview Questions Document*
