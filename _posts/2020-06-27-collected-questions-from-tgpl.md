---
layout    : post
title     : "Collected Questions From TGPL"
date      : 2020-06-27
lastupdate: 2020-06-27
categories: go
---

### Index

* TOC
{:toc}

# 1. Short variable declarations with `:=`

1. What's the difference between `=` and `:=`? 
1. Is there any problem in the following code?

    ```go
    func main() {
    	err := fmt.Errorf("")
    	a, err := f() // Q: compile error?
    	b, err := f() // Q: compile error?
    	...
    }
    ```

1. And how about this one:

    ```go
    func main() {
    	c, err := f()
    	c, err := f() // Q: compile error?
    	...
    }
    ```

# 2. Explain Unicode, `rune` and UTF-8 and their relationships

Tips:

* Charater set
* A character's index in a given character set
* Encoding (compressing) to the index


# 3. Convert integer into comma separated format

Take a string representation of an
integer, such as "12345", insert commas every three places, as in "12,345".

Implementation constraints:

* Provide both iterative and recursive style versions
* Assume input is valid
* Keep the code as short as possible

```go
func comma(s string) string {
}
```

<a name="q4"></a>

# 4. Nil slice & empty slice

1. What's the difference between **a nil slice** and **an empty slice**?
1. What are their respective `len` (length) and `cap` (capacity)?
1. If we have a slice `s`, with `len(s) == 0` and `cap(s) == 0`, then `s` must
   be a `nil` slice?

Decide the `len(s)` in following expressions, and whether `s == nil`: 

```go
var s []int    // len(s) == ?, s == nil ?
s = nil        // len(s) == ?, s == nil ?
s = []int(nil) // len(s) == ?, s == nil ?
s = []int{}    // len(s) == ?, s == nil ?
```

<a name="q5"></a>

# 5. Address of a map item

Can we get a map item's address? Why?

```go
_ = &m[k] // Q: compiler or runtime error?
```

Further more, can we get a slice item's address? Why?

```go
_ = &s[k] // Q: compiler or runtime error?
```

<a name="q6"></a>

# 6. Operation on nil maps

Is is safe to lookup, delete, iterate over a `nil` map? and how about insert?

```go
var ages map[string]int
fmt.Println(ages == nil)    // Q: true?
fmt.Println(len(ages) == 0) // Q: true?

ages["carol"] = 21          // Q: what happens?
```

<a name="q7"></a>

# 7. Decide if two maps are equal

Following function is used for testing whether two map instances (with type
`map[string]int`) are equal:

```go
func equal(x, y map[string]int) bool {
	if len(x) != len(y) {
		return false
	}
	for k, v := range x {
		if v != y[k] {
			return false
		}
	}
	return true
}
```

Does the function have any bugs? If cannot see any problem, consider
following case:

```go
x := map[string]int{"A": 0}
y := map[string]int{"B", 20}
fmt.Println(equal(x, y))     // Q: true or false
```

What will be the output?

# 8. JSON marshal/unmarshal

# 9. `len` and `cap` of a slice

# 10. Reverse a slice in place

Implement a `reserve()` function, which takes a slice (type `[]int`) as input,
and reverses all the items in the given slice.

For example:

```go
s := []int{1, 2, 3, 4}
reverse(s)
fmt.Println(s) // output [4, 3, 2, 1]
```

Can you implement the body with no more than 3 lines of code?

```go
func reverse(s []int) {
    // line 1
    // line 2
    // line 3
}
```


# 11. Why can not compare two slices?

# 12. Exponential backoff retry

# 13. Lexical scope

# 14. Variadic function

# 15. `defer` inside loop


# 16. `defer` execution order after panic

# 17. get call stack after panic

# 18. Simple breadth first crawler

# 19. Custom type: auto convert value to pointer


# 20. safely copy

# 21. Nil Is a Valid Receiver Value

# 22. Interface as contracts

