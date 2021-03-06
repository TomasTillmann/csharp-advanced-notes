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
* C# can make only one implicit conversion
    * but it can make one implicit conversion (between two branches) and then many implicit conversions that are just climbing up the hierarchy tree
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

* its actually not that simple, if the parameter is value type, than this is true, but
    * if the parameter is reference type, then there is only one implmementation of the class, that specializes for all the possible reference constraints

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

### Impossible constraints
* there is no constraint, how to make T have some static method
* example
```cs
public struct Complex<T> {
    T real;
    T imaginary;

    public static operator +(Complex<T> t1, Complex<T> t2) =>
        // how to make T to have + operator?
        new Complex<T>(t1.real + t2.real, t1.imaginary + t2.imaginary);
}
```

* there is no way how to do it
* if double, float, etc ... implemented some interface IArithemitcs

```cs
public struct Complex<T> where T : IArithmetics {
    ...
}
```

* then our problem would be solved, but this interface doesnt exist (for now, it is in C# source code, but it is commented out)


# Interface concrete implementation of method 
```cs
public interface IRead {
    ...
    public void Close();
}

public interface IWrite {
    ...
    public void Close();
}

public class TCP : IRead, IWrite {
    // it's important, there is no public/private in front !
    void IReader.Close() {
        // implements IReader close 
    }

    void IWriter.Close() {
        // implements IWriter close
    }

    /// this is unnenecessary
    public void Close() {
        // implements both IWriter close and IReader close
        
        // this is wrong, recursion
        // this.Close();

        ((IWriter) this).Close();
        ((IReader) this).Close();
    }
}

// client code
public void writeAndClose(IWrite writer) {
    //writes somehow
    ...

    // closing
    writer.Close();

    // calls IWriter.Close(), so it only closes writer stream, which is exactly what we wanted
}


TCP connection = new TCP();
writeAndClose(connection);

// closes both streams
connection.Close();
``` 

## Combined with generics
```cs
interface I {
    void m(T x);
    T m ();
}
// A needs to implement both m methods taking int and long and returning int and long - how to do it?
class A : I<int>, I<long> {
    // the only way is to use explicit iterface implementation of methods

    public void m(int x) { ... }
    public void m(long x) { ... }

    public int m() { return 1; }
    // public long m() {return 2; } // not possible

    int I<long>.m() {return 2; }
}
```

# Covariance, Contravariance, Invariance
## Definition
* type C is parametrized by T
```cs
class C<T> {

}

// or
string[] array = {"a", "b", "c"};
// this array is parametrized by string
```

* type B is compatible with type A
```cs
A a = new B(); // is okay
```

### Covariance by T
* if both of the above holds and C<B> is compatible with C<A> (can assign C<B> instance to C<A>), then C is covariant by parameter T 

### Contravariance by T
* if both of the above hold and C<A> is compatible with C<B> (can assign C<A> instance to C<B>), then C is contravariant by parameter T

### Invariant by T
* is nor covariant or contravariant

## Covariance in C#
* array of reference types are covariant
* array of value types on the other hand are invariant

```cs
object[] a1 = {...};
string[] a2 = {...};

a1 = a2; // is legal! covariance

Console.Write(a1[0]); // reading is okay

a1[0] = "string"; // if writing the same type, it is still okay but ...
a1[0] = 1; // note boxing

// this is not okay, because if we for instance later in our code write this:

foreach(string str in a2) {
    // do something
    // str from position a2[0] is int, not string, hence a1[0] = 1; results in run time error
}
```

### Conclusion
* read is okay
* write is not
* every write access to array costs at least one if (run time check, whether covariance wasnt abused) (in C it doesnt happen)
    * program can be very slow because of this
    * how to make it faster?
        1. make it sealed - that kills any possibility of covariance    
        2. extremely dirty trick 

```cs
// declare this struct
struct X<T> where T : class {
    private T t;

    public static implicit operator X<T>(T t) {
        X<T> x;
        x.t = t;
        return x;
    }
    
    public static implicit operator T(X<T> x) =>
        x.t
}

// this allows freely to convert from our class to this struct parametrized by our class

// client code
// dont want to make this sealed
public class A {
    ...
}

// instead of iterating array of A, where the run time check occurs, convert it to value type error, this is invariant, hence run time check doesnt happen
```

## Contravariance in C#
* array of value types is contravariant
    * value can be anything, but reference has still the same structure
* read is not okay
* write is okay

### Why?
* imagine this compiles
```cs
object[] o = {...};
string[] s = {...};

// imagine arrays are contravariant
s = o;
```

* we can write only string to this array (or children, but string is sealed - no children) 
* hence write is okay, because object can work with strings - there is implicit conversion
* on the other hand, we cannot read, because we might get something "better" than string, something that is an ancestor of object


## Interfaces are variant
* that means, we can set each parameter to define covariance, contravariance or invariance

```cs
// covariant, invariant, contravariant, contravariant
interface I<out T1, T2, in T3, in T4> {

}

I<A,B,C,D> i = new X();

// what can X be?

class X : I<E,F,G,H> {

}
```
* where:
    * E is child of A
    * F is equal B
    * G is parent of C
    * H is parent of D

* the reason is, that somewhere in I, there is method that returns T1 (getter), some method that has as input parameter T3 or T4 (setters) and some method that could have both input and return type set to T2

### Examples
1. IList<T>
    * has to be invariant, because it allows for read/write

2. IReadOnlyList<out T>
    * only read is performed, hence is covariant 


# Interfaces
## Differences between abstract classes
### interface
* can implement multiple interfaces
    * doesnt have any data
    * inheriting multiple methods is not a problem
* define contract
* extensibility is optional
    * opt-in
        * virtual / abstract ...

### Abstract class
* doesnt support multiple inheritance
    * diamond problem
    * problem with inheriting data
* define contract
* promise extensibility in future
    * opt-out
        * sealed override

## When what to use?
* use abstract class when data is needed
* use abstract class for non public contract
    * protected, private, internal
* use interface for multiple inheritance


## Virtual methods table
* is filled at compile time
* each class has its own table
* instance has reference to its type (GetType())
* this type has reference to its class virtual methods table
* `virtual` makes new record in this table
* `override` overrides this record with new implementation

## Interface methods table
* class remembers it whenever it implements some interface
* for the interface, it is empty
* the reference doesnt point to concrete methods machine code, by rather on the method istelf (double dereference)

* eg 
```cs
interface I {
    public void m();
    public void l();
}

public class A : I {
    public virtual void m() {
        ...
    }

    public void l() {
        ...
    }
}

```

* interface table for `A` has record for m's implementation pointing to virtual methods table (m is virtual) of A
* l's implementation is pointing to normal list of methdos of A

```cs
public class B : A {
    public override void m() {
        ...
    } 
}
```
* B's virtual methods table's record for m is now overriden 

```cs
A a = new B();

// what gets called?
a.m();

// override in B - easy

A a = new A();
a.m();

// virtual function living in A

// we havent need interface method table in these examples

// how about now?
I x = new A(); 
x.m();

// it just depends on the instance, hence the virtual version of m is executed
```

* now interface method table is necessary
* JIT checks interface method table for class A
    * for class A, becuase x is instance of A (reference is in type (GetType()))
* in this table, he finds, that m is living in A as a virtual method (in VMT), finds its machine code (or generates it) and executes the function
* double reference

```cs
A a = new B();
I x = a;
// what happens now?
x.m();
```

* interface methods are inherited, that means, it is filled the same way as in the parent

* when the interface is implemented once again in some child, it is not inherited, but the interface method table is filled once again

```cs
// if B was implemented like this
// no change, table is filled the same way as in parent
class B : A, I {
    public override void m() {
        ...
    }
}

// still no change, l was already implemented at A
class B : A {
    public override void m() {
        ...
    }

    public void l() {

    }
}

I x = new B();
x.l();      // calls A method l


// interface method table is different now, because l is new
class B : A, I {
    public override void m() {
        ...
    }

    public void l() {
        ...
    }
}

I x = new B();
x.l(); // calls B method l
```

### Summary
* if not interface handler
    * get instance type and there is reference on:
        * normal methods list
        * virtual methods table
    * finds record of called method either in VMT or normal methods list
    * execute

* if interface handler
    * get instance type and it's interface methods table
    * there is reference on method (not machine code) that implements it
        * can be either to VMT or normal methods list
    * execute

* interface method table is inherited (if not explicit implementation in child)
    * it is the same for child as for parent

* interface method table is initialized (filled) every time we implement the coresponding interface

# Delegates
* reference type pointing to a function
* to what function the delegate can point is restrained by its definition
* its just a shorthand (syntactic sugar)
```cs
delegate int D(string a);

// is shorthand for (not really but this is the idea)

class D : MultiCastDelegate {
    public int Invoke(string a) {
        ...
    }
}
```

* delegates are allocated on heap

```cs
D d = new D(a.m);

// where m is function that obeys constraints of type D


d.Invoke("string");
// is the same as
d("string");
```

### Data in delegate
* remembers pointer to "this" (context of the method) and the method itself
* if the method is static, then "this" is null
* is immutable
    * nice for parallel programming too

```cs
D d = new D(a.m);

// not allowed - cannot change the context, because it's immutable
d.this = b;
d();
```

* what method to choose is decided at compile time not run time
* for example, when we assign method from VMT to the delegate, the process of finding the correct implementation is done at compile time, not run time
* then, when we call the delegate, then we just call the function that the delegate remembers on the remembered context

### Syntactic sugar time
```cs
public void m() {
    ...
}

// if m is null, does not do anything, if it is not, than it calls m
a?.m();

// if the method wasnt assigned to the delegate, than it is very useful
d?.Invoke();

d = new D(m);
d.Invoke();
```

## Why Multicast Delegate?
* can chain delegates together

```cs
// returns new delegate (immutable)
D d = Delegate.Combine(d1, d2);

// calls both d1.Invoke() and d2.Invoke(), in this order!
d.Invoke();

// there is a little parallel with chain of responsibility design pattern
```

* MulticastDelegate works as a stack
```cs
d.Remove();
// removes the last added
```
* can be syntactically shortened
```cs
// p = Delegate.Combine(p, a1.f);
p += a1.f;
p += a2.f;

// p.Remove(new D(a1.f))
p -= new D(a1.f);
p -= new D(a2.f);
```

## Combine with generic delegates
* in .NET, there are predefined delegates Action and Func 
* many overloads
* Func<T1, T2, ..., TN, TResult>(T1 t1, T2 t2, .. Tn tn)
* Action<T1, T2, ...>(T1 t1, T2 t2, ...)

## Anonymous method
```cs
List list = new List();
int sum = 0;

// anonymous method
list.ForAll(delegate (Node p) {sum += p.value; });
```

* just points to a method that is declared "somewhere else"

# Type inference
* compiler somehow needs to guess what the delegate is going to return
* always at compile time!
```cs
delegate (int a, int b) {return a + b; }
```

```cs
Func<int,int> f1 = x => x + 1;
Func<int,double> f2 = x => x + 1;
Func<double,int> f3 = x => x + 1; // error, implicit onversion from double to int doesnt exist
```

# Lambda functions
* syntactis shorthand for delegate ( + more)
```cs
(int a, int b) => a + b;
```
* can be only passed to delegates
* it's also an anonymous function

```cs
(int a) => {int b = 5; return a + b + c; }
```

* c is free/open variable (volna promena)

## Closure
* lambda function with no free variables
* c# supports only closures
    * unlike other languages, that have functions as first class entities

* that means, c# needs to make capture of the free variable

## Capture
## By Value
* in C
    * capture by value
    * `[=](...) { ... }`

* no problems

## By Reference
* in C
    * capture by reference
    * `[](...) { ... }`

* sometimes, we need to make a life of free variable longer than one run of the function
    * eg `list.ForEach((int a) => {sum += a; });`
    * sum needs to be somewhere else

* in C, it is very unsafe
* the lambda function can be returned, hence the life of this lambda function is longer than the life of a function it was called/declared in!
* this captured by reference variable then, if it lived in that function is gone (its lifespan has ended) and that memory can be occupied by something else => very unsafe 


* in c#, always capture by reference

## How does it work
* capture by reference only
* free variables can only be local variables (variables in current method scope)
    * even if it is a field/property of this instance (local variable this)
```cs
public class A {
    private int x;
    private int y;

    public Action<int> f(int z) {
        int a = 3;
        return () => x + y + z + a;
    }

    // is on JIT leven translated to
    public Action<int> f(A this, int z) {
        int a = 3;
        return () => this.x + this.y + z + a;
    }

    // and 'this' is than just a local variable
}

// at JIT level (conceptually)

// it is not called Scope but something random (anonymous class)
public class Scope {
    public A thisA;
    public int a;
    public int z;

    // lambda method
    public void Lambda() {
        return thisA.x + thisA.y + z + this.a;
    }
}

// and the f funciton
public Action<int> f(int z) {
    Scope s = new Scope();
    s.z = z;
    s.a = 3; 

    Action<int> d = new Action<int>(s.Lambda); 
    return d;
}
```

* this way, lifespan of captured variables is prolonged
* so even if return the delegate the life of captured variables isnt over
    * this solution doesnt result in unsafe memory access (like in C)

## Remarks
* lambda function captures variables, not names!

```cs
for (int i = 0; i < 10; i++){
    int j = i;
    for (int x = j < 20; x++) {
        int a = x;
        ...
    }
    ...
}
```

* few names, but a lot of different variables
* each current scope has one Scope class

# Private / internal interfaces
```cs
public I1 {
    void m();
}

internal I2 {
    void f();
}

public class A : I1, I2 {
    public void m() {
        ...
    }

    public void f() {
        ...
    }
}

// or, if dont want f() as a public API

public class A : I1, I2 {
    public void m() {
        ...
    }

    void I2.f() {
        ...
    }
}

// and now only those who see I2 can access f() 
```

# Namespaces
* module to put things together
* only a shortcut

```cs
namespace A {
    namespace B {
        class C {

        }
    }
}
```

* its just shortcut for
```cs
A.B.C
```

* it's just syntactic sugar

## Current context
```cs
namespace A {

    namespace B {
        class C {
            // current context of C is B
        }

        class D {
            // current context of D is B
        }

        class E {
            public void f() {
                G g = new G();
            }
        }
    }

    class G {
        // this gets called in f
    }

class Program {
    static void main(string[] args) {
        // current context is A
    }
}
```

* current context is the closest namespace
* searching of some entity (namespace, class, struct) goes from current context, to higher context and so on to root

## Global context
* `global::...` is default root
* Aliases : Global by default
    * you can unset

## Using
* imports types not namespaces!

# Nested Types
* .NET knows about this hierarchy, unlike in namespaces

```cs
namespace N {
    class A {
        class B {

        }
    }
}
```

* on CLR level, name of B is
    * N.A + B
    * ToString() calls would yield such a result

* but as a programmer, you would access B as N.A.B

* in JAVA, B would have a reference for A, but in C# this doesnt hold at all

## When to use
* use when B is internal detail of A

```cs
public class A {

}

internal class B {

}
```
* still can be seen in current assembly

```cs
public class A {
    private class B {

    }
}
```

* nobody can see this class
* eg when B is implementation detail of A (A is some data structure that uses red-black tree, B is node of this tree) or enumerator 

* Access is given syntactically

# Private contract
```cs
public class C {
    // this defines the private contract between C and D
    private interface I3 {
        void g();
    }

    // I1 can be some public interface
    public class D : I1, I3 {
        // I1 method
        public void f() {
            ...
        }

        // private contract implementation
        void I3.g() {
            ...
        }
    }

    ...
    public void usePrivateContractMethodInternally() {
        I3 x = new C.D();
        
        // g() can be called only by those who can access I3 iterface, but this is only class C
        x.g();
    }
}
```

# Enumerator
* IEnumerable interface
    * implements enumerator method
    * and many more

* type that implements IEnumerable interface can be enumerated
```cs
IEnumerable enumerable;
foreach(var x in enumerable) {
    // much easier access than using MoveNext() etc ...
}
```

# Yield return
* compiler generates state machine of enumerator implementation

# Anonymous class
* used by lambda functions for scope classes
* its just a syntactit shorthand

* this is not anonymous function but it is similiar
```cs
X x = new X{a = 5};

// is at CIL level

X x = new X();
x.a = 5;
```

* this is an example of anonymous function
```cs
new {x = alpha, y = beta};

// is translated to

class Anon1 {
    // type inference (___ is the infered type from alpha and beta)
    public ___ x;
    public ___ y;
}

new Anon1{x = alpha, y = beta};
```

```cs
// what to write on the left side though?
new {x = alpha, y = beta};

```
* we dont know the name of anon class

```cs
// the only way - type inference again
var anon = new {x = alpha, y = beta};
```

```cs
var anon1 = new {x = alpha, y = beta};
var anon2 = new {y = gama, x = delta};

// anon1 and anon2 are different types
```

* whenever we create intasnce of new anonymous class, compiler looks if there is already existing anonymous class with same type and same ordering of assignment 

```cs
var anon1 = new {x = alpha, y = beta};
var anon2 = new {x = gama, y = delta};

// if alpha and gama are the same type and beta and delta are the same type, anon1 and anon2 are the same anon function - same type

// we can do this then

anon1 = anon2;
```

* implements ToString() very nicely
* implements Equals() very nicely
    * structural comparision
* doesnt implement IEquitable<T>, thats not so nice
    * `anon1Instance == anon2instance` cannot be used, only 
    `anon1instance.Equals(anon2Instance)`

```cs
// this is also possible
var a = new {this.GetName(), this.GetAge()};
```
* compiler tries to deduce some nice names for its properties

## What are they for?
* only local variable (var - type inference)
* cannot be a property or return type or input of a function
* but!
```cs
public void f<T>(T a) {
    ...
}

// works!!! - type inference
f(new {x = 5});

// but its completely useless
// we can only work with it as object in function f, which is useless ...
```
* so it makes sense to use anonymous types only as local variables
* use when you want some n-tuple of values

# Tuple
## Tuple<T1, T2, ...>
* its a class though, thats a huge overhead
* dont use this

## ValueTuple, ValueTuple<T1, T2, ...>
* struct
```cs
var a = (1,2,3);        // this is ValueTuple

// this is ValueTuple
public void f((int, int, int) a) {

}

// this is also ValueTupple
public void g((int a, int b) x) {
    x.a = //

    // much nicer than
    x.Item1 = //
}

// how does CLI know about those names? He keeps attributes around this function, that define the names
```
* this is good


## What are they used for
* its mainly used when you want to return many values

```cs
public (int x, int y) f() {
    return (1,2);
}

// type inference very nicely
var a = f();
print(a.x);    // not Item1
print(a.y);

// or
// deconstruction
var (x,y) = f();
print(x);
print(y);

// overwrite of x and y
(x,y) = f();
print(x);
print(y);

// if we dont care about x for example
(_, var y) = f();
```

### Deconstructor
```cs
public A {
    public firstName;
    public lastName;

    public void Deconstruct(out string firstName, out string lastName) {
        firstName = this.firstName;
        lastName = this.lastName;
    }
}

var a = new A{"Johny", "Lawrence"};

// deconstruction, only calls Deconstruct method
(string firstName, string lastName) = a;
```

* so ValueTuple doesnt use any black magic, it just has this method Deconstruct(...) which is then called

# LINQ
* used as a query language on our data structures
* query language
    * maps on function calls of LINQ library

* extension methods
* lazy evaluation - IEnumerable

* the query doesnt actually do anything, it just creates an instance of WhereEnumerable
    * data structure that describes the query
    * no computation is done yet!

* this instance remembers:
    1. the lambda query
    2. data source

* another part of the query is another instance, this remembers again:
    1. the lambda query
    2. data source, the result of the first instance

* example
```cs
q = x.Where(p1).Where(p2).Select(x => x + 1);
```

* (x, p1) <- (Where1, p2) <- (Where2, x => x + 1) <- q
* each node in this chain implements IEnumerable, so we can iterate over it
* the computation happens only when we iterate over it
    * maximum laziness

```cs
// only now computation happens !
foreach(var x in q) {
    ...
    // GetEnumerator()

    // iteratively
    // MoveNext()
    // Current()
    // this recursively calls GetEnumerator() MoveNext() and Current() in this chain
}
```
* cannot be always lazy
* eg, OrderBy() cannot be lazy
* this means, that different queries / different orders can have different efficiency eg OrderBy() can be super slow

# Threads
* each thread has its own stack
* data on stack are only seen by this thread
* data on heap can be shared by many different threads
    * here all the problems arise (deadlock, race condition, ...)

* the only reason why we would like to use threads is in order to improve performance

```cs
using System.Threading.Thread

// can pass a delegate there
var t1 = new Thread(...);

// start the thread - doesnt actually mean that OS will at this instance make this thread running
t1.Start();


// this current thread now waits for t1 to finish (passive waiting)
t1.Join();
```
