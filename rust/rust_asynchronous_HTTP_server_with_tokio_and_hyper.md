# Rust asynchronous HTTP server with tokio and hyper
Article from: https://blog.guillaume-gomez.fr/articles/2017-02-22+Rust+asynchronous+HTTP+server+with+tokio+and+hyper

In the two last weeks, I worked on a project: writing a small REST API server in Rust. The goal was to compare the performance between this one and the one wrote in C. To do so, we decided to use tokio and hyper.

The C server has been written with a different design: threading and epolling.

### Asynchronous and Rust
Recently, hyper switched to tokio in order to perform asynchronous I/O on sockets. It sounded very attractive because it would allow hyper (in theory) to perform many more operations on a same thread on multiple clients at once: when we're waiting data on a client's socket, we can compute something else on another client with the same thread. Sounds promising, right?

### Testing
In order to compare the Rust and the C server, we used a tool which sent 10.000 requests of 40MB each from 32 threads (the machine had 40 hyper-threads). We consider a request done once it has received the server's answer. All PUT requests write on /dev/null to only test request balancing instead of I/O writing to avoid to be disk-bound.

The C server was around 5500 requests/second.

### First Draft
So I made a quick and small first version (the source code of the reader-writer crate and the Cargo.toml are both at the bottom):

```rust
extern crate futures;
extern crate hyper;
extern crate net2;
extern crate num_cpus;
extern crate reader_writer;
extern crate tokio_core;

use futures::{Future, Stream};
use hyper::{Delete, Get, Put, StatusCode};
use hyper::header::ContentLength;
use hyper::server::{Http, Service, Request, Response};
use net2::unix::UnixTcpBuilderExt;
use tokio_core::reactor::Core;
use tokio_core::net::TcpListener;

use std::thread;
use std::path::Path;
use std::net::SocketAddr;
use std::sync::Arc;
use std::io::{Read, Write};

fn splitter(s: &str) -> Vec<&str> {
    s.split('/').filter(|x| !x.is_empty()).collect()
}

#[derive(Clone)]
struct AlphaBravo {
    rw: reader_writer::ReaderWriter,
}

impl AlphaBravo {
    pub fn new<P: AsRef<Path>>(path: &P) -> AlphaBravo {
        AlphaBravo {
            rw: reader_writer::ReaderWriter::new(path).expect("ReaderWriter::new failed"),
        }
    }
}

impl Service for AlphaBravo {
    type Request = Request;
    type Response = Response;
    type Error = hyper::Error;
    type Future = Box<::futures::Future<Item=Self::Response, Error=Self::Error>>;

    // Handle HTTP requests made to the server.
    fn call(&self, req: Request) -> Self::Future {
        let path = req.path().to_owned();
        ::futures::finished(match *req.method() {
            Get => {
                let values = splitter(&path);
                if values.len() != 1 || !self.rw.exists(&values[0]) {
                    Response::new().with_status(StatusCode::NotFound)
                } else if let Some(mut file) = self.rw.get_file(&values[0], false) {
                    let mut out = Vec::new();
                    if file.read_to_end(&mut out).is_ok() {
                        Response::new().with_body(out)
                    } else {
                        Response::new().with_status(StatusCode::InternalServerError)
                    }
                } else {
                    Response::new().with_status(StatusCode::NotFound)
                }
            }
            Put => {
                // If we didn't receive a key in the uri, we can do nothing.
                let values = splitter(&path);
                if values.len() != 1 {
                    Response::new().with_status(StatusCode::NoContent)
                } else if let Some(mut file) = self.rw.get_file(&values[0], true) {
                    match req.headers().get::<ContentLength>() {
                        Some(&ContentLength(len)) => {
                            // If there is no content, there is nothing to do.
                            if len < 1 {
                                Response::new().with_status(StatusCode::NotModified)
                            } else {
                                // The interesting part is here: for each chunk, we write it into
                                // the file.
                                return Box::new(req.body()
                                                   .for_each(move |chunk| {
                                    if file.write(&*chunk).is_ok() {
                                        Ok(())
                                    } else {
                                        Err(hyper::Error::Status)
                                    }
                                }).then(|r| match r {
                                      Ok(_) => Ok(Response::new().with_status(StatusCode::Ok)),
                                      Err(_) => Ok(Response::new().with_status(
                                                    StatusCode::InsufficientStorage)),
                                  }));
                            }
                        }
                        None => Response::new().with_status(StatusCode::NotModified),
                    }
                } else {
                    Response::new().with_status(StatusCode::InternalServerError)
                }
            }
            Delete => {
                let values = splitter(&path);
                if values.len() == 1 && self.rw.exists(&values[0]) {
                    match self.rw.remove(&values[0]) {
                        Ok(_) => Response::new().with_status(StatusCode::Ok),
                        Err(_) => Response::new().with_status(StatusCode::InternalServerError),
                    }
                } else {
                    Response::new().with_status(StatusCode::NotFound)
                }
            }
            _ => {
                Response::new().with_status(StatusCode::NotFound)
            }
        }).boxed()
    }
}

// Nothing fancy in here, we start an instance of our server with one reactor.
fn start_server<P: AsRef<Path>>(addr: &str, path: P) {
    let addr = addr.parse().unwrap();
    let (listening, server) = Server::standalone(|tokio| {
        let values = Rc::new(RefCell::new((HashMap::new())));
        Server::http(&addr, tokio)?.handle(move || Ok(AlphaBravo::new(&path)), tokio)
    }).unwrap();
    println!("Listening on http://{}", listening);
    server.run();
}

fn main() {
    start_server("127.0.0.1:8080", "/tmp/data-rs/");
}
```

