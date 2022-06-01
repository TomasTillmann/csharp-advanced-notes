# Implicit / Explicit conversion
* everything inherits from object
```cs
// implicit conversion
string s = "string";
object o = s;
```
* string inherits from object => implicit conversion happens

```cs
// explicit conversion
int i = 0;
long l = 1;
l = (long)i;

// or

i = (int)l;
```
* int doesnt inherit from long => explicit conversion is required

```cs
// implicit conversion - but boxing
int i = 0;
object o = i;
```
* int inherits from object, but int is value type, and object is reference type
* boxing needs to be performed

## Boxing
* allocating value type on heap, instead of stack

```cs
object o;
// super slow
for(int i = 0; i < 1000; i++) {
    o = i;
}
```
* every time assign happens, new instance of value type on heap is allocated, with new reference pointing to it

## Explicit conversion - run time checks
```cs
// compiles
class A { }
class B : A { }

object o;
...

A a = (A)o;
```
* compiles, but run time error is possible
* depends on value of `o`
```cs
// runtime error
object o = 3;
A a = (A)3;
```
* class A and int are not in any inheriting relationship => no explicit conversion exists => run time error

```cs
object o = new B();
A a = (a)b;
```
* is okay
* there is no way how can c# distinguish between these two cases in compile time, hence run time error are the only option, since c# is type safe language

# Extension Methods
## Problem
* how to add new functionality to already existing types? (like string, double, or from some other framework ...)

```cs
// T is 3rd party/.NET class 

class Extensions {
    // not extension method
    public static newFunc(T t, int a) {
        ...
    }
}

// client
// still not the syntex we would like
Extensions.newFunc(t, 42);


// still not quite correct
class Extensions {
    // extension method
    public static newFunc(this T t, int a) {
        ...
    }
}

// this is exactly what we wanted
t.newFunc(42);
```

* extension method just turns on this syntax possibility, but in CIL code, it is written the same as in the first example `Extensions.newFunc(t,42)` instead of `t.newFunc(42);`

## How does compiler find extension methods?
* it needs to be in a static class

```cs
// finally correct extension method
static class Extensions {
    // extension method
    public static newFunc(this T t, int a) {
        ...
    }
}
```

* compiler first tries to find extension methods in current namespace
    * obviously prefers methods on the current instance, than it looks for other static classes

* then it looks to all namespaces added with `using`
* if ambigous, it doesnt compile
    * we can still explicitly choose using the worse syntax

## Remarks
### Access
* extension methods can  only use public API
    * cannot use implementation details, hence it can be very inefficient in some cases

### Inheritance vs extension methods
```cs
public class A {
    public int x;
    public A(int x) {
        this.x = x;
    }
}

public static class AExtensions {
    public static int CompareTo(this A first, A second) => first.x.CompareTo(second.x);
}

// A doesnt implement IComparable interface!

// but it fails at run time!!, not compile time
// without the extension method, it fails at run time too
IComparable<A> a = (IComparable<A>)new A(3);



// this works
new A().CompareTo(new A(5));
```

### mutable struct problems
```cs
public struct A {
    public int X;
    public int Y;

    public void Reset() {
        X = 0;
        Y = 0;
    }
}

// vs

public struct B {
    public int X;
    public int Y;
    
    // no reset
}

// but extension method Reset exists

public static class BExtensions {
    public static void Reset(this B b) {
        b.X = 0;
        b.Y = 0;
    }
}


// client code
A a = new A(1,2);
// pritns 1,2
a.Reset();
// prints 0,0 as expected

// but

B b = new B(1,2);
// prints 1,2
b.Reset();
// still prints 1,2 !!!
```

* reseting `b` reseted only the passed copied struct value in extension method Reset
* reseting `a` resets the reference, because it is implicitly passed as tracking reference
    * instance methods of struct implicitly take this as and tracking reference parameter

```cs
// from C# 7.2 this is possible
public static class BExtensions {
    // use ref in extension method too
    public static void Reset(this ref B b) {
        b.X = 0;
        b.Y = 0;
    }
}
```

* now it works as expected


## When to use
1. we would like to add new API to already existing class, that we cannot change

2. seperation of modules at complex software

3. use instead of hungarian notation

```cs
// somewhere in program
float speed;
float position;

// this is allowed, even though it semantically doesnt make sense
position = speed;

...

// better solution

struct Speed {
    public float Value;
}

struct Position {
    public float Value;
}

// this is now not allowed
Position position = new Position(3);
Speed speed = position; 

...

// we somewhere use some .NET math calculations, eg.

double res = Math.Sin(...);

// we would like to interpret res as speed for example
// this leads to this nice use of extension methods

Speed speed = Math.Sin(...).ToSpeed();

public static class DoubleExtensions {
    public static ToSpeed (this double d) => return new Speed(d);
}
```
