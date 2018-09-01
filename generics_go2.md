## Thoughts about generics and contracts

The point of contracts is you just write the code to specify the contract.

Why not just write the code to specify the contract? Instead of making
"contracts" a distinct thing, let people use shared type names across
multiple functions, and simply use the actual code of the functions *as*
the contracts.

So, for instance:

```go

type t1, t2 parametric

func Sum(type t1)(s []t1) t1 {
	var t t1
	for _, x := range s {
		t += x
	}
	return t
}

func Keys(type t1, type t2)(m map[t1]t2, s []t1) []t2 {
	var a []t2
	for _, x := range s {
		if v, ok := m[x]; ok {
			a = append(a, m[x])
		}
	}
	return a
}
```

This requires both that t1 be addable and that x be an acceptable map key. You
don't need to write a separate contract, or determine whether a contract
applies to just one type or multiple types. Each function using one of those
types contributes to that type's requirements or contracts; in this case,
because Sum and Keys both use "t1", Sum can't be instantiated for a type
which isn't comparable, and Keys can't be instantiated with a key type that
isn't addable.

One other thing that might be useful: Allow parametric functions named "_"
which exist only to communicate type contracts.

The obvious problem with this is that you can accidentally change a contract.
On the other hand, it's not obvious what we should do now if one of the
implementation functions uses functionality not guaranteed by a contract.
Another alternative would be to keep contracts explict, but still use
the simpler form of "you just declare functions on these types, marked
in some way as being the contract(s)". I still think it's nicer and
cleaner to be able to share and combine contracts, and more idiomatic
to just use standard function declarations to do it.