So this first version was nice. It worked fine, but it was clearly not fast enough (around 500 requests/second).

### Second draft: more concurrency
@seanmonstar then showed me some code to be able to run multiple reactors concurrently. The start_server function got replaced by the following code:

```rust
extern crate num_cpus;

// Start a new `Core` by reusing the server socket.
fn serve<P: AsRef<Path>>(addr: &SocketAddr, protocol: &Http, path: P) {
    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let listener = net2::TcpBuilder::new_v4().unwrap()
                        .reuse_port(true).unwrap()
                        .bind(addr).unwrap()
                        .listen(128).unwrap();
    let listener = TcpListener::from_listener(listener, addr, &handle).unwrap();
    core.run(listener.incoming().for_each(|(socket, addr)| {
        protocol.bind_connection(&handle, socket, addr, AlphaBravo::new(&path));
        Ok(())
    })).unwrap();
}

fn start_server<P: AsRef<Path> + Send + Sync>(nb_instances: usize, addr: &str, path: P) {
    let addr = addr.parse().unwrap();

    let protocol = Arc::new(Http::new());
    {
        let rpath = path.as_ref();

        for _ in 0..nb_instances - 1 {
            let protocol = protocol.clone();
            let ppath = rpath.to_path_buf();
            thread::spawn(move || serve(&addr, &protocol, ppath));
        }
    }
    serve(&addr, &protocol, path);
}

fn main() {
    start_server(num_cpus::get(), "127.0.0.1:8080", "/tmp/data-rs/");
}
```

Not many changes, right? The performances went up to 900 requests/second! However, this code has a major downside: the client dispatching is stupid and problematic. When running my server on 32 threads, it uses 100% of them for 70% of the total test time, then 4 or 5 threads only continued to run while all the others did nothing. Quite a huge loss!

### Third draft: rewrite dispatcher
This is where I had to go deeper in hyper and tokio. The dispatcher was stupid, it was first come, first served; which led to the previously evoked problem.

In order to solve this issue, I decided to rewrite the dispatcher in order to make the distribution of the clients more stable. To put it simply:

* first client goes to first thread
* second client goes to second thread
* ...
* (if you have 32 thread) thirty-third client goes to first thread
thirty-fourth client goes to second thread
* ...
* I think you got the main idea. Now let's take a look at the code:

```rust
// This struct will be used to handle clients queue.
struct ThreadData {
    // Each time a new connected client is sent to this thread, because sending it to the reactor,
    // we need to put it here.
    entries: Mutex<Vec<(TcpStream, SocketAddr)>>,
    blocker: Condvar,
}

impl ThreadData {
    pub fn new() -> Arc<ThreadData> {
        Arc::new(ThreadData {
            entries: Mutex::new(Vec::new()),
            blocker: Condvar::new(),
        })
    }
}

struct Foo<F: Fn()> {
    c: F,
}

// This is where the magic occurs, our object needs to be polled anytime it receives a new client!
impl<F: Fn()> Future for Foo<F> {
    type Item = ();
    type Error = ();

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        (self.c)();
        // If we returned `Async::Ready`, this function will never be called again so let's
        // avoid it.
        Ok(Async::NotReady)
    }
}

fn make_reader_threads<P: AsRef<Path> + Send + Sync>(path: P,
                                                     total: usize) -> Vec<Arc<ThreadData>> {
    let mut thread_data = Vec::new();
    for _ in 0..total {
        let data = ThreadData::new();
        thread_data.push(data.clone());
        let rpath = path.as_ref();
        let ppath = rpath.to_path_buf();
        thread::spawn(move || {
            let mut core = Core::new().unwrap();
            let handle = core.handle();
            let protocol = Http::new();
            core.run(Foo { c: || {
                let client = {
                    let mut entries = data.entries.lock().unwrap();
                    if entries.is_empty() {
                        None
                    } else {
                        Some(entries.remove(0))
                    }
                };
                if let Some(client) = client {
                    protocol.bind_connection(&handle, client.0, client.1, AlphaBravo::new(&ppath));
                }
                let data = data.clone();
                let task = task::park();
                // The trick is here: we need to call `unpark` on the `Task` in order to make the
                // `Task` calls our `Foo::poll` method.
                thread::spawn(move || {
                    let mut entries = data.entries.lock().unwrap();
                    // We block while we don't have a client.
                    while entries.is_empty() {
                        entries = data.blocker.wait(entries).unwrap();
                    }
                    task.unpark();
                });
            }}).unwrap();
        });
    }
    thread_data
}

fn serve<P: AsRef<Path> + Send + Sync>(addr: &SocketAddr, path: P, total: usize) {
    let thread_data = make_reader_threads(path, total);

    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let listener = net2::TcpBuilder::new_v4().unwrap()
                        .bind(addr).unwrap()
                        .listen(1000).unwrap();
    let listener = TcpListener::from_listener(listener, addr, &handle).unwrap();
    let mut counter = 0;
    // We're now back on reading the server socket with only one thread to then dispatch
    // clients evenly.
    core.run(listener.incoming().for_each(move |(socket, addr)| {
        // Taking corresponding thread data.
        let ref entry = thread_data[counter];
        {
            // Pushing the new client into it.
            entry.entries.lock().unwrap().push((socket, addr));
            // Notify the condvar.
            entry.blocker.notify_one();
        }
        // Incrementing the counter to select next thread.
        counter += 1;
        if counter >= total {
            counter = 0;
        }
        Ok(())
    })).unwrap();
}

fn start_server<P: AsRef<Path> + Send + Sync>(nb_instances: usize, addr: &str, path: P) {
    println!("===> Starting server on {} threads", nb_instances);
    let addr = addr.parse().unwrap();

    println!("=> Server listening on \"{}\"", addr);
    println!("=> Saving files in \"{}\"", path.as_ref().display());
    serve(&addr, path, nb_instances);
}
```

With this code, we went up to 3000 requests/second. Quite the improvement! But still under the C
server performances... It's getting harder to make it faster.

### Fourth draft: optimizations
The mutex handling was a bit heavy, so I tried to make it as small as possible. In short, here is the new code:

```rust 
// We removed the condvar and replaced it with the `Task` to send the `unpark` not from the thread
// but directly from the main thread.
struct ThreadData {
    entries: Vec<(TcpStream, SocketAddr)>,
    task: Option<task::Task>,
}

impl ThreadData {
    pub fn new() -> Arc<Mutex<ThreadData>> {
        Arc::new(Mutex::new(ThreadData {
            entries: Vec::new(),
            task: None,
        }))
    }
}

fn make_reader_threads<P: AsRef<Path> + Send + Sync>(path: P,
                                                     total: usize) -> Vec<Arc<Mutex<ThreadData>>> {
    let mut thread_data = Vec::new();
    for _ in 0..total {
        let data = ThreadData::new();
        thread_data.push(data.clone());
        let rpath = path.as_ref();
        let ppath = rpath.to_path_buf();
        thread::spawn(move || {
            let mut core = Core::new().unwrap();
            let handle = core.handle();
            let protocol = Http::new();
            core.run(Foo { c: || {
                let mut data = data.lock().unwrap();
                for (socket, addr) in data.entries.drain(..) {
                    protocol.bind_connection(&handle, socket, addr, AlphaBravo::new(&ppath));
                }
                // We reset the task in our `ThreadData` in case we switched context.
                data.task = Some(task::park());
            }}).unwrap();
        });
    }
    thread_data
}

fn serve<P: AsRef<Path> + Send + Sync>(addr: &SocketAddr, path: P, total: usize) {
    let thread_data = make_reader_threads(path, total);

    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let listener = net2::TcpBuilder::new_v4().unwrap()
                        .bind(addr).unwrap()
                        .listen(1000).unwrap();
    let listener = TcpListener::from_listener(listener, addr, &handle).unwrap();
    let mut counter = 0;
    core.run(listener.incoming().for_each(move |(socket, addr)| {
        let ref entry = thread_data[counter];
        {
            let mut entry = entry.lock().unwrap();
            entry.entries.push((socket, addr));
            // This is now where we call the `unpark` method.
            if let Some(task) = entry.task.take() {
                task.unpark();
            }
        }
        counter += 1;
        if counter >= total {
            counter = 0;
        }
        Ok(())
    })).unwrap();
}
```

