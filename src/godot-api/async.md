<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# Async programming

Execute logic using a future or a deferred function at the end of the frame.


## Running Deferred logic

Sometimes it is useful to run logic after all of the other logic of the other
nodes has complete.  While you can ask the Godot engine
[to execute exported Rust functions with `call_deferred`](https://godot-rust.github.io/gdnative-book/bind/calling-gdscript.html#function-calls)
, godot-rust also provides a type-safe way to defer executed logic to the next
frame with `run_deferred`.

```rust
use godot::prelude::*;

#[derive(GodotClass)]
#[class(init, base=Node)]
struct Game {
   base: Base<Node>,
}

#[godot_api]
impl Game {
  fn now_and_later(&mut self) {
      godot_print!("This was run at the beginning of the frame!");
      self.run_deferred(|_this| {
          godot_print!("This was run at the end of the frame!")
      });
  }
}
```


## Futures

Rust's futures are fully supported for integrating asynchronous programming.
The key point is that you will need you will need a Godot pointer that can be passed
to `godot::task::spawn`. The function must be setup to receive a Godot pointer
instead of the base class.

```rust
use godot::prelude::*;

#[derive(GodotClass)]
#[class(init, base=Node)]
struct Game {
   base: Base<Node>,
}

#[godot_api]
impl Game {
    async fn sleep(&self, duration: f64) {
        // Your logic to sleep here
    }

    // Show one message immediately, and other after one second.
    #[func(gd_self)] // Allow attaching the callback with the Godot editor.
    fn show_messages(this: Gd<Self>, _area: Gd<Node2D>) {
        godot::task::spawn(async move {
            godot_print!("Immediate message!");
            this.bind().sleep(1.0).await;
            godot_print!("Message after one second!")
        });
    }
}
```

While it is possible to get
[a Godot pointer inside of a class method](https://godot-rust.github.io/book/register/functions.html?highlight=bind_mut#calling-rust-methods-binds),
`bind()` and `bind_mut()` will not be able to return a guarded object as it is ready
implicitly bound for the method call.  The below code sample describes how this
approach would fail.

```rust
#[godot_api]
impl Game {
    fn crash_the_program(&mut self) {
        let gd = self.to_gd();

        // Furthermore `drop` is a noop on a borrowed value.
        std::mem::drop(self);

        godot::task::spawn(async move {
            // Because the `self` passed to `crash_the_program` has implicitly had
            // `bind_mut` called, the rebind below will cause the program to crash.
            gd.bind_mut();
        });
    }

}
```

Because both Rust and Godot run in the same process (and even in the same
thread), trying to block a thread outside of a spawned task waiting for the
future will cause the program to freeze.

```rust
#[godot_api]
impl Game {
    async fn sleep(&self, duration: f64) {
        // Your logic to sleep here
    }

    fn freeze_the_program(&mut self) {
        // The program will freeze when block_on is called here.
        // The correct approach is to use `godot::task::spawn` to
        // start async tasks.
        futures::executor::block_on(self.sleep(1.0));
    }

}
```
