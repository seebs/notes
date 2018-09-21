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
		}
	}()
	out = "sad"
	if explode {
		panic("oops")
	}
	return "happy"
}
```

If `panicky` is called with `true`, it will return "oops sad". The value
assigned to `out` is available to it, and the value passed to `panic` is
available to it also (as an `interface{}`). If it is called with `false`,
it will return "happy". The `return` statement effectively assigns the
given expression (or expressions for a multi-valued function) to the
return variables. The deferred function can access the values that were
stored in those variables, whether by a `return` statement or direct
assignment, and can modify those variables.
