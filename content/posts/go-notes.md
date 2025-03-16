---
title: Learning Go notes
date: 2025-03-16
tags: ["Go", "Programming"]
author: "Kapil Agrawal"
comments: false
---

# General rules

- Identifiers starting with an uppercase letter are exportable outside the package

| Type           | Zero value |
| -------------- | ---------- |
| bool           | false      |
| uint/int/float | 0          |
| string         | ""         |
| slice          | nil        |
| map            | nil        |
| pointer        | nil        |
| function       | nil        |
| interface      | nil        |
| chan           | nil        |

---

# Slices

- slice internals
  - https://go.dev/blog/slices-intro
  - https://developer20.com/what-you-should-know-about-go-slices/

```go
//Create a new slice. Capacity is optional
//By default, capacity = length during initialization
s := make([]Type, length, capacity)

//example
s := make([]string, 5)

//Append items to slice
s := append(s, "hi", "bye", "goodbye")

//Append one slice to another
s1, s2 := []int{10, 20, 30}, []int{50, 60, 70}
s1 := append(s1, s2...) //notice the ... variadic parameter unpacking operator

//Note:- This does not create a copy of elements. s3 points to the same underlying array as s2
s3 := s2

//changes made to s3 will affect s2 as well
s3[0] = 100

// Copy elements from one slice to another use this technique
s2Copy := make([]int, len(s2))
copy(s2Copy, s2)

//reset values in slice to it's zero value
var s := []int{10,20,30,40,50}
clear(s)                     // [0 0 0 0 0]
fmt.Println(len(s))          // 3
fmt.Println(cap(s))          // 3

```

---

# Maps

```go
//declaration
map[KeyType]ValueType

// key type = string ; value type = byte slice
var m map[string][]byte

// key type = int ; value type = string slice
var m := map[int][]string{
	10: []string{"Ten", "ten"},
	1: []string{"One", "one"}
}
```

In most cases, it's better to initialize the map right away (e.g., `m := make(map[int][]string))` because:

- Accessing or modifying a nil map results in runtime errors.
- It simplifies your code, reducing the risk of forgetting to initialize the map.

```go
// map is declared but uninitialized. This makes it a nil map
var m map[int]string

func main(){
	// reading a nil map is OK. it just prints an empty map[]
	fmt.Println(m)

	// writing to a nil map directly causes panic
	m[10] = []string{"Ten"}

	// Initializing makes it not-nill
	if m == nil{
		m = make(map[string][]string)
	}

	// prints an empty map[]
	fmt.Println(m)

	// Initializing made it writable
	m[10] = []string{"Ten"}

	// prints map[10:Ten]
	fmt.Println(m)
}

```

## The Comma-OK idiom

- Unlike Python, Go does not have a built-in way to check if a key is present in a Map or not
- Go’s map lookup does not throw errors for missing keys. Instead, it returns the zero value of the `ValueType` which makes this idiom essential for distinguishing between "key not present" and "key present with zero-value."

```go
package main

import fmt

func main(){

	totalWins := map[string]int{}
	totalWins["Orcas"] = 1
	totalWins["Lions"] = 2
	totalWins["Sharks"] = 0

	// Sharks is a valid key ; prints 0
	fmt.Println(totalWins["Sharks"])

	// Tigers is not a key ; also prints 0
	fmt.Println(totalWins["Tigers"])

	// to check whether a key is present in a map or not
	if _, ok := totalWins["Sharks"]; ok {
		// condition evaluates true as Sharks is a key
		fmt.Println("Sharks were present")
	}

	// condition evaluates false as Kittens is not a key
	if _, ok := totalWins["Tigers"]; ok {
		fmt.Println("Tigers were present")
	}

}
```

## Map operations

```go
//Define & initialize
totalWins := map[string]int{}

//Add an element
totalWins["Orcas"] = 1
totalWins["Lions"] = 2
totalWins["Sharks"] = 0

//Check if a key is present
if _, ok := totalWins["Tigers"]; ok{
	// do something
}

//Delete an element
delete(totalWins, "Lions")

//Remove all elements
clear(totalWins)

//Iterating over Map
for k,v := range totalWins{
	// do something
}
```

---

# Structs

- Define a struct when you have related data that needs to be grouped together

```go
//Using uppercase first letter (PascalCase) makes this struct exportable outside a package
type struct Employee{
	Id        uint64
	Name      string
	StartDate string
	Title     string
	Salary    uint64
}

//exportable
type struct Employee{
	Id        uint64
	Name      string
	Title     string
	startDate string    //unexported field
	salary    uint64    //unexported field
}

//non-exportable
type struct employee{
	id        uint64
	name      string
	startDate string
	title     string
	salary    uint64
}

```

## Anonymous struct

- Useful for marshaling a struct into JSON format and vice versa
- Remember, `struct` is a type. Anonymous struct types are NOT exportable outside the package where they’re declared because the type itself has no name, hence anonymous.
- But the variable which references the anonymous struct can be exported outside the package
- General conditions apply for what can be exported (upper case vs lower case)

