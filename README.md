# lifeline-rs
Lifeline is a dependency injection library for message-based applications.  Lifeline produces applications which are:
 - **Clean:** Bus implementations provide a high-level overview of the application, and services clearly define the messages they send and receive.
 - **Decoupled:**  Services and tasks have no dependency on their peers, as they only depend on the message types they send and receive.
 - **Stoppable:** Services and tasks are trivially cancellable.  For example, you can terminate all tasks associated with a connection when a client disconnects.
 - **Greppable:**  The impact/reach of a message can be easily understood by searching for the type in the source code.
 - **Testable:**  Lifeline applications communicate via messages, which makes unit testing easy.  Create the bus, spawn the service, send a message, and expect an output message.

In order to achieve these goals, lifeline provides patterns, traits, and implementations:
 - The **Bus**, which constructs & distributes channel Senders/Receivers, and Resources.
 - The **Carrier**, which translates messages between two Bus instances.  Carriers are critical when building large applications, and help minimize the complexity of the messages on each bus.
 - The **Service**, which takes channels from the bus, and spawns tasks which send and receive messages.
 - The **Task**, an async future which returns a lifeline when spawned.  When the lifeline is dropped, the future is immedately cancelled.
 - The **Resource**, a struct which can be stored in the bus, and taken (or cloned) when services spawn.

