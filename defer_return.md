# Using `defer` to change return values in Go

It is possible for a deferred function call to modify the return value
of its caller in go. For this to happen, two things must be the case:

1. The return value must be named.
2. The defer function must be a function literal declared inside
the caller.

Because function literals are closures and have access to the
enclosing environment's local variables, the deferred function can
modify the named return values, which are treated as local variables
declared at the beginning of the function.

This is particularly relevant in the case where a function needs
to use `recover` to recover from a call to `panic`.

Example:

```
func panicky(explode bool) (out string) {
	defer func() {
		if r := recover(); r != nil {
			out = r.(string) + " " + out
		} else {
			out = "yay " + out
		}
	}()
	out = "sad"
	if explode {
		panic("oops")
	}
	out = "happy"
	return out
}
```

If `panicky` is called with `true`, it will return "oops sad". The value
previously assigned to `out` is available to it, and the value passed to `panic`
is also available to it (as an `interface{}`). If it is called with `false`,
it will return "yay happy". The `return` statement assigns the
given expression (or expressions for a multi-valued function) to the
return variables. The deferred function can access the values that were
stored in those variables, whether by a `return` statement or direct
assignment, and can modify those variables.

Now consider this version of the same function, with an unnamed
return value, but a variable declared at the top of the function:

```
func panicky(explode bool) (string) {
	var out string
	defer func() {
		if r := recover(); r != nil {
			out = r.(string) + " " + out
		} else {
			out = "yay " + out
		}
	}()
	out = "sad"
	if explode {
		panic("oops")
	}
	out = "happy"
	return out
}
```

If the panic triggers, the variable `out` is modified to contain "oops
sad", but this has no impact on the returned value. If the `panic`
doesn't trigger, the function just returns "happy". The `return` statement
obtains the current value of `out`, and assigns it to the unnamed return
value. Then the defer executes, and changes the value of the variable
`out`. However, there's no connection between that variable and the return
value. In short, `return` statements are pass-by-value; they compute the
actual values of the expressions given to them, and push those values
into the return value space. Changes to variables used by a `return`
statement after the return statement executes are irrelevant. (In the
previous case, the reason the `defer` mattered is that the change was
to the actual return value variable; that the variable had been recently
used in a `return` statement is a coincidence.)

This seems to be a recurring source of confusion, and I think the fault is
in part an unclarity in the spec. The spec states that a defer executes
"immediately before the surrounding function returns". It's important
to make a distinction between "executes a `return` statement" and "flow
control passes to the caller".