Surprisingly, the performances went up again to be around 3600 requests/second. This is still under the 5500 requests/second from the C server...

### Conclusion
So what should I conclude? Did I use the tokio crate incorrectly? Or maybe handling requests asynchronously wasn't the good way to go in order to compete with the C server? It's still hard to say but I have to admit that I was greatly disappointed. Maybe I was expecting too much from tokio...

Anyway, I think I'll try to go back to this code some time later when both tokio and hyper will have evolved a bit more.

Again, thank you @seanmonstar for all your help!

### Extern files
Here is the Cargo.toml file for the HTTP server:
```toml
[package]
name = "data"
version = "0.0.1"
authors = ["Guillaume Gomez <guillaume1.gomez@gmail.com>"]

[dependencies]
crc = "1.4.0"
futures = "0.1.3"
hyper = { git = "https://github.com/hyperium/hyper.git", branch = "master" }
libc = "0.2.20"
num_cpus = "1.2.1"
net2 = "0.2.26"
reader-writer = { path = "../reader-writer" }
tokio-core = "0.1.4"

[dev-dependencies]
reqwest = "0.2.0"
tempdir = "0.3.5"

[[bin]]
name = "data"
path = "src/main.rs"
```
Here is the Cargo.toml file for the reader-writer crate:
```toml
[package]
name = "reader-writer"
version = "0.0.1"
authors = ["Guillaume Gomez <guillaume1.gomez@gmail.com>"]

[dependencies]
futures = "0.1"
libc = "^0.2"
tokio-core = { git = "https://github.com/tokio-rs/tokio-core", branch = "master" }

[dev-dependencies]
tempdir = "0.3.5"

[lib]
name = "reader_writer"
```
And here is the reader-writer source code:
```rust
/// This is mostly a wrapper around the `fs::File` struct to perform conversions
/// to tokio types.
#[derive(Clone)]
pub struct ReaderWriter {
    folder: PathBuf,
}

impl ReaderWriter {
    /// Open a folder or create it if it doesn't exist. It doesn't truncate.
    pub fn new<P: AsRef<Path>>(path: &P) -> io::Result<ReaderWriter> {
        if path.as_ref().is_dir() || create_dir_all(path).is_ok() {
            Ok(ReaderWriter {
                folder: path.as_ref().to_path_buf(),
            })
        } else {
            Err(io::Error::new(io::ErrorKind::Other, "Path isn't a directory"))
        }
    }

    /// Returns a `File` with read/write access and creates it if it doesn't exist.
    pub fn get_file<P: AsRef<Path>>(&self, file_id: &P, truncate: bool) -> Option<File> {
        OpenOptions::new()
                    .write(true)
                    .create(true)
                    .read(true)
                    .truncate(truncate)
                    .open(self.folder.join(file_id))
                    .ok()
    }

    /// Remove the corresponding file.
    pub fn remove<P: AsRef<Path>>(&self, file_id: &P) -> io::Result<()> {
        remove_file(self.folder.join(file_id))
    }

    /// Checks if the corresponding file exists
    pub fn exists<P: AsRef<Path>>(&self, file_id: &P) -> bool {
        let path = self.folder.join(file_id);
        path.exists() && path.is_file()
    }
}
```
