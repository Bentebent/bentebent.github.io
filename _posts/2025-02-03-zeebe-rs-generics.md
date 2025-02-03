---
title: Generics, builders, pain
date: 2025-02-03 01:00:00 +0100
categories: [Programming, Rust]
tags: [programming, rust]
description: Send + Sync + 'crying
---
## A quick look at zeebe-rs

I recently released [zeebe-rs](https://github.com/Bentebent/zeebe-rs), an open source [Camunda Zeebe](https://camunda.com/platform/zeebe/) client for Rust. The need for an up-to-date Zeebe client arose at work and I felt this was a good opportunity to explore something new. Both gRPC (which I haven't used before), a bit more of async Rust and bringing something potentially useful to the open source community. Fair warning, this blog entry contains _a ton_ of code.

The design and implementation of this library is really simple. It is in essence just a set of [typestate](https://cliffle.com/blog/rust-typestate/) builders for gRPC requests to make it as easy as possible to use without messing up any inputs to Zeebe. I decided to not worry too much about performance (hello .Clone() my dear friend) because any Zeebe integrations are more likely to be I/O-bound anyway. This is really straight forward for just building a simple gRPC request as you can see here:

```rust
pub struct Initial;
pub struct WithName;
pub struct WithKey;

pub trait PublishMessageRequestState {}
impl PublishMessageRequestState for Initial {}
impl PublishMessageRequestState for WithName {}
impl PublishMessageRequestState for WithKey {}

#[derive(Debug, Clone)]
pub struct PublishMessageRequest<T: PublishMessageRequestState> {
    client: Client,
    name: String,
    correlation_key: String,
    time_to_live: i64,
    message_id: String,
    variables: serde_json::Value,
    _state: std::marker::PhantomData<T>,
}

impl<T: PublishMessageRequestState> PublishMessageRequest<T> {
    pub(crate) fn new(client: Client) -> PublishMessageRequest<Initial> {
        PublishMessageRequest {
          //Default initialization
        }
    }

    fn transition<NewState: PublishMessageRequestState>(self) -> PublishMessageRequest<NewState> {
        PublishMessageRequest {
          //Move all fields
        }
    }
}

impl PublishMessageRequest<Initial> {
    pub fn with_name(mut self, name: String) -> PublishMessageRequest<WithName> {
        self.name = name;
        self.transition()
    }
}

impl PublishMessageRequest<WithName> {
    /// Sets the correlation key of the message
    /// The correlation key is used to determine which process instance
    /// the message is published to.
    pub fn with_correlation_key(
        mut self,
        correlation_key: String,
    ) -> PublishMessageRequest<WithKey> {
        self.correlation_key = correlation_key;
        self.transition()
    }

    /// Publish a message without a correlation key
    /// Use for message start events
    pub fn without_correlation_key(self) -> PublishMessageRequest<WithKey> {
        self.transition()
    }
}

impl PublishMessageRequest<WithKey> {
    pub fn with_time_to_live(mut self, ttl: Duration) -> Self {
        self.time_to_live = ttl.as_millis() as i64;
        self
    }

    pub fn with_message_id(mut self, message_id: String) -> Self {
        self.message_id = message_id;
        self
    }

    pub fn with_variables<T: Serialize>(mut self, data: T) -> Result<Self, ClientError> {
        self.variables = serde_json::to_value(data)
            .map_err(|e| ClientError::SerializationFailed { source: e })?;
        Ok(self)
    }

    pub async fn send(mut self) -> Result<PublishMessageResponse, ClientError> {
        let res = self
            .client
            .gateway_client
            .publish_message(proto::PublishMessageRequest {
                name: self.name,
                correlation_key: self.correlation_key,
                time_to_live: self.time_to_live,
                message_id: self.message_id,
                variables: self.variables.to_string(),
            })
            .await?;

        Ok(res.into_inner().into())
    }
}
```

And this is how you would use it.

```rust
PublishMessageRequest::new()
  .with_name(String::from("hello_world"))
  .without_correlation_key()
  .with_variables(HelloWorld {
    hello: String::from("world"),
  })?
  .send()
  .await?;
```

Because this is encoded into the type system the user can't mess it up without getting a compile error. It is a bit verbose but the trade-off is worth it for constructs such as these in my opinion. Especially once you introduce any kind of branching as we will get to later.

## Workers
The more interesting part of Zeebe is the concept of registering [job workers](https://docs.camunda.io/docs/components/concepts/job-workers/). Job workers are long running services that continously fetch jobs from Zeebe for execution then reports back the results. This is represented as a [worker](https://docs.rs/zeebe-rs/latest/zeebe_rs/struct.Worker.html) that has two parts; a producer that continously polls jobs from Zeebe and a consumer that runs a callback and reports back results. The interface to create a worker is also a builder that diverges when you register the job handler callback. The most basic case looks as follows:

```rust
let worker = client
    .worker()
    .with_job_timeout(Duration::from_secs(60))
    .with_request_timeout(Duration::from_secs(10))
    .with_max_jobs_to_activate(5)
    .with_concurrency_limit(3)
    .with_job_type("example-service")
    .with_handler(|client, job| async move {
        // Process job and return result
        client.complete_job().with_job_key(job.key()).send().await;
    })
    .build();

// Start the worker
worker.run().await?;
```

But workers can also share a `State<T>`, which looks like this:

```rust
let stock = Arc::new(SharedState(Mutex::new(0)));

//Same as above
    .with_state(stock)
    .with_handler(|client, job, state| async move {
        // Process job here using shared state somewhow
        client.complete_job().with_job_key(job.key()).send().await;
    })
//...
```

I had some design goals for this interface and it got surprisingly hard I'll be honest. These were the goals:
1. Users should be able to just pass in any closure or free function that matches either of the available signatures.
2. Users should not have to re-write or register the logic to report results to Zeebe over and over.
3. Handlers only need to be async.
4. If possible use static dispatch everywhere.

### Implementation - with a caveat
As I was typing this blog entry out I wanted to recreate a minimal sample of what the Worker builder looks like to illustrate the problems I encountered and why I ended up _not_ solving it using static dispatch. Well after some struggles to recreate the problem I instead ended up with a better implementation that solved all my problems so now I have to go back and refactor `zeebe-rs` instead, congratulations to me! But let's try and look at it and see what conclusions can be drawn.

#### Handling callback outputs
This part was unchanged between my two implementations and is just such an enjoyable aspect of Rust's trait system. The idea is that this would fulfill design goal #2, enable re-using logic for reporting results back to Zeebe. I simply expose a trait with a single async function that is responsible for handling the value.

```rust
pub trait OutputHandler {
    fn handle(self) -> impl Future<Output = ()> + Send + 'static;
}
```

Users of the library can now just implement this trait for whatever type they see fit. I ship it with two implementations, one of which is for the unit value `()`:

```rust
impl OutputHandler for () {
    fn handle(self) -> impl Future<Output = ()> + Send + 'static {
        std::future::ready(())
    }
}
```
As long as `OutputHandler` is implemented for any values returned by a worker callback we're good, It Just Works™. With Rust's generics and trait bounds we even get a great error message if the user forgot to implement `OutputHandler` for their own types.

```
error[E0277]: the trait bound `(): OutputHandler` is not satisfied
   --> examples\worker_test.rs:230:20
    |
230 |     let worker_a = WorkerBuilder::new()
    |                    ^^^^^^^^^^^^^ the trait `OutputHandler` is not implemented for `()`, which is required by `{closure@examples\worker_test.rs:232:23: 232:28}: Handler<_>`
    |
help: this trait has no implementations, consider adding one
   --> examples\worker_test.rs:3:1
    |
3   | pub trait OutputHandler {
    | ^^^^^^^^^^^^^^^^^^^^^^^
```
This really is one of my favourite things about Rust (until I inevitably bash my head against the [orphan rule](https://ianbull.com/notes/rusts-orphan-rule/), but that's a story for another day) and it just highlights how flexible of a system it can potentially be. Together with the clear compiler error its a 10/10 experience, no notes so far.

_Nevermind._
```rust
impl<T> OutputHandler for T
where
    T: Serialize,
{
    fn handle(self) -> impl Future<Output = ()> + Send + 'static {
        async move {
            //Serialize self and send to Zeebe
        }
    }
}
```

```
error[E0119]: conflicting implementations of trait `OutputHandler` for type `()`
  --> examples\worker_test.rs:15:1
   |
9  |   impl OutputHandler for () {
   |   ------------------------- first implementation here
...
15 | / impl<T> OutputHandler for T
16 | | where
17 | |     T: Serialize,
   | |_________________^ conflicting implementation for `()`
```

Maybe I have some notes. This is where the proverbial hammer [SFINAE](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error), competitor for the worst acronym ever next to [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization), rears its face. If I was writing C++ it would be a fairly natural pattern to just create a set of overload resolutions where anything that can be serialized is handled by the code above but `()`, which is technically serializable, just NOPs. My instincts said that monomorphization would solve the problem, maybe I even get it inlined by the compiler? Well I was wrong but at least the compiler told me in clear text. 

I have a great love for C++ templates, especially after the introduction of [concepts](https://en.cppreference.com/w/cpp/language/constraints) in C++20. I feel like it really became possible to write templated/metaprogramming code where your successors would not use a photo of your face as a dartboard with this addition. The [specialization RFC](https://rust-lang.github.io/rfcs/1210-impl-specialization.html) has been around since 2015 but progress is looking [pretty dead](https://github.com/rust-lang/rust/issues/31844) right now so I wouldn't count on it for the near future. This is a good motivator for me to try and look into the differences between templates (specifically C++) and Rust generics so I hope to spend more time on that in my next project!

#### The callback trait
Nevermind SFINAE, our new hammer is traits and everything looks like a nail! This is also where I kinda got lost in the intricacies of async in Rust. Let's look at the first implementation of my JobHandler trait.

```rust
type BoxFutureOf<T> = Pin<Box<dyn Future<Output = T> + Send>>;

pub trait JobHandler<Output>: Send + Sync {
    fn execute(&self, client: Client, job: ActivatedJob) -> BoxFutureOf<Output>;
}

impl<F, Fut, Output> JobHandler<Output> for F
where
    F: Fn(Job) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = Output> + Send + 'static,
    Output: Send + 'static,
{
    fn execute(&self, job: Job) -> BoxFutureOf<Output> {
        Box::pin((self)(job))
    }
}
```
![Yikes](assets/img/daffy_yikes.png)
_...what is going on?_

This is a mess. Pin? Box? `Send + Sync + 'static` everywhere? When implementing this I shot down a rabbit hole of googling how to work with Tokio, directed by the hallucinations of Chad Gibbidy. As you can tell the results were kinda rough. There's even a little `dyn` in here so we lose static dispatch. Just looking at it this instinctively feels way too complicated for what is supposed to be very simple functionality. Do I really need dynamic dispatch and manual pinning? The main takeaway here is

1. I need to read the [Rust async book](https://rust-lang.github.io/async-book/04_pinning/01_chapter.html) before I delve deeper into async.
2. There's a great hint by the compiler if you try to create a public trait with an async function.

```
warning: use of `async fn` in public traits is discouraged as auto trait bounds cannot be specified
  --> examples\worker_test.rs:14:5
   |
14 |     async fn foo();
   |     ^^^^^
   |
   = note: you can suppress this lint if you plan to use the trait only in your own code, or do not care about auto traits like `Send` on the `Future`
   = note: `#[warn(async_fn_in_trait)]` on by default
help: you can alternatively desugar to a normal `fn` that returns `impl Future` and add any desired bounds such as `Send`, but these cannot be relaxed without a breaking API change
   |
14 -     async fn foo();
14 +     fn foo() -> impl std::future::Future<Output = ()> + Send;
   |
```
Well **spoiler alert**, we already did this for the `OutputHandler` but when implementing this originally I started with the `JobHandler` and hit a wall. Let's look at the revised version using the hint above:

```rust
trait JobHandler {
    type Output: OutputHandler + Send + Sync + 'static;
    fn handle(&self, job: Job) -> impl Future<Output = ()> + Send;
}

impl<F, Fut> JobHandler for F
where
    F: Fn(Job) -> Fut + Send + Sync + 'static,
    Fut: Future + Send + Sync + 'static,
    Fut::Output: OutputHandler + Send + Sync + 'static,
{
    type Output = Fut::Output;

    async fn handle(&self, job: Job) {
        let res = self(job).await;
        res.handle().await;
    }
}
```
Way cleaner than the previous version. The biggest downstream impact here is the removal of `JobHandler` being generic over `Output` and instead using an associated type. We handle the outputs with `OutputHandler` so we don't actually need `JobHandler` to be generic over outputs and this makes type inference significantly easier once the `JobHandler` is known for the builder. There's another difference here and that's moving the handling of the output into the `JobHandler` implementation. The `Worker` and `WorkerBuilder` now have no generics over the handler output which makes type inference easier at the branch of a worker with or without state.

#### Let's build this
You can see the full original worker implementation [here](https://github.com/Bentebent/zeebe-rs/blob/c536a0e52edb973b6de1cf877b16e993d8f19ddd/src/worker.rs) but the important bits is that I had to split my `WorkerBuilder` into two parts to solve my type inference problems.

```rust
pub struct WorkerBuilder<T, Output> {
    worker_callback: Option<Arc<Box<dyn JobHandler<Output>>>>,
    _state: std::marker::PhantomData<T>,
}

pub struct WorkerStateBuilder<T, Output: Send + 'static> {
    builder: WorkerBuilder<WithJobType, Output>,
    state: Arc<SharedState<T>>,
}
```
We can see how there's no static dispatch since the callback is `dyn JobHandler<Output>`. The `WorkerStateBuilder` comes into action when calling `with_state` where we branch into this "sub-builder".

```rust
impl<Output: Send + 'static> WorkerBuilder<WithJobType, Output> {
    pub fn with_state<T>(self, shared_state: Arc<SharedState<T>>) -> WorkerStateBuilder<T, Output>
    where
        T: Send + Sync + 'static,
    {
        WorkerStateBuilder {
            builder: self,
            state: shared_state,
        }
    }
}
```
I'll admit I think it's the wrong way to do it here (obviously, since I found a better solution) but at the time it solved my type inference problem so I was happy. I also added my shared state implementation after I had already built the first `Worker` implementation and generics really leak _everywhere_. Especially the fact that you have to type out your trait bounds for every `impl` block of a struct or trait is a bad time that motivated me to find a way to break off the state generics to its own struct. You can see a much more useful example of this pattern of you check out the [process instance modification request](https://github.com/Bentebent/zeebe-rs/blob/c536a0e52edb973b6de1cf877b16e993d8f19ddd/src/process_instance/modify.rs) where it is used to build several instances of a complicated object used by the request. The new builder instead looks like this.

```rust
struct WorkerBuilder<S, H = (), T = ()>
where
    S: WorkerBuilderState,
    T: Send + Sync + 'static,
{
    state: Option<Arc<State<T>>>,
    callback: Option<Arc<H>>,
    _marker: std::marker::PhantomData<S>,
}
```
The real kicker here is actually the default types for our generics. This allows type inference to work before we have specified our state and callback making the builder as easy to use as stated in design goal #1. That pesky `dyn` is also gone and we're back to static dispatch! We're getting close to the end here, let's look at some more boilerplate to see the difference in how this is used.

```rust
impl WorkerBuilder<Initial> {
    pub fn with_state<T: Send + Sync + 'static>(
        self,
        state: Arc<State<T>>,
    ) -> WorkerBuilder<WithState, (), T> {
        WorkerBuilder {
            state: Some(state),
            callback: None,
            _marker: std::marker::PhantomData,
        }
    }

    pub fn without_state(self) -> WorkerBuilder<WithoutState> {
        WorkerBuilder {
            state: None,
            callback: None,
            _marker: std::marker::PhantomData,
        }
    }
}
```
We're no longer using the `self.transition()` method but rather explicitly returning a new `WorkerBuilder` where we replace the default generic parameters as we go depending on the need. In the end we have an API that that fulfills all four of our stated goals. No type annotations are needed and we managed to sneak in static dispatch.

```rust
#[tokio::main]
async fn main() {
    let worker_a = WorkerBuilder::new()
        .without_state()
        .with_handler(|job| async move {
            println!("{:?}", job);
            sleep(Duration::from_secs(rand::random::<u64>() % 3 + 1));
            job
        })
        .build();

    let state = Arc::new(State(0));
    let worker_b = WorkerBuilder::new()
        .with_state(state)
        .with_handler(|job, state| async move {
            println!("Worker B: {:?} with state {:?}", job, state);

            sleep(Duration::from_secs(rand::random::<u64>() % 3 + 1));
            job
        })
        .build();

    let worker_c = WorkerBuilder::new()
        .without_state()
        .with_handler(|job| async move {
            println!("worker_c {:?}", job);
            sleep(Duration::from_secs(rand::random::<u64>() % 3 + 1));
            Foo
        })
        .build();

    tokio::join!(worker_a.run(), worker_b.run(), worker_c.run());

    sleep(Duration::from_secs(5));
}
```

## Takeaways
Overall writing this blog entry was almost more of a success than the implementation of `zeebe-rs`. First and foremost let's just mention that refactoring in Rust is incredible. You can bang out a huge refactoring with such speed and confidence compared to any other language I've ever tried and it feels amazing. I was previously unaware of the issues especially related to type inference with generics and my own predisposition to reach for SFINAE at every turn. I do think I should more readily reach for [associated types](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html) rather than going straight for generics as I would in C++. I'm also more and more realizing that I am in _love_ with the typestate pattern for APIs such as these, so much that I started applying it to the Python code I write at work as well. I also realized I have huge gaps in my understanding of async Rust that will need proper study of the underlying core principles. Finally I also need to look into how different generics are from templates as the difference seems bigger than I previously understood.

That's a lot, but I'm looking forward to it. In the meantime, checkout [zeebe-rs](https://github.com/Bentebent/zeebe-rs) and [Camunda](https://camunda.com/) if you're interested in microservice orchestration with Rust. You can also see the entire refactoring of the `Worker` implementation in [this pull request](https://github.com/Bentebent/zeebe-rs/pull/25/files). I appreciate any comments, inputs or suggestions on the topics I discussed in this blog as I am certain I have missed several aspects that could improve my implementations!
