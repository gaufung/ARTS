---
date: 2010-09-10
status: draft
title: Digest in Effective C#
---

# 1 Use Properties Instead of Accessible Data Members

Property is first-class citizen in the C# lanauge. You can verify value to set data member. 

```CSharp
public class Person
{
    private string name;
    public string Name
    {
        get {return name;}
        set
        {
            if(string.IsNullOrEmpty(value))
                throw new ArgumentException("name cannot be blank");
            name = value;
        }
    }
}
```

Property also supports the feature of methods, such as *virtual*.

```csharp
public class Person
{
    public virtual string Name 
    {
        get;
        set;
    }
}
```
Besides, you can also specify different accessiblity modifier to *get* and *set* accessor in the property.

```csharp
public class Person
{
    public virtual string Name
    {
        get;
        protected set;
    }
}
```

Property syntax has extended beyond simple data fields. It can be treated as indexer. 

```csharp
public int this[int index]
{
    get { return thisValue[index]; }
    set { thisvalue[index] = value; }
}

public Address this[string name]
{
    get { return addressValues[name]; }
    set { addressValues[name] = value; }
}

public int this[int x, int y]
{
    get { return ComputerValue(x, y); }
}
```


# 2 Prefer `readonly` to `const`

Constants in C# can be divided into two categries:

- compile-time: it can be declared inside methods and used only for primitive types.
- runtime: it cannot be declared inside methods but it can be any type.

*readonly* values are for instance constrants, storing different value for each instance of class type. However Compile-tiem constants are for static constants. 

Note: Using const has performance advantages than readonly. 


# 3 Understand the Relationship Among the Many Different Concepts of Equality

C# provides four different funcitons that determine whether two different objects are "equal":

```csharp
public static bool ReferenceEquals(object left, object right);
public static bool Equals(object left, object right);
public virtual bool Equals(object right);
public static bool operator == (MyClass left, MyClass right);
```

Mostly, you override the third `Equals` method in your creating instance. Occasionally, overide the operator `==()` for performance consideration. Types that override `Eqauls` should implement the `IEquatable<T>`. 

- `Object.ReferenceEqual()` returns true if two variable refer to the same object (same object identiy), no matter what they are value type or reference type. It means that it will return false when you test equality for value type, Even when you compare a value type itself.

```csharp
int i = 5;
if(Object.ReferenceEquals(i, i))
    Console.WriteLine("Never happens.");
else
    Console.WriteLine("Always happends");
```

- `Object.Equals()` to test whether two variables are equal when you don't know the runtime type of the two arguments. Remmeber that it will implement by the `object` instance `Equals` method finally.

```csharp
public static bool Equals(object left, object right)
{
    if(Object.ReferenceEquals(left, right))
        return true;
    if(Object.ReferenceEquals(left, null) || Object.ReferenceEquals(right, null))
        return false;
    return left.Equals(right);
}
```

- Instance `Equals()` method. The default behavior is exactly the same as `Object.ReferenceEquals()`. But `ValueType` override the `Object.Equals()` but it's not efficient implementation (Reflection). So you must write a much faster override of `Equals()` for any value type. For reference type, if you want semantic instead of reference semantics for equality, override this method. 

```CSharp
public class Foo : IEquatable<Foo>
{
    public override bool Equals(obejct right)
    {
        if(object.ReferenceEquals(right, null))
            return false;
        if(object.ReferenceEquals(this, right))
            return true;
        if(this.GetType() != right.GetType())
            return false;
        
        return this.Eqlas(right as Foo);
    }

    public bool Equals(Foo other)
    {
        // elided
        return true;
    }
}
```

- `operator==()`: Anytime you create a value type, redefine `operator==()`. 


# 4 Rules for `GetHashCode()`

1. If two objects are equal, they must generate the same hash value.
2. For any object A, A.GetHashCode() must be an instance invariant. No matter what methods are called on A. 
3. The hash function should generate a random distribute among all integer for all input.


# 5 Use Optional Parameters to Minimize Method Overides.

Named parameters mean that in any API with default parameters, you only need to specify those parameters you intend to use.

# 5 Understand the Attraction of Small Functions

Instead of JITing your entire application when it starts, the CLR invokes the JITer on a funciton-by-function basis.