```Go
// anonymous struct (non-exportable variable)
var employee struct{
	id        uint64
	name      string
	startDate string
	title     string
	salary    uint64
}

// anonymous struct (exportable variable)
var Employee struct{
	Id        uint64
	Name      string
	startDate string  //non-exportable
	Title     string
	salary    uint64  //non-exportable
}
```

---

# Functions

- Unlike Python, Go does not have named and optional input parameters
- Although it is possible to _simulate_ python `**kwargs` using a struct

```go
type Employee struct {
	Id     uint32
	Name   string
	Title  string
	salary uint32
}

// accepts struct as an input parameter
func ShowEmployeeInfo(empl Employee){
	fmt.Println(empl)
}

func main(){
	// notice the use of struct to pass optional number of args
	emp := Employee{
		Name: "John Doe",
		Id: 1000,
	}
	ShowEmployeeInfo(emp)
}
```

- Go does offer a way to pass arbitrary number of input parameters (variadic parameters) similar to Python's `*args`
- The variadic parameter must be the last (or only) parameter in the input parameter list. You indicate it with three dots (...) before the type

```Go
// accepts arbitrary number of args of type Employee
func HolidayBonusAdd(empl ...Employee) {
	// random number in range 10K(min) to 15K(max)
	bonus := rand.Intn(5000) + 10000
	for _, e := range empl {
		fmt.Println(e.Name, bonus+int(e.Salary))
	}
}

func main() {

	emp1 := Employee{
		Name:   "John Doe",
		Id:     1001,
		Salary: 85000,
	}
	emp2 := Employee{
		Id:     1003,
		Name:   "Chris P Bacon",
		Salary: 82000,
	}

	// option 1
	HolidayBonusAdd(emp1, emp2)
	// option 2 ; notice the use of variadic unpacking when passing a slice
	HolidayBonusAdd([]Employee{emp1, emp2}...)
	// option 3 ; same as option 2 but more idiomatic
	e := []Employee{emp1, emp2}
	HolidayBonusAdd(e...)
}

```

## Function as a type

- Functions are first class citizens in Go. They can be used as types

```Go
type myFuncType func(int, int) int

func Add(a, b int) int {
	return a + b
}

func Subtract(a, b int) int {
	return a - b
}

var calc map[string]myFuncType

func main(){
	a, b := 10, 20
	calc = map[string]myFuncType{
			"add":      Add,
			"subtract": Subtract,
			"multiply": Multiply,
		}
	fmt.Println(calc["add"](a, b))     //prints 30
}
```

- Functions can also be assigned directly to variables. This uses the same philosophy as above i.e function signature is considered a valid type in Go

```go
var myNewFunc func(string) int

func strlen(s string) int {
	return len(s)
}

func main(){
	myNewFunc = strlen
	count := myNewFunc("Hello World")
	fmt.Println(count)
}
```

## Anonymous function

- You can define new functions inside a function and assign them to variables. The inner functions don't have a name, hence anonymous.

```go
func AnonymousFunc() {
	anon := func(s string, j int) {
		for i := 0; i <= j; i++ {
			fmt.Println(s)
		}
	}
	anon("Happy New Year", 2025)
}

func main(){
	AnonymousFunc()
}
```

- Anonymous functions don't always need to be assigned to a variable. You can write them inline and call them immediately.

```go

func AnonymousFunc(y int) {
	func(s string) {
		for i := 1; i <= y; i++ {
			fmt.Println(s)
		}
	}("Happy New Year")
}

```

# Closure

- Functions declared inside functions are special; they are closures.
- Inner functions can access and modify the variables present in the outer function

> WARNING ⚠️
> using `:=` instead of `=` inside the closure creates a new variable that ceases to exist when the closure exits. When working with inner functions, be careful to use the correct assignment operator, especially when multiple variables are on the lefthand side.

```go
// ----- Example 1 ------- //
func SumClosure(a, b int) (sum int){
	func() int {
		sum := a + b      // new sum variable gets created which is scoped locally to inner func()
		return sum
	}()
	return sum
}
SumClosure(10,20)                     // returns 0



// ----- Example 2 ------- //
func SumClosure(a, b int) (sum int){
	func() int {
		sum = a + b     // uses the sum variable from outer function
		return sum
	}()
	return sum
}
SumClosure(10,20)                   // returns 30



// ----- Example 3 ------- //
type ComputeSum func() int
func SumClosure(a, b int) (cs ComputeSum) {
	cs = func() int {
		return a + b
	}
	return cs         // Since functions are values, inner functions can be also be returned
}

func main(){
fmt.Println(SumClosure(10,20))     // prints memory address of the returned function
fmt.Println(SumClosure(10,20)())   // prints 30
}



// ----- Example 4 ----- //
func SumClosure(a, b int) func() int {
	return func() int {            // returns the inner function (simplified version of Example 3)
		return a + b
	}
}

func main(){
fmt.Println(SumClosure(10,20)())   // prints 30
}
```

## Anonymous function vs. Closure

