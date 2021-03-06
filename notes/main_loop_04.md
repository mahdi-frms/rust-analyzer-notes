# Summary of ERA04: main loop

> You can watch actual video here:

[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/bfE_ZVDLHmo/1.jpg)](https://www.youtube.com/watch?v=bfE_ZVDLHmo)

## RA entry point : main()
The main function does a bunch of stuff:
* enable debugging the binary
* check for command line flags and sub commands
* check for the configuration and cargo meta data
* finally, run the main loop

> NOTE: Throughout this lecture, topics such as handling Cargo checks,metadata and loading Cargo configs are not explained, due to their complicated logic and implementation.

## GlobalState
The `GlobalState` is the state of Rust Analyzer with is composed of many types, the most important being: 
* `Vfs` the virtual filesystem
* `AnalysisHost` the analysis data
* `TaskPool` the thread pool for concurrency
* `ReqQueue` maintains the incomming and outgoing LSP messages

* `memdoc` property, which keeps content of the files whose changes are not saved to the disk, and are handled by the editor
* `loader` property, which is a handle to the [vfs loader](vfs_02.md)
The `run` method of this struct will run the main loop.

> The name GlobalState was initally used in [Sorbet](https://github.com/sorbet/sorbet) project

## Main Loop
The main loop is an event loop that keeps waiting for `Event`s, and once it recieves one, handles it and returns to the waiting state. Note that handling each event is supposed to take less than 100 miliseconds (which is sometimes violated). The main loop is basically the following block of code:
``` rust
// crates/rust-analyzer/src/main_loop.rs
while let Some(event) = self.next_event(&inbox) {
    if let Event::Lsp(lsp_server::Message::Notification(not)) = &event {
        if not.method == lsp_types::notification::Exit::METHOD {
            return Ok(());
        }
    }
    self.handle_event(event)?
}
```
The `next_event` method will block on 4 crossbeam channels to get an event instance. After that, it will be handled by `handle_event` function.

## Events
There are four types of events in RA:
``` rust
enum Event {
    Lsp(lsp_server::Message),
    Task(Task),
    Vfs(vfs::loader::Message),
    Flycheck(flycheck::Message),
}
```
### Vfs
As the loader keeps watching for file changes and loads the files, these changes mut be applied to the vfs instance of the global state. An important point is that the main loop applies the vfs changes **in bulk**, meaning that after handling a vfs event, if there are already extra vfs events, they will immediately be handled before **processing changes**.

### LSP
An LSP Message can either be a `Request`, `Response`, or a `Notification`. Both the server and LSP client can send request and recieve responses. Each request is given an ID and is supposed to be handled by a response with the same ID. A notification is another type of request which does not need a response. They're covered in the next sections.

## Handling LSP Messages
Each request is either handled by `on` or `on_sync` methods (the only difference being that `on` spawns a thread for the job). after the handling is done, the response will be sent to the channel as a `Response` instance. This is how the main loop actually provides the responses to request. the response messages in the event loop are in fact generated by request handling methods themselves.The main loop In case of recieving a response, knows that it's the result of a processed request and puts them in the `ReqQueue`.
Notifications are about changes done to a file, which are not stored on the disk yet.

## Processing changes
After each event is handled, the changes will be processes (except vfs events which are handled in bulk). the changes done to the vfs will finally be applied to the `AnalysisHost` instance of `GlobalState`.