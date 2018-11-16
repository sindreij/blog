---
layout: post
title: "Bringing Elm's architecture to Rust and Webassembly"
date: 2018-11-16 17:35:22 +0100
categories: willow rust elm
---

tl;dr: See <https://github.com/sindreij/willow>

I really like Elm. It is a delightful language with an amazing ecosystem. It
has an interesting architecture called TEA, The Elm Architecture. TEA and Elm
are not separable, you can't have one thing without the other. And that makes
Elm a pleasure to use. TEA has also been an inspiration for how Redux is used to
handle state in the React ecosystem.

Another language I like is Rust. On paper, Rust is completely different from
Elm, but in using them both, I have seen some resemblance. They both have great
type systems that make it easier to refactor and gives few[^1] runtime exceptions.
They have similar support for tagged unions and pattern matching. They both
handle errors using the sum-type `Result`. It's easy to see how they are
inspired by a similar set of languages.

The big difference is that Elm compiles to JS and Rust compiles to machine code.
The main goal for Elm is writing frontend to be run in the browser,
while Rust can only be run natively and is primarily meant to be used for servers and
other native code.

Until recently.

There is something brewing, called WebAssembly, or wasm for short. It is a
compile target that makes it possible to run Rust (and C and C++ and lots of
more languages) in the browser. Rust has had support for compiling to wasm for a
while, but in the last few months the support has become really great, the
Rust team has created some amazing tools that make using Rust with wasm a
breeze.

## The idea

Having used both Elm and Rust I had something I wanted to try. Would it be
possible to create The Elm Architecture in Rust?

This is a basic Elm app (excluding the imports):

```elm
type Msg
    = Increment
    | Decrement

type alias Model =
    { counter : Int
    }

update : Msg -> Model -> Model
update msg model =
    case msg of
        Increment ->
            { model | counter = model.counter + 1 }

        Decrement ->
            { model | counter = model.counter - 1 }

view : Model -> Html Msg
view model =
    div []
        [ button [ onClick Increment ] [ text "+" ]
        , div [] [ text (String.fromInt model.counter) ]
        , button [ onClick Decrement ] [ text "-" ]
        ]

main: Program () Model Msg
main =
    Browser.sandbox { init = Model 0, update = update, view = view }

```

We have a `Msg` which is the action type in Elm.
Each event in Elm creates one such Message. Then we have the `update` function
which takes in a message and a model and returns the new model. We have a
`view` function which takes a model and returns the HTML to render. And at last,
we have a `main`-function that connects everything.

If we try to translate this to Rust, it will become something like:

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Msg {
    Increment,
    Decrement,
}

#[derive(Debug, Clone)]
pub struct Model {
    counter: i32,
}

fn update(msg: &Msg, model: &mut Model) {
    match msg {
        Msg::Increment => model.counter += 1,
        Msg::Decrement => model.counter -= 1,
    }
}

fn view(model: &Model) -> Html<Msg> {
    div(
        &[],
        &[
            button(&[on_click(Msg::Increment)], &[text("+")]),
            div(&[], &[text(&model.counter.to_string())]),
            button(&[on_click(Msg::Decrement)], &[text("-")]),
        ],
    )
}

pub fn main() -> Program<Model, Msg> {
    Program::new(view, update, init: Model { counter: 0 }}
}
```

This is compilable Rust code, and using this project it will render and run exactly the
same as the Elm code. You can try it [here](http://sindrejohansen.no/willow/counter/)[^2]

Note how much the rust code resembles the Elm code. The `Msg` is translated from an Elm `type`
to a Rust `enum`, but apart from having different names and syntax, it is exactly the same. The
`Model` is becoming a Rust struct. The largest change is in the `update` function. Rust has
no built-in support for immutable structures, so instead we mutate the model.

Rust's powerful borrow system means that we can control where the model is mutable, meaning that we can only
change it here in the update-function, and not for example in the view-function. Therefore
I think using mutations here will not mean that we are less safe than in Elm code.

Last we have the main function, which returns a `Program<Model, Msg>`. Closely resembling the same as
the `Program () Model Msg` returned from the main function in the Elm app. It's interesting
to what degree the types work out to look the same.

## Implementation

After having formulated the idea and written up some code, I started on the implementation. The
first iteration just rendered the HTML in the DOM. I then added events and messages and needed to update
the DOM. The first "virtual diffing" just deleted the whole DOM and recreated it with the new
HTML, but I later added a "real" dom-diffing algorithm.

Writing this is possible without writing any javascript-code, thanks to
the [web-sys](https://crates.io/crates/web-sys) and [js-sys](https://crates.io/crates/js-sys)
crates, which builds on [wasm-bindgen](https://crates.io/crates/wasm-bindgen).

All this is enough to render the TodoMVC application, but doing anything more than that will probably
mean you will miss something. For example a function `Html<A> => Html<B>` like Elm's
[Html.map](https://package.elm-lang.org/packages/elm/html/latest/Html#map).

## Conclusion

All in all, this is just an experiment to see how far Rust has come in doing web development.
I think the above shows it has come really far. I hope it will inspire someone to create the
next awesome web-framework using Rust, and that I one day can use Rust to write webapps in
my day-job.

I have written two examples of using willow,
[counter](https://github.com/sindreij/willow/blob/master/examples/counter/src/app.rs)
(the code above) and [todomvc](https://github.com/sindreij/willow/blob/master/examples/todomvc/src/app.rs).
TodoMVC is manually converted from Evan's [elm-todomvc](https://github.com/evancz/elm-todomvc).

If you found a spelling mistake, feel free to correct it
[here](https://github.com/sindreij/blog/blob/gh-pages/_posts/2018-11-16-willow-elm-in-rust.md)

[^1]: When a colleague proofread this, he argued that in elm case it should be "practically zero". NoRedInk, with 200 000 lines of elm-code in production has had exactly [one runtime exception](https://twitter.com/rtfeldman/status/961051166783213570). In Rust's case, you can create "Runtime Exceptions" using `panic!`, however, a panic should be a rarity. The difference in philosophy on how to handle errors is interesting when comparing these two languages. They are very similar, and at the same time very different.
[^2]: The only changes in the actual implementation of this is that the update function returns a Cmd. If Willow had a `sandbox` constructor the counter code using that would be exactly as this code. The various derives are needed for the Virtual Dom diffing and just for debugging.
