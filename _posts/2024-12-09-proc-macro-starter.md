---
title: A first foray into Rust proc macros
date: 2024-12-09 00:30:00 +0100
categories: [Programming, Rust]
tags: [programming, rust]
description: "#define not_your_grandpas_macros"
---

I've been on and off the Rust learning path for about ~6 months at this point. Most of my learning has been focused on real-time rendering with WASM and [wgpu](https://wgpu.rs/) or building web services using [Rocket](https://rocket.rs/). I've mostly been learning features of Rust based on the needs of my projects and have not really encountered any reason to build my own macros until now. It didn't help that the emphasis when having a cursory look into macros was always _"this is really powerful but also very, very hard"_. But now I had a problem that I felt could be suited so let's get into it.

## Intermission
Handling deprecations between services can be challenging, especially in a distributed environment with remote teams. In my experience, several issues arise when integrations are deprecated or sunset, even within the same company. These challenges are further compounded when dealing with public APIs that have a broad set of consumers. Common issues include:

* A deprecation was communicated in an e-mail, presentation, documentation or Confluence page months ago and then forgotten
* An integration consumer was forgotten and never notified of the deprecation
* An integration has an unknown set of consumers and there are no set communication channels
* An integration consumer was continuously notified through the proper channels but 
  * The information did not filter down to the correct people
  * They didn't prioritize handling the deprecation

Disregarding the last case all of these are problems related to async communication in some way. The integration owner is responsible for informing consumers but they have imperfect information on who those consumers are and unreliable communication channels. Even if you did your best informing of the deprecation you will be blamed when your users suffer catastrophic failures because they didn't migrate in time.

So how do we move as much responsibility as possible (at least information wise) from the owner to the consumers?

Enter the [proposed HTTP deprecation headers](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-deprecation-header). As far as RFCs go this one is very easy to digest and I recommend reading it for the full picture. The summary is a proposed addition of three different headers that communicate API deprecation, see the below table.

| Name        | Value                                                                | Comment                                          |
| ----------- | -------------------------------------------------------------------- | ------------------------------------------------ |
| deprecation | "@1735689599"                                                        | Epoch timestamp when resource will be deprecated |
| link        | "<https://api.example.com/docs>; rel="deprecation"; type="text/html" | Link to documentation providing more information |
| sunset      | "Wed, 31 Dec 2025 23:59:59 GMT"                                      | HTTP date when resource will be sunset           |

Now we're actively submitting the deprecation information with every single request made to our resource. Any client code that is deprecation/sunset aware will be able to pick up this information and continuously inform its owner of the incoming change.

## Ideation

Now we can formulate a problem statement. As a developer I want to
* Add the relevant deprecation headers to an existing HTTP resource without refactoring the current implementation
* Clearly communicate that the resource is to be deprecated and when for anyone reading the code

Python has been my go-to backend service language for a few years now so let's see what this would look like using [FastAPI](https://fastapi.tiangolo.com). This is made simple by using a decorator as in the below example.

```python
def deprecated(deprecation: str, link: str = "", sunset: str = ""):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            response = await func(*args, **kwargs)
            
            if not isinstance(response, Response):
                response = Response(content=response)

            #Header formatting omitted for brevity 
            response.headers["Deprecation"] = deprecation
            if link:
                response.headers["Link"] = f'<{link}>; rel="deprecation"'
            if sunset:
                response.headers["Sunset"] = sunset
            
            return response
        return wrapper
    return decorator

# We have to add the response_class here otherwise FastAPI automatically wraps
# the return as json
@app.get("/", response_class=PlainTextResponse)
@deprecated(
    deprecation = "2024-12-31T23:59:59Z",
    link = "https://example.com/documentation", 
    sunset = "2025-12-31T23:59:59Z"
)
async def index():
    return "Hello, world!"
```
I'd say this fulfills the requirements we set out above. No refactoring was required as we only layered the new headers on top of the existing implementation. Readers of the code only need to look at the signature of the endpoint to see that this is being deprecated and when it is expected to happen. So how do we make this happen in Rust?

Below is a Hello World-endpoint example using Rocket. 

```rust
#[get("/example")]
fn example() -> &'static str {
    "Hello, world!"
}
```
As we can see it's very similar in structure to its FastAPI equivalent. Rocket already uses a [procedural macro attribute](https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros) to turn this into a HTTP endpoint which to me indicated that this might be a reasonable path to go down for solving my problem. Let's do the naive implementation first though, we're just going to wrap the return in our own type. 