For a quick introduction, see the [hello.rs example.](https://github.com/austinjones/lifeline-rs/blob/master/examples/hello.rs)
For a full-scale application see [tab-rs.](https://github.com/austinjones/tab-rs)

## Quickstart
Lifeline can be used with the [tokio](https://docs.rs/tokio/) and [async-std](https://docs.rs/async-std/) runtimes.  By default, lifeline uses `tokio`.
```toml
lifeline = "0.6"
```

[async-std](https://docs.rs/async-std/) can be enabled with the `async-std-executor` feature.  And the `mpsc` implementation can be enabled with the `async-std-channels` feature:
```toml
lifeline = { version = "0.6", default-features = false, features = ["dyn-bus", "async-std-executor", "async-std-channels"] }
```

Lifeline also supports [postage channels](https://docs.rs/postage/), a library that provides a portable set of channel implementations (compatible with any executor).
Postage also provides Stream and Sink combinators (similar to futures StreamExt), that are optimized for async channels.  
Postage is intended to replace the LifelineSender/LifelineReceiver wrappers that were removed in lifeline v0.6.0.

## Upgrading
v0.6.0 contains several breaking changes:
- The LifelineSender and LifelineReceiver wrappers were removed.  This was necessary due to the recent changes in the Stream ecosystem, and the upcoming stabilization of the Stream RFC.
If you need Stream/Sink combinators, take a look at [postage](https://crates.io/crates/postage), or [tokio-stream](https://crates.io/crates/tokio-stream).
- The barrier channel was removed.  It can be replaced with [postage::barrier](https://docs.rs/postage/0.3.1/postage/barrier/index.html).
- The subscription channel was removed.  If you need it back, you can find the code before the removal [here](https://github.com/austinjones/lifeline-rs/blob/b15ab2342abcfa9c553d403cb58d2403531bf89c/src/channel/subscription.rs).
- The Sender and Receiver traits were removed from prelude.   This is so that importing the lifeline prelude does not conflict with Sink/Stream traits.  You can import them with:
`use lifeline::{Sender, Receiver}`.

## The Bus
The bus carries channels and resources.  When services spawn, they receive a reference to the bus.

Channels can be taken from the bus.  If the channel endpoint is clonable, it will remain available for other services.  If the channel is not clonable, future calls will receive an `Err` value.  The Rx/Tx type parameters are type-safe, and will produce a compile error
if you attempt to take a channel for an message type which the bus does not carry.

### Bus Example
```rust
use lifeline::lifeline_bus;
use lifeline::Message;
use lifeline::Bus;
use myapp::message::MainSend;
use myapp::message::MainRecv;
use tokio::sync::mpsc;

lifeline_bus!(pub struct MainBus);

impl Message<MainBus> for MainSend {
    type Channel = mpsc::Sender<Self>;
}

impl Message<MainBus> for MainRecv {
    type Channel = mpsc::Sender<Self>;
}

fn use_bus() -> anyhow::Result<()> {
    let bus = MainBus::default();

    let tx_main = bus.tx::<MainRecv>()?;
    tx_main.send(MainRecv {});

    Ok(())
}
```

## The Carrier
Carriers provide a way to move messages between busses.  Carriers can translate, ignore, or collect information,
providing each bus with the messages that it needs.

Large applications have a tree of Busses.  This is good, it breaks your app into small chunks.
```
- MainBus
  | ConnectionListenerBus
  |  | ConnectionBus
  | DomainSpecificBus
  |  | ...
```

Carriers allow each bus to define messages that minimally represent the information it's services need to function, and prevent an explosion of messages which are copied to all busses.

Carriers centralize the communication between busses, making large applications easier to reason about.

### Carrier Example
Busses deeper in the tree should implement `FromCarrier` for their parents - see the [carrier.rs example](https://github.com/austinjones/lifeline-rs/blob/master/examples/carrier.rs) for more details.

```rust
let main_bus = MainBus::default();
let connection_listener_bus = ConnectionListenerBus::default();
let _carrier = connection_listener_bus.carry_from(&main_bus)?;
// you can also use the IntoCarrier trait, which has a blanket implementation
let _carrier = main_bus.carry_into(&main_bus)?;
```

## The Service
The Service synchronously takes channels from the Bus, and spawns a tree of async tasks (which send & receive messages).  When spawned, the service returns one or more Lifeline values.  When a Lifeline is dropped, the associated task is immediately cancelled.

It's common for `Service::spawn` to return a Result.  Taking channel endpoints is a fallible operation.  This is because, depending on the channel type, the endpoint may not be clonable.  Lifeline clones endpoints when it can (`mpsc::Sender`, `broadcast::*`, and `watch::Receiver`).  Lifeline tries to make this happen as early as possible.

```rust
use lifeline::{Service, Task};
pub struct ExampleService {
    _greet: Lifeline,
}

impl Service for ExampleService {
    type Bus = ExampleBus;
    type Lifeline = Lifeline;

    fn spawn(bus: &Self::Bus) -> Self::Lifeline {
        let mut rx = bus.rx::<ExampleRecv>()?;
        let mut tx = bus.tx::<ExampleSend>()?;
        let config = bus.resource::<ExampleConfig>()?;

        Self::task("greet", async move {
            // drive the channels!
        })
    }
}
```

## The Task
The Task executes a `Future`, and returns a `Lifeline` value when spawned.  When the lifeline is dropped, the future is immediately cancelled.

`Task` is a trait that is implemented for all types - you can import it and use `Self::task` in any type.  In lifeline, it's 
most commonly used in Service implementations.

Tasks can be infallible:
```rust
Self::task("greet", async move {
    // do something
})
```

Or, if you have a fallible task, you can return `anyhow::Result<T>`.  Anyhow is required to solve type inference issues.

```rust
Self::try_task("greet", async move {
    // do something
    let something = do_something()?;
    Ok(())
})
```

# Testing
One of the goals of Lifeline is to provide interfaces that are very easy to test.  Lifeline runtimes are easy to construct in tests:

```rust
#[tokio::test]
async fn test() -> anyhow::Result<()> {
    // this is zero-cost.  channel construction is lazy.
    let bus = MainBus::default();
    let service = MainService::spawn(&bus)?;

    // the service took `bus.rx::<MainRecv>()`
    //                + `bus.tx::<MainSend>()`
    // let's communicate using channels we take.
    let tx = bus.tx::<MainRecv>()?;
    let rx = bus.rx::<MainSend>()?;

    // drop the bus, so that any 'channel closed' errors will occur during our test.
    // this would likely happen in practice during the long lifetime of the program
    drop(bus);

    tx.send(MainRecv::MyMessage)?;

    // wait up to 200ms for the message to arrive
    // if we remove the 200 at the end, the default is 50ms
    lifeline::assert_completes!(async move {
        let response = rx.recv().await;
        assert_eq!(MainSend::MyResponse, response);
    }, 200);

    Ok(())
}
```

# The Details
### Logging
Tasks (via [log](https://docs.rs/log/0.4.11/log/)) provide debug logs when the are started, ended, or cancelled.

If the task returns a value, it is also printed to the debug log using `Debug`.
```
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] START ExampleService/ok_task
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] END ExampleService/ok_task

2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] START ExampleService/valued_task
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] END ExampleService/valued_task: MyStruct {}
```

If the task is cancelled (because it's lifeline is dropped), that is also printed.
```
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] START ExampleService/cancelled_task
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] CANCEL ExampleService/cancelled_task
```

If the task is started using `Task::try_task`, the `Ok`/`Err` value will be printed with `Display`.

```
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] START ExampleService/ok_task
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] OK ExampleService/ok_task
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] END ExampleService/ok_task

2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] START ExampleService/err_task
2020-08-23 16:45:10,422 ERROR [lifeline::service] ERR: ExampleService/err_task: my error
2020-08-23 16:45:10,422 DEBUG [lifeline::spawn] END ExampleService/err_task
```

### A note about autocomplete
`rust-analyzer` does not currently support auto-import for structs defined in macros.  Lifeline really needs the
struct defined in the macro, as it injects magic fields which store the channels at runtime.

There is a workaround: define a `prelude.rs` file in your crate root that exports `pub use` for all your bus implementations.  
```
pub use lifeline::*;
pub use crate::bus::MainBus;
pub use crate::other::OtherBus;
...
```
Then in all your modules:
`use crate::prelude::*`

## The Resource
Resources can be stored on the bus.  This is very useful for configuration (e.g `MainConfig`), or connections (e.g. a `TcpStream`).

Resources implement the `Storage` trait, which is easy with the `impl_storage_clone!` and `impl_storage_take!` macros.

```rust
use lifeline::{lifeline_bus, impl_storage_clone};
lifeline_bus!(MainBus);
pub struct MainConfig {
    pub port: u16
}

impl_storage_clone!(MainConfig);

fn main() {
    let bus = MainBus::default()
    bus.store_resource::<MainConfig>(MainConfig { port: 12345 });
    // from here
}
```

Lifeline does not provide `Resource` implementations for Channel endpoints - use `bus.rx()` and `bus.tx()`.

## The Channel
Channel senders must implement the `Channel` trait to be usable in an `impl Message` binding.

In most cases, the `Channel` endpoints just implement `Storage`, which determines whether to 'take or clone' the endpoint on a `bus.rx()` or `bus.tx()` call.

Here is an example implem
```rust
use lifeline::Channel;
use crate::{impl_channel_clone, impl_channel_take};
use tokio::sync::{broadcast, mpsc, oneshot, watch};

impl<T: Send + 'static> Channel for mpsc::Sender<T> {
    type Tx = Self;
    type Rx = mpsc::Receiver<T>;

    fn channel(capacity: usize) -> (Self::Tx, Self::Rx) {
        mpsc::channel(capacity)
    }

    fn default_capacity() -> usize {
        16
    }
}

impl_channel_clone!(mpsc::Sender<T>);
impl_channel_take!(mpsc::Receiver<T>);
```

Broadcast senders should implement the trait with the `clone_rx` method overriden, to take from `Rx`, then subscribe to `Tx`.
