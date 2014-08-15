% Destructors

Unlike constructors, destructors in Rust have a special status: they are added
by implementing `Drop` for a type, and they are automatically invoked as values
go out of scope.

In addition to destroying values reliably and deterministically,
destructors provide *failure safety* by ensuring that resources are cleaned
up properly when a task fails.
Interactions between destruction and failure can lead to
situations that are tricky, or even "impossible",
to handle correctly,
but by following some guidelines can be trusted to mostly work just fine >_<.

> **[FIXME]** This section needs to be expanded.

### Destructors should not fail. [RFC]

When a task fails it *unwinds* the stack,
destroying every value on it,
and thus everything owned by the task.
*A second task failure during unwinding is a unrecoverable error*,
and will force the program to abort.
Because of this, any destructor that might fail risks aborting the program.

Instead of failing in a destructor,
provide an explicit destructor method, e.g. `close`,
that consumes `self`,
and that returns a `Result` to signal problems.

### Destructor bombs. [RFC]

In cases where a type must be destroyed via an explicit method call,
use the destructor to verify that the type was destroyed correctly.

```
struct Database {
    destroyed: bool
}

impl Database {
    fn new() -> { Database { destroyed: false } }

    fn close(self) -> Result<(), ()> {
        // destruct some things
        // ...

        // disarm the dtor bomb
        self.destroyed = true;
        drop(self);

        Ok<()>
    }
}

impl Drop for Database {
    fn drop(&mut self) { assert!(self.destroyed) }
}
```

This adds a runtime guarantee that the value is destroyed
correctly via the explicit destructor.
Without a destructor bomb,
and without otherwise having a destructor,
it would be very easy to accidentally forget
to destroy a value.

This pattern has a downside:
while it ensures failure safety by making the user responsible
for destroying the value,
and it adds a runtime check to ensure that the value is destroyed correctly,
if the user doesn't call the destructor correctly then the `drop` assertion will fail,
*triggering the double failure we were trying to avoid*.

To maintain failure safety,
the user of the type must either disarm the destructor bomb during task failure,
likely from another destructor,
or it must wrap it in another destructor bomb for some code further up the stack to disarm.

### Destructors should not block. [RFC]

Similarly, destructors should not invoke blocking operations,
which can introduce unexpected latency,
and make debugging much more difficult.
Again, consider providing a separate method for explicit destruction.

### Destructors should just do the best they can. [RFC]

Even when you might prefer that users call an explicit destructor
to absolve yourself of the whole destructor-unwinding paradox,
it's often more user-friendly to write a destructor that makes
minor compromises in order to maintain failure safety.

This could mean compromises that result in blocking, ignoring errors,
or even "minor" memory leaks! What must never be compromised though
is *memory safety*.

Such situations should be accompanied with thorough documentation.

As an example,
when writers with buffers (such as `BufferedWriter`) are destroyed they attempt
to flush any remaining bufferred data.
There is a small chance that the flush will return an error,
and because the `drop` method cannot fail,
the flush error must be ignored.
Callers who are concerned about this behavior should flush `Writer`s
before dropping them.