Rocket expects all return types from an endpoint to implement the [Responder trait](https://api.rocket.rs/v0.5/rocket/response/trait.Responder) so let's build the struct that we will use for the rest of this exercise. Any struct implementing the Responder trait is composable with other Responders so we can wrap any Responder T in our DeprecatedResponder.

```rust
pub struct DeprecatedResponder<T: for<'r> Responder<'r, 'static>> {
    pub inner: T,
    pub deprecation: Header<'static>,
    pub link: Option<Header<'static>>,
    pub sunset: Option<Header<'static>>,
}

impl<'r, 'o: 'r, T> Responder<'r, 'o> for DeprecatedResponder<T>
where
    T: for<'any> Responder<'any, 'static>,
{
    fn respond_to(self, req: &'r Request<'_>) -> response::Result<'o> {
      //Here we just build the inner response and overlay our deprecation headers
    }
}
```

Here we apply it to our example function.

```rust
#[get("/example")]
fn example() -> DeprecatedResponder<&'static str> {
    DeprecatedResponder::new(
        "Hello, world!",
        "2024-12-31T23:59:59Z",
        "https://example.com/documentation",
        "2025-12-31T23:59:59Z",
    )
}
```
I'm immediately unhappy here. Even for this trivial example DeprecatedResponder is leaking everywhere. We had to refactor the function and you still have to go into the function body to actually figure out when this function deprecates. It's easy to imagine this scaling poorly for a non-trivial endpoint. At least it won't be pretty.

## Enter procedural macros
Like I mentioned before with Rocket I am already using a macro to turn our function into something that can be consumed as a HTTP endpoint, the [#[get(...)] attribute](https://api.rocket.rs/v0.5/rocket/attr.get). Even without having implemented our own macro yet intuition says we should be able to get very close to the FastAPI implementation with something like this. 

```rust
#[get("/example")]
#[deprecation(
    "2024-12-31T23:59:59Z",
    link = "https://api.example.com/docs",
    sunset = "2025-12-31T23:59:59Z"
)]
pub fn example() -> &'static str {
    "Hello, world!"
}
```

### Setup
Turns out there are even more steps before we can get to implementation. Because proc macros manipulate [token streams](https://doc.rust-lang.org/reference/procedural-macros.html#r-macro.proc.proc_macro.token-stream) they need to be deployed as a separate crate that is compiled before any downstream crate. Because of this proc macro crates also cannot export anything but macros. Because our macro depends on the struct `DeprecatedResponder` we actually need two crates to make this work. This is the workspace setup I used for this.

```
├───.git
├───.vscode
├───rocket_sunset
│   ├───src
│   ├───tests
|   └───Cargo.toml //Re-exports proc macro crate, exports DeprecationResponder
├───rocket_sunset_macro
|   ├───src
|   └───Cargo.toml //Proc macro crate, internal to rocket_sunset
└───Cargo.toml //Workspace definition   
```
```toml
[package]
name = "rocket_sunset"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]
name = "rocket_sunset"
path = "src/lib.rs"

[dev-dependencies]
trybuild = "1.0.41"

[dependencies]
rocket = { version = "0.5.1", features = ["json"] }
rocket_sunset_macro = { version = "0.1.0", path = "../rocket_sunset_macro" }
```
{: file="~/rocket_sunset/Cargo.toml" }

```toml
[package]
name = "rocket_sunset_macro"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
proc-macro2 = "1.0.92"
quote = "1.0.37"
chrono = "0.4.38"
syn = { version = "2.0.90", features = ["proc-macro", "parsing"] }
rocket = { version = "0.5.1", features = ["json"] }
```
{: file="~/rocket_sunset_macro/Cargo.toml" }

There's a lot of setup and dependencies to unpack here but we now have a crate `rocket_sunset_macro` that is used as an internal dependency for `rocket_sunset` that can by consumed downstream as a library crate. `rocket_sunset_macro`, while available as a published crate, would not be used directly. Instead the relevant macro is re-exported from `rocket_sunset` that also contains the `DeprecatedResponder` struct.

### The basics
It's been a long time coming but we can finally get into implementation. If you're like me your primary macro experience before Rust were pre-processor defines in C/C++.
```c
#define DEBUG_PRINT(fmt, ...) \
    fprintf(stderr, "[DEBUG] %s:%d:%s(): " fmt "\n", __FILE__, __LINE__, __func__, ##__VA_ARGS__)

//Source
int main() {
    int a = 42;
    DEBUG_PRINT("Program started. a = %d", a);
    return 0;
}

//Post macro expansion
int main() {
    int a = 42;
    fprintf(stderr, "[DEBUG] %s:%d:%s(): Program started. a = %d\n", __FILE__, __LINE__, __func__, a);
    return 0;
}
```
It's simple and gets the job done but it has several downsides. As they get more advanced they become difficult to understand, maintain and debug. They're also considered [non-hygienic](https://en.wikipedia.org/wiki/Hygienic_macro) because they risk introducing variable conflicts, scope conflicts and multiple evaluation of arguments.

Rust is able to dodge these issues at least partially by using the `TokenStream` that is parsed to a [Abstract Syntax Tree (AST)](https://rustc-dev-guide.rust-lang.org/syntax-intro.html) for macro implementations instead of relying on text substitution. Using the `TokenStream` as a source allows the Rust compiler to maintain existing scoping. It also allows for name mangling in the generated code to avoid conflicts.

Let's look at the "unit" proc macro, a macro that when applied to a function returns the function in an unmodified state.

```rust
use proc_macro::TokenStream;

/// A procedural macro attribute that returns the unmodified token stream.
/// #[unit]
/// fn example() {
///     println!("This function is unmodified.");
/// }
#[proc_macro_attribute]
pub fn unit(_attr: TokenStream, item: TokenStream) -> TokenStream {
    // `_attr` is the attribute's token stream, ignored here.
    // `item` is the token stream of the decorated function or item.

    // Return the original item unchanged.
    item
}
```
We have two input `TokenStream` and we return, in this case, an unmodified `TokenStream`.

_attr: TokenStream
: This is our proc macro attribute input. `#[unit]` has no input but for `#[deprecation(...)]` it would be the following as a sequence of tokens `"2024-12-31T23:59:59Z", link = "https://api.example.com/docs", sunset = "2025-12-31T23:59:59Z"`.

input: TokenStream
: This stream contains the tokenized version of the function that the macro has been applied on.

We have a few problems we need to solve, let's try and tackle them in the following order:

1. Parse and validate macro arguments.
2. Wrap the function signature's return type with our DeprecatedResponder.
3. Return a DeprecatedResponder that wraps the function implementation as `inner`.

#### A note on dependencies
The Rust community has organically coalesced around [proc-macro2](https://docs.rs/proc-macro2/latest/proc_macro2/), [syn](https://docs.rs/syn/latest/syn/) and [quote](https://docs.rs/quote/latest/quote/) as the de facto toolkit for procedural macro development. While not mandated by the language or its core libraries, these tools have become the unofficial standard due to their robustness, ease of use, and deep integration into the ecosystem. 

[proc-macro2](https://docs.rs/proc-macro2/latest/proc_macro2/)
: A stable, portable and feature rich token stream library

[syn](https://docs.rs/syn/latest/syn/)
: Parses `TokenStream` into AST, comprehensive error handling and diagnostics

[quote](https://docs.rs/quote/latest/quote/)
: Enable the creation of `TokenStream` through a declarative syntax


### Argument parsing and validation
Like I mentioned above we have a separate `TokenStream` that contains all our input arguments to our macro. We could just parse this stream directly inside our macro function body but let's use the [parse trait from syn](https://docs.rs/syn/latest/syn/parse/index.html) instead.

```rust
#[derive(Debug)]
struct DeprecationMacroInput {
    timestamp: String,
    link: QuoteOption<String>,
    sunset: QuoteOption<String>,
}
```
The only thing of note here is `QuoteOption<T>`. `quote` interpolates `Option<T>` in the following manner:

```rust
Some(x) => x
None => //Empty token
```
This is the most consistent and correct behavior as detailed in [this issue](https://github.com/dtolnay/quote/issues/20) but it is not what we want in our case because `DeprecatedResponder` expects `link` and `sunset` to be `Optional<String>`. Due to this we implement `QuoteOption<T>` and the [ToTokens trait](https://docs.rs/quote/latest/quote/trait.ToTokens.html) from `quote`:

```rust
#[derive(Debug)]
pub struct QuoteOption<T>(Option<T>);

impl<T: std::fmt::Debug> From<Option<T>> for QuoteOption<T> {
    fn from(option: Option<T>) -> Self {
        QuoteOption(option)
    }
}

impl<T: ToTokens> ToTokens for QuoteOption<T> {
    fn to_tokens(&self, tokens: &mut TokenStream) {
        match &self.0 {
            Some(t) => tokens.append_all(quote! { ::std::option::Option::Some(#t) }),
            None => tokens.append_all(quote! { ::std::option::Option::None }),
        }
    }
}
```

We use the [quote!](https://docs.rs/quote/latest/quote/macro.quote.html) macro to perform variable interpolation and produce our output `TokenStream`. Note the `#t` that grabs the `t` in our scope and inserts it at the designated position for our output tokens. This works for any type that implements `ToTokens` (as our generic bound specifies already). Now when interpolating `QuoteOption<T>` we instead get:

```rust
Some(x) => ::std::option::Option::Some(x)
None => ::std::option::Option::None
```

Now that our data structure is ready we can start implementing parsing. We will do this through the [parse trait](https://docs.rs/syn/latest/syn/parse/index.html) from `syn`. According to the proposed standard the deprecation timestamp is mandatory and we treat is as such so let's make it a positional argument to our macro that has to be first in the argument list.

All input parsing takes place within the following function:
```rust
impl Parse for DeprecationMacroInput {
    fn parse(input: ParseStream) -> Result<Self> {
      ...
    }
}
```

This is the code necessary to extract a string literal and turn it into an epoch timestamp.
```rust
//Checks that the first token of the macro argument list is not empty and a string literal
let timestamp: LitStr = input
    .parse()
    .map_err(|e| syn::Error::new(input.span(), format!("Missing or invalid deprecation timestamp, {}", e)))?;

//Attempts to parse the string to a ISO8601 timestamp and convert it to epoch
//Throws an error if the string is not a valid ISO8601 timestamp
let timestamp: i64 = DateTime::parse_from_rfc3339(&timestamp.value())
    .map_err(|_| {
        syn::Error::new(
            timestamp.span(),
            "Deprecation timestamp is not a valid ISO8601 timestamp",
        )
    })?
    .timestamp();
```
The interesting functions here are `parse` and `span`. `parse` parses the current syntax tree node in our [ParseStream](https://docs.rs/syn/latest/syn/parse/struct.ParseBuffer.html) and advances the stream to the next position within the stream. It doesn't implement any iterator traits but for all intents and purposes we are iterating through this buffer.

`span` contains context information, primarily the location textually within the source code, about the token or stream in question. This information allows us to accurately report compile errors that point the user to the token that failed compilation which is what we do when we return [syn::Error](https://docs.rs/syn/latest/syn/struct.Error.html). As an example we can see what our error message looks like if we skip the timestamp and only input a sunset timestamp:

```
error: Missing or invalid deprecation timestamp, expected string literal
  --> tests/build/missing_deprecation.rs:11:15
   |
11 | #[deprecation(sunset = "2024-01-01T00:00:00Z")]
   |               ^^^^^^
```
As you can see `sunset` is correctly highlighted in our error message here, clearly indicating where the error is. Now this error message isn't perfect but it gets us started.

If all we had were positional arguments we would just repeat this process two more times and we'd be done. But since `sunset` and `link` are optional the following parsing is slightly more involved.

```rust
//Keep looping over the stream as long as the next token is ","
//Each iteration consumes everything up until the next ","
//We make extensive use of error propagation (?) 
//where we don't care about custom compiler errors
while input.peek(Token![,]) {
    //Peek does not consume any token so we grab it here and discard the result
    input.parse::<Token![,]>()?;

    //An Ident is any word of valid Rust code
    //In our case it's the keyword for our optional arguments
    let ident: Ident = input.parse()?;

    //Because both sunset and link are optional we need to match to see what identifier we got
    match ident.to_string().as_str() {
        "link" => {
            //We don't need to care about whitespace 
            //we do need to consume "=" to make sure the macro is adhering to proper syntax
            input.parse::<Token![=]>()?;
            link = Some(input.parse::<LitStr>()?.value());
        }
        "sunset" => {
            input.parse::<Token![=]>()?;
            let sunset_raw = input.parse::<LitStr>()?;

            //We enforce ISO8601 timestamps for everything just for consistency's sake
            let sunset_ts = DateTime::parse_from_rfc3339(&sunset_raw.value()).map_err(|_| {
                syn::Error::new(
                  sunset_raw.span(), 
                  "Sunset timestamp is not a valid ISO8601 timestamp")
            })?;

            //We can use the sunset_raw.span() for both validations here
            //as they refer to the same token
            if sunset_ts.timestamp() < timestamp {
                return Err(syn::Error::new(
                    sunset_raw.span(),
                    "Sunset timestamp must not be earlier than deprecation timestamp",
                ));
            }

            sunset = Some(sunset_ts.format("%a, %d %b %Y %H:%M:%S GMT").to_string());
        }
        _ => {
            //The fallthrough case handles any keyword argument that isn't link or sunset
            return Err(syn::Error::new(
                ident.span(),
                "Expected 'link' or 'sunset' identifiers.",
            ))
        }
    }
}
```
We can now populate and return our `DeprecationMacroInput`:
```rust
...
Ok(DeprecationMacroInput {
    timestamp: timestamp.to_string(),
    link: link.into(),
    sunset: sunset.into(),
  })
}
```

Let's get this into our macro implementation using `parse_macro_input!` from `syn`.

```rust
#[proc_macro_attribute]
pub fn deprecation(args: proc_macro::TokenStream, input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    let args = parse_macro_input!(args as DeprecationMacroInput);

    ...

    let timestamp = args.timestamp;
    let link = args.link;
    let sunset = args.sunset;

    ...
}
```

Finally! We've parsed all our arguments in a type safe way with validation and everything. With this as a foundation I think it should be easy to build any kind of macro argument parsing in the future and it wasn't much of a hassle all things considered. It's worth considering breaking this into nicer composable chunks but this is good enough for now.

### Modifying the function signature

Next up we want to modify the function signature to return `DeprecatedResponder<&'static str>` to make sure the signature matches our future return. Lucky for us `syn` already ships with support for [parsing the function](https://docs.rs/syn/latest/syn/struct.ItemFn.html).

```rust
let ItemFn { attrs, vis, sig, block } = parse_macro_input!(input as ItemFn);
```
We only care about `sig` for the changes we are doing but let's look at all of the information we get.

[attrs](https://docs.rs/syn/latest/syn/struct.Attribute.html)
: This is a vector of attributes that can contain any attribute that is valid for a function such as macros, documentation or comments

[vis](https://docs.rs/syn/latest/syn/enum.Visibility.html)
: An enum representing the visibility of the function

[sig](https://docs.rs/syn/latest/syn/struct.Signature.html)
: The signature of our function

[block](https://docs.rs/syn/latest/syn/struct.Block.html)
: The function body

We're again going to use the `quote!` macro here, same as we did for `QuoteOption<T>`.
```rust
let output = match &sig.output {
    ReturnType::Type(_, ty) => {
        quote! { -> ::rocket_sunset::DeprecatedResponder<#ty> }
    }
    ReturnType::Default => {
        quote! { -> ::rocket_sunset::DeprecatedResponder<()> }
    }
};
```
Here we create a `TokenStream` that takes the existing output type `#ty` and wraps it in our `DeprecatedResponder`. Notice that we need to add the `->` token as well or we'll end up with invalid syntax. The unit type `()` is separate because we have no type information to use for the match expression but as you can see there is no real difference.

Now we need to put it together. Recall the unit macro we implemented above where we just returned `input`. Since we are modifying the function signature we need to reconstruct the entire function as a new `TokenStream` from its individual components.

```rust
  ...
  //fn_name and fn_args are fetched from sig
  let expanded = quote! {
      #(#attrs)*
      #vis fn #fn_name(#fn_args) #output #block
  };

  expanded.into()
}
```
We now have our entire function reconstructed with our modified return type in `#output`. There's nothing really new here except for `#(#attrs)*`. This iterates through `#attrs` and interpolates each element, in this case our attributes, without any separator.

### Wrapping our result
Time to put it all together and finalize this macro by wrapping `#block`, that contains our function body, in `DeprecatedResponder`. We simply replace `#block` with the following code:

```rust
let expanded = quote! {
    #(#attrs)*
    #vis fn #fn_name(#fn_args) #output {
        ::rocket_sunset::DeprecatedResponder::new(
            #block,
            #timestamp,
            #link,
            #sunset
        )
    }
};
```
Instead of directly returning `#block` we now insert it into `DeprecatedResponder` as its `inner` member, together with the input parameters that we parsed from our macro arguments. Due to the declarative syntax of `quote` it's really easy to differentiate between what tokens we are directly inserting through text into the stream and what are variables within our current scope.

Let's put it all together to see the final function.

```rust
#[proc_macro_attribute]
pub fn deprecation(args: proc_macro::TokenStream, input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    let args = parse_macro_input!(args as DeprecationMacroInput);

    let ItemFn { attrs, vis, sig, block } = parse_macro_input!(input as ItemFn);

    let fn_name = &sig.ident;
    let fn_args = &sig.inputs;

    let output = match &sig.output {
        ReturnType::Type(_, ty) => {
            quote! { -> ::rocket_sunset::DeprecatedResponder<#ty> }
        }
        ReturnType::Default => {
            quote! { -> ::rocket_sunset::DeprecatedResponder<()> }
        }
    };

    let timestamp = args.timestamp;
    let link = args.link;
    let sunset = args.sunset;

    let expanded = quote! {
        #(#attrs)*
        #vis fn #fn_name(#fn_args) #output {
            ::rocket_sunset::DeprecatedResponder::new(
                #block,
                #timestamp,
                #link,
                #sunset
            )
        }
    };

    expanded.into()
}
```

We now have a fully featured procedural macro attribute that takes a function, some arguments and modifies the result of said function. We have reached the goal that was set initially, no refactoring required and the deprecation status of a resource is clearly visible just by looking at the function signature. Let's see what our expanded result looks like using [cargo-expand](https://github.com/dtolnay/cargo-expand).

```rust
#[doc(hidden)]
#[allow(unused)]
pub use rocket_uri_macro_full_7976942622727546180 as rocket_uri_macro_full;
fn example() -> ::rocket_sunset::DeprecatedResponder<&'static str> {
    ::rocket_sunset::DeprecatedResponder::new(
        { "Hello, world!" },
        "1735689599",
        ::std::option::Option::Some("https://api.example.com/docs"),
        ::std::option::Option::Some("Wed, 31 Dec 2025 23:59:59 GMT"),
    )
}
```
I need to research more on macro hygiene and see what kind of improvements should be made here but for a first attempt at implementing my own proc macro in Rust I'm pretty happy with how this turned out. It's definitively a solid base for future exploration for me and I feel like I have a better understanding of the most important bits within the macro ecosystem.

## Unit testing
You might have noticed that `rocket_sunset` contained a dev dependency called [trybuild](https://github.com/dtolnay/trybuild). Trybuild is a test harness that allows for unit testing of error reporting involving procedural macros, also known as _ui testing_. Using this gave me a lot of confidence during development and refactoring of my macros. The essence of these tests is using a Rust source file that is compiled with rustc and then the output gets compared to a `.stderr` file to see if it matches. These `.stderr` files can be generated by `trybuild`.

```rust
use rocket::{
    get,
    routes,
};
use rocket_sunset::{
    deprecation,
    DeprecatedResponder,
};

#[get("/")]
#[deprecation("2024-12-31T23:59:59Z", invalid = "uri_placeholder")]
pub fn index() -> &'static str {
    "Hello, world!"
}

fn main() {
    let _ = rocket::build().mount("/", routes![index]);
}
```
{: file="~/invalid_ident.rs" }

```
error: Expected 'link' or 'sunset' identifiers.
  --> tests/build/invalid_ident.rs:11:39
   |
11 | #[deprecation("2024-12-31T23:59:59Z", invalid = "uri_placeholder")]
   |                                       ^^^^^^^

warning: unused import: `DeprecatedResponder`
 --> tests/build/invalid_ident.rs:7:5
  |
7 |     DeprecatedResponder,
  |     ^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

error[E0423]: expected function, found struct `index`
  --> tests/build/invalid_ident.rs:12:19
   |
10 | #[get("/")]
   | -----------
   | |
   | `index` defined here
   | in this procedural macro expansion
11 | #[deprecation("2024-12-31T23:59:59Z", invalid = "uri_placeholder")]
12 | pub fn index() -> &'static str {
   |                   ^ help: use struct literal syntax instead: `index {}`
   |
   = note: this error originates in the attribute macro `get` (in Nightly builds, run with -Z macro-backtrace for more info)
```
{: file="~/invalid_ident.stderr" }

You can see samples of this implemented for `rocket_sunset` [here](https://github.com/Bentebent/rocket_sunset/tree/main/rocket_sunset/tests).

## Conclusion
In this blog I've done my best to cover my first foray into developing my own proc macros for Rust. When building this on my own I struggled to find resources that covered the process from end-to-end, especially with the more boilerplate heavy bits such as how to organize my crates. I hope that if you are new to proc macros and read this you found at least something helpful on what is a tool that feels incredibly powerful and way more enticing to use than old school C-style preprocessor defines.

If you want to checkout the full source code you can find [rocket_sunset on Github](https://github.com/Bentebent/rocket_sunset) or on [crates.io](https://crates.io/crates/rocket_sunset).

Thank you for reading!