- In Go, most anonymous functions are closures.
- Here's the distinction:

> Anonymous function: Any function defined without a name. It can be assigned to a variable, used directly, or passed as an argument.
>
> Closure: An anonymous function becomes a closure if it interacts with variables from its surrounding lexical scope. This means it can access and modify variables declared outside its own body.

```go
// This anonymous function doesn't capture any external variables, so it's not a closure.
func main() {
    f := func(x int) int {
        return x * x
    }
    fmt.Println(f(5)) // Output: 25
}

// Captures 'y' from the surrounding scope. This is a closure
func main() {
    y := 10
    f := func(x int) int {
        return x + y
    }
    fmt.Println(f(5)) // Output: 15
}
```

# Pointers

- The zero value for a pointer is nil
- When passing a pointer variable to a function, you're actually passing _a copy_ of the pointer variable (memory address) which holds to the original data.
- A pointer variable holds a Memory address as it's value and just like any other variable, it's value (memory address) can be changed to point to another location in the memory.

>     With the operator * you follow the address
>     With the operator & you take the address

```go
func main() {
	num := 10
	addr := &num                                       // just another variable which holds the memory address of `num`
	fmt.Println("num, &num, addr : ", num, &num, addr) // 10 0x14000102020 0x14000102020

	addr = FunPtr(addr)                          // addr now stores a new memory address
	fmt.Println("addr, *addr", addr, *addr)      // 0x14000102028 20
	fmt.Println("num, &num, addr : ", num, &num) // 10 0x14000102020 ; But memory addr of `num` remains unchanged
}

// Let's try changing the pointer address
func FunPtr(i *int) *int {
	x := 20
	i = &x
	fmt.Println("value x, &x, i, *i ", x, &x, i, *i) // 20 0x14000102028 0x14000102028 20
	return i
}
```

### Performance with Pointers

https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html

- When returning values from a function, you should favor value types. Use a pointer type as a return type only if there is state within the data type that needs to be modified.
- But if you are passing megabytes of data between functions, consider using a pointer even if the data is meant to be immutable.

## Maps and Structs under the hood

- While maps may resemble to dictionary type in Python but they're not the same.
- Within the Go runtime, a map is implemented as a pointer to a struct. Passing a map to a function means that you are copying a pointer and which is why you should consider carefully before using maps for function input parameters or return values.
- Passing around Maps in function calls is discouraged because they say nothing about the values contained within; nothing explicitly defines any keys in the map, so the only way to know what they are is to trace through the code.
- Go is a strictly typed language and it likes to be explicit about everything.
- If you need to pass around a data structure with key:value pairs, instead of passing a map consider using struct or a slice of structs.

```go
/* Bad */
func main(){
	animals := map[string][]string{
	"Cat": {"Juniper", "Domestic Longhair", "Orange", "4"},
	"Dog": {"Bruce", "Labradoodle", "Golden", "8"},
	fmt.Println(animals)
	}
}


/* Good */
type Animal struct {
	Kind  string
	Name  string
	Breed string
	Color string
	Age   int
}

func main() {
	animals := make([]Animal, 5)
	animals = []Animal{
		{
			Age:   4,
			Kind:  "Cat",
			Breed: "Domestic Longhair",
			Name:  "Juniper",
			Color: "Orange",
		},
		{
			Age:   8,
			Kind:  "Dog",
			Breed: "Labradoodle",
			Name:  "Bruce",
			Color: "Golden",
		},
	}

	fmt.Println(animals)
}
```

## Method Sets

> Ref: https://go.dev/wiki/MethodSets

What is a method set?

- The method set of a type T is the group of all methods with a receiver T
- The method set of a type \*T is the group of all methods with a receiver T and \*T

```go
type Animal []string

func (a Animal) SuggestNames() []string{}

func (a *Animal) PetID() int{}

func main(){
pet1 := Animal       // method set contains SuggestNames()
pet2 := &Animal      // method set contains SuggestNames() and PetID()
}

```

## When to use a pointer receiver, when to use a value receiver

Use a pointer receiver when :

> - Your struct is heavy (otherwise, Go will make a copy of it)
> - You want to modify the receiver (for instance, you want to change the name field of a struct variable)
> - Your struct contains a synchronization primitive (like sync.Mutex) field. If you use a value receiver, it will also copy the mutex, making it useless and leading to synchronization errors.
> - When your other receivers are pointers

Use a value receiver when :

> - Your struct is small
> - You do not intend to modify the receiver
> - The receiver is a map, a func, a chan, a slice, a string, or an interface value (because internally it’s already a pointer)

# Interfaces

- An interface is like an abstract class in Python.
- `interface` is a type which defines abstract behavior via methods.
- An instance is said to have satisfied the contract (met a behavior) once it implements all the methods defined in the interface definition.

# Packages and Modules

### Downloading a Go module outside of a project

To download a go module outside of a Go project without compiling it use the following

```sh
go mod download github.com/openfga/openfga@latest
```

This downloads the module under $GOPATH/pkg/mod

```
go env GOMODCACHE
```