```csharp
public string BuildMsg(bool takeFirstPath)
{
    StringBuilder msg = new StringBuilder();
    if(takeFirstPath)
    {
        msg.Append("something");
        msg.Append("\nThis is a problem");
        msg.Append("imagine much more text");
    }
    else
    {
        msg.Append("this path is not so bad");
        msg.Append("\nThis is a problem");
        msg.Append("imagine much more text");
    }
    return msg.ToString();
}
```
When the BuildMsg get called, both path are JITed, but only one is needed. You can change it this way.

```csharp
public string BuildMsg2(bool takeFirstPath)
{
    if(takeFirstPath)
    {
        return FirstPath();
    }
    else
    {
        return SecondPath();
    }
} 
```
Small functions mean that the JIT compiler compiles the logic that's needed, not lengthy sequences of code that won't be used immediately.

Note:
The C# compiler generates the IL for echa method, and the JIT compiler translates that IL into the machine code on the destination machine.


# 6 Prefer Member Initializer to Assignment Statements

Constructing member variable when you declare that variable is natural in C#

```csharp
public class MyClass
{
    private List<string> labels = new List<string>();
}
```

# 7 Use Proper Initialization for Static Class Member

```csharp
public class MySingleton
{
    private static readonly MySingleton theOneAndOnly;

    static MySingleton
    {
        try
        {
            theOneAndOnly = new MySingleton();
        }
        catch
        {
            // elided
        }
       
    }

    public static MySingleton TheOnly
    {
        get { return theOneAndOnly; }
    }

    private MySingleton()
    {

    }
}
```
The CLR calls your static constructor automatically before your type is first accessed in an application space(an AppDomain). Using static constructor helps you handle unexpected exceptions.


 # 8 Minimize Duplicate Initialization Logic

 Constructor initializier allow one constructor to call another constructor.

 ```csharp
 public class MyClass
 {
     private List<ImportantData> coll;

     private string name;

     public MyClass() : this(0, "")
     {

     }

     public MyClass(int initialCount):this(initialCount, string.Empty)
     {

     }

     public MyClass(int initialCount, string name)
     {
         coll = (initialCount > 0) ? new List<ImportantData>(initialCount): new List<ImportatnData>();
         this.name = name;
     }
 }
 ```

# 9 Avoid Creating Unnecessary Objects.

All reference types, even local variables are allocated on the heap. Every local variable of a reference type becomes garbage as soon as that function or method exits.

```csharp
protected override void OnPaint(PaintEventArgs e)
{
    using(Font myFont = new Font("Arial", 10.0f))
    {
        e.Graphics.DrawString(DateTime.Now.ToString(), MyFont, Brushes.Black, new PointF(0,0));
    }
    base.OnPaint(e);
}
```

`OnPaint()` gets called frequently. Every time it gets called, you create another Font object that contains the exact settings. You should promote the Font object from a local variable to member variable.

```csharp
private readonly Font myFont = new Font("Arial", 10.0f);

/// elied
```
 
Creating static member variable for commonly used used instance of the reference type you need is best practice.


# 10 Implement the Standard Dispose Pattern

The implementation of your `IDisposable.Dispose()` method is responsible for four tasks.

1. Freeing the unmanaged resources;
2. Freeing all managed resources(including unhooking events);
3. Settings a state flag to indicate the object has been disposed. 
4. Supressing finalization. You can call `GC.SuppressFinalize(this)` to accomplish this task.


# 11 Disinguish Between Value Types and Reference Types

- Value types are not polymorphic. They are better suited to storing the data that your application manipulates 
- Reference types can be polymorphic and should be used to define the behavior of your application.

```csharp
MyType[] arrayOfTypes = new MyType[100];
```
If `MyType` is value type, one allocation 100 times the size of `MyType` object occurs. However if MyType is reference type, one allocation just occurs and every element of the array is null. When you initialize each elements in the array, you will have performaned 101 allocations. 


# 12 Make Sure That 0 Is a Valid State for Value Types

All `enums` are derived from `System.ValueType` 

```csharp
public enum Planet
{
    Mercury = 1,
    Vernus = 2,
    Earth = 3,
    Mars = 4
}
```
If `Planet sphere = new Planet()`, what does `sphere` means? 0 is not valid value.

```csharp
public enum Planet
{
    None = 0,
    Mercury = 1,
    Earch = 3,
    //elided
}
```

