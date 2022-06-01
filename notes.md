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

// truncation
i = (int)l;

// note, that this is implicit conversion 
l = i;
```

* int doesnt inherit from long and no implicit conversion is defined => explicit conversion is required

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


List<A> list = new List<A>();
list.Append(new A(1)); list.Append(new A(2)); list.Append(new A(3));

// but this doesnt work - run time error
list.Sort();


// sort uses internally CompareTo, but it needs to be the interface implementation of CompareTo, not as an extension method
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
1. we would like to add new API to already existing class, that we cannot change the already existing class

2. seperation of modules at complex software

3. use compactly with "hungarian notation"

```cs
// somewhere in program
float speed;
float position;

// this is allowed, even though it semantically doesnt make sense
position = speed;

...

// better solution - "hungarian notation on steroids"

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

# Method Overloading
```cs
m(int i) {...}
m(long l) {...}

// knows what method to call
m(1);
```

* at CIL level, there are two completely independent methods
    * `m'int`
    * `m'long`

* at run time it is completely known what method to call, what method to call resolves at compile time

## How does compiler decide this
1. arrity
* number of parameters
* in context of type of the current instance, finds all methods with this name and and number of parameters
    * only methods **declared** in this context, not all accesible! (eg it doesnt look for parent methods)

```cs
// remark - context is instace's a type, not the intsance itself!
a.m(1);
```

* we now have list of all methods that passed the test above
* if no methods in this list, compiler goes straight to step 2
* compiler konw tries to find the best find by these rules
    1. the most specific
        * the "closest" in inheritance tree
    2. the least amount of "work" at run time
        * eg, preferes implicit conversion instead of boxing

### comlex example
```cs
class A {}
class B {}
class C {
    public static implicit operator B(C c) => 
        // converts C to B somehow ... 
        new B();
}

class Program {
    public static void f(B b) {
        // some work
    }

    public static void f(object o) {
        // some work
    }

    public static void Main(string[] args) {
        // what f will be choosed?
        f(new C());
    }
}
```
* context is class program

* both b and object makes perfect sense
* C# compiler will choose `f(B b)`
* the reason is, it is more specific
    * there is probably more functionality on `B` than or `C`'s parent

* so even though it costs more "work" (explicit call to implicit method conversion), specifity has bigger priority 

* also, the conversion is implicit, hence it makes sense to use it freely

* imagine `B` is person class and `C` is student class

2. jumps one context up and goes to step 1

### Another, maybe a little unintuitive example
```cs
class A {
    public void f(int i) {
        ...
    }
}
 
class B {
    public void f(double d) {
        ...
    }
}

// client code
B b = new B();

// what is called?
b.f(1);
```
* called is `f(double d)`
* context is `class B`, because `b` is of type `B`
* there is only one method f
* that's it
* compiler doesn't look one context up in this case, because it found mathing function, so he is happy and ends
* even though in the parent is `f(int i)`, compiler doesn't care

### Why compiler works this way?
* parent can live in some .dll
* in parent, there is not m(int i), when we implement in child m(double d)
* after some time, they add m(int i)
* if compiler would prefer this m(int i) from parent, it can do something we wouldn't want (we would like to still use m(double i)) 



## Remarks
* C# can make only one **user** implicit conversion
    * but it can make one user implicit conversion and then many implicit conversions (eg climbing up the hierarchy tree)
* in C, it can make more, but it is too complex for real life programs 
* we should be careful using conversions


# Generic methods
* adds functionality to some family of objects
* one implementation can be used by many objects (better than copy paste or huge switch)

```cs
T f<T>(T t1, T t2) {

}
```
* introduce by type parameters
    * not passed around on stack
    * information for compiler only

* from the perspective of compiler, generic methods with different type parameters are completely different methods

## How generic methods work in C
* they are called templates (template methods)
* every time template method is called with different type parameters, compiler recompiles the whole program
    * compiler needs to have access to source code of the template methods
* it is harder to distribute, binary of dll isnt sufficient (dynamic library for instance)

## How it works in C#
* prioritize accesibility and distribution of generic methods
* at CIL code level, the method is still generic, after compilation

### Client - CIL - machine code
* m<int>() - m'int (somehow notes that we want to call generic m specialized on int) - generated machine code from CIL method, speicialized on int

* this allows creating machine code of concrete specializations at run time, unlike in C, where machine code of concrete specializations has to be known before running
    * JIT vs AOT

## Duck Typing
* works in C and python with generic methods
* it means, there is no constraint for the type parameter
    * in C, because it recompiles every time we want to use concrete specialization, we get the error at compile time
* in C#, it compiles only once, that would lead to run time errors, hence, we need type constraints (duck typing could be used in C#, dynamic keyword)

## Type Constraints
```cs
public T Max<T>(T t1, T t2) where T : IComparable {
    if(t1.CompareTo(t2)) {
        return t1;
    }
    else {
        return t2;
    }
}
```

## Generic extension methods
* work the same as generic methods, becuase extension methods only bring "syntax sugar"


## Remarks
* we can call generic method without type parameters, than it tries to find the "best match"
    * similiar algorithm of finding the correct method overload    

* when choosing what overload between generic methods and methods to choose, compiler uses still the same algorithm, where generic method is less specific


# Generic Types
* defines a family of types

## How does it work
* works the same way as in generic methods
* at CIL code level, there is only one generic class
* whenever in source code is used (written eg A<int>, note that it doesnt have to be declaration or instantiation!) a concrete specialization of this generic class, the implementation (machine code of this class) is created (JIT) (in C, again, AOT, everything is always recompiled)

### Independence
* as in generic methods, generic types are completely independent of each other

```cs
public A<T> : C {

}

// client code
var a = new A<int>();
var b = new A<long>();

// not possible
b = a;

// there is no relationship between these two types, they are completely independednt of each other

// on the other hand,
C c = a;
// or
C c = b;

// is obviously okay
```

* static types belong to the concrete specialized type 
* also, static (class) constructors are always called for each specialization (when the first instance is created)


# What can be the constraints
```cs
public class A<T> where T : IComparable, ISeriazable {
    ...
}

// means that T must implement both interfaces
```

```cs
public class B<T> where T : new() {
    ...
        ... new T();
}

// T has parameterless constructor
```

* there is no way how to make T use constructor with parameters

```cs
public void m<T,U>(T t, U u) where T : A where U : T {
    ...
} 

// can constraint a parameter by another constraint
```

* where T : struct
    * value type

* where T : class
    * reference type









