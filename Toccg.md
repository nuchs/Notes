# The Opinionated CSharp Coding Guidelines

## Introduction

All non trivial software is complex, and the larger the piece software the more
complex it tends to be. At a certain point software can become so complex that
no-one understands what it's doing anymore. 

At this point a program will soldier on until it breaks (as no one is able to
work out how to fix it) or the users require it do something new (as no is able to work
out how to change it).

Even in less extreme cases software is often abandoned because it takes so much
effort to understand or update that the maintainers feel it would be simpler to
create a new system.

Common problems include

- The code is bloated, the user (in this case the maintainer of the code) is
  required to search through large swathes of code, much of which may be
  unused, to try and find the part that does what they are interested in.
- There are unnecessary dependencies beteen different parts of the system, it
  becomes impossible to change one component without another being affected.
- The code does not clearly express its intent, it was written with the
  assumption that 'non-one will ever need to change this' and so is filled with
  obscurely named methods and variables which made sense at the time.

This situation is not inevitable.

There are many principles that can be followed or techniques that can be used
to help manage complexity and prevent code from rotting. This guide provides a
high level overview of a number of them.

It also makes the following assumptions

  - That code is read far more often than it is written 
  - That the requirements for a project always change
  - That the next person who has to understand what you've written is a trigger happy pyschopath with a shotgun and your home address <2>

It therefore recommends a style of coding which emphasizes ease of change and
readability over other factors such as speed of writing or [performance](#Premature optimisation  == Math.Sqrt(evil);)

## Principles

Software development is not a precise science and there are no rules which will
take you to the correct solution, there are however a number of principles 
which can help you avoid making mistakes made by others who have gone before 
you.

It's worth noting that these are *principles* and not rules thus you need to
exercise judgment in deciding when to apply them. That said they hold for most
cases so if you decide to go against their recommendations make sure you
understand the tradeoff you are making.

### SOLID <3>

The SOLID principles for object oriented design

**Single Responsibility Principle**

A class should have a single responsibility (equivalently a class should only
have one reason to change)
Conjuctive

**Open/Closed Principle**

Software entities should be open for extension but closed for modification.

I have to admit I was tempted to omit this one as I've never fully got to
grips with it but SLID doesn't seem as memorable as SOLID so here it is.

My best understanding of this is that it suggests that you shoudl strive to
keep published interfaces (small I) stable, even when you are adding new 
functionality <4>. This may not directly relate to readability but is good
advice non the less.

**Liskov Substitution Principle**

Intuatively this principle states that calling code shouldn't have to care if
its operating on a particular class or a descendent of that class. This is a
stronger rule than the typical subtyping rules enforced by the compiler e.g.

```csharp
class Rectangle
{
  public void GetWidth ()
  {
    return width;
  }

  public void GetHeight ()
  {
    return height;
  }

  public virtual void SetWidth (int newWidth)
  {
    width = newWidth;
  }

  public virtual void SetHeight (int newHeight)
  {
    height = newHeight;
  }

  public int Area()
  {
    return Width * Height;
  }

  private int height;
  private int width;
}

class Square : Rectangle
{
  public override void SetWidth (int newWidth)
  {
    width = newWidth;
    height = width;
  }

  public override void SetHeight (int newHeight)
  {
    height = newHeight;
    width = height;
  }
}

// Meanwhile...

public void DoubleArea (Rectangle rectangle)
{
  rectangle.SetWidth(2 * rectangle.GetWidth);
}
```

This example will compile fine but will violate the LSP. If a Square with a
side of length 1 is passed into the DoubleArea method then its new area will
quadroupled rather than doubled. 

To avoid falling foul of situations like this the LSP states that:

1. Child classes must provide the same interface as their parents (This one tends to be easy as the compiler will enforce it)
2. Child classes must not throw any new types of exceptions unless those exceptions are subtypes of exceptions thrown by the parent
3. Child classes cannot strengthen any preconditions for use (otherwise there will be instances where the parent can be used by the child cannot)
4. Child classes cannot weaken any preconditions (otherwise there will be cases
   where the child can take the system to a state which would be invalid for
   the parent)
5. The parents invarients must be preserved by the child
6. A child cannot provide a way to break the parents encapulation

When designing a class hierarchy, if you can't meet these conditions then you
need to question whether inheritance is the right tool for the job.

**Interface Segregation Principle**

A client should not be forced to depend on methods that it doesn't use.

```csharp
interface IThingDoer
{
  void PrintDocument(Docuemnt doc);
  void LayBet(Bet bet);
  void CleanDishes(List<Dish> dishes);
}

public class PrintQueue
{
  public PrintQueue (IThingDoer printer)
  {
    this.printer = printer;
  }

  public AddItem (Document doc)
  {
    queue.Add(doc);
  }

  public ProcessQueue()
  {
    foreach (doc in queue)
    {
      printer.PrintDocument(doc);
    }
  }

  private List<Document> queue = new List<Document>();
  private IThingDoer printer;
}
```

In the above example the IThingDoer clearly violates this principle, the
PrintQueue is now coupled (albeit loosly) to the Bet and Dish classes. This
means changes which should not affect this class in the slightest will now
require it to be rebuilt. In the case of a small class this might seem trivial
but this can (and does) build up and will create situations where libraries are
having to be redeployed even though they have functionaly not changed.

On a simpler note it also means that the interface takes longer to read and you
can't build up a coherant picture of what it's purpose is because there isn't
one.

When designing an interface (small i) decide on what role an object of that
interface is supposed to play and only include methods appropriate to that
role. To take the above example you would split IThingDoer into three.

```csharp
interface IPrinter
{
  void PrintDocument(Docuemnt doc);
}

interface IBetter
{
  void LayBet(Bet bet);
}

interface IDishWasher
{
  void CleanDishes(List<Dish> dishes);
}
```

**Dependency Inversion Principle**

>A. High-level modules should not depend on low-level modules. Both should depend on abstractions.
>B. Abstractions should not depend on details. Details should depend on abstractions.

Pictorially the first part can be thought of as moving from this

Higher Layer ----> Lower Layer

to this

Higher Layer ----> Interface <---- Lower Layer

This means that both layers are insulated from each other and can change
independently as long as the interface remains stable.

The second part is effectivly saying what is allowed to go in to the interface,
that is nothing which belongs to either layer. If you do that then you've
reintroduced a dependency between the layers.

In practical terms you would normally define the interface as part of the
higher level module to begin with as this helps hide the implementation details
for the layer. If you need to reuse the lower layer then the interface can be
moved out into its own module and both layers will depend on that.

###YAGNI (You ain't gonna need it)

This principle states that you shouldn't implement something until you actually
need it.

The rationale behind this is that it is hard to guess what features will be
needed in the future and how they should be built. Even if you do get it
right you have spent time developing a feature which isn't required just now at
the expense of developing a feature which is. Finally you've increased the
complexity of your code without any short term gain (as the feature is not
required at the moment) meaning that it will be harder to change and to
understand.


http://c2.com/cgi/wiki?YouArentGonnaNeedIt
http://martinfowler.com/bliki/Yagni.html


###DRY (Don't repeat yourself)

> Every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

The Pragmatic Programmer

When this is true code is easier to change bec

###DAMP (Descriptive And Meaningful Phrases)
###Tell don't ask (Or the Law of Demeter)

###Principle of least surprise (or astonishment)

Simply put, a system should behave in the way its users expect e.g. if you 
drag a file into the wastepaper on your desktop the operating system should 
delete the file and not email it to a national newspaper.

Although most people encounter this principle in the context of UI design it is
just as applicable when writing code. 

```csharp
void DoSomething (IDictionary<string, int> aDictionary)
{
  // Do some important work ...
  PrintValue(aDictionary, "icecream");
}
```

My expectation when I look at this code is that this method does some important
work and then prints out the value assoicated with the key "icecream" in the
provided dictionary.

If it turns out that the implementation of PrintValue is actually:

```csharp
void PrintValue (IDictionary<string, int> aDictionary, string key)
{
  Console.WriteLine("The value of " + key + " is " + aDictionary[key]);  
  aDictionary.Remove(key);
}
```

Then my expectation has been confounded and I am going to have a very hard time
working out where all the values in my dictionary are going.

Two principles which support this one are

- The [SRP] (Single Responsibility Principle), if a class or method only does one thing it is much less likely that it will be something unexpected
- [Expressive Naming] means that you should be able understand what a class or method does just from the name

###Expressive Naming


###Premature optimisation == Math.Sqrt(evil);

Optimising code is a trade off, you alter the code and sacrifice some aspect of
its design in order to gain an increase in performance (either in time or
space). For it to be beneficial the performance gain must be worth more than
whatever was sacrificed.

The most common sacrifice made is to increase the complexity of the code,
consequently making it harder to understand and to change.

Before making *any* optimisation, always, always measure. Programmers as a
group tend to be very bad at when it comes to determining what code needs
optimised and wil tend to apply micro optiomisations whose performance gain is
ouweighed by the design cost hence Knuth's famous quote 'Premature Optimisation is
the root of evil'<add footnote saying you should read the full thing as the
quote looses something without the context>

A general stratgy for optimising code is

1. Is the current performance sufficient? If so stop.
2. Measure the systems's performance and Identify the biggest bottleneck
3. Optmise the code to reduce the impact of the bottleneck identified in step 2.
4. Goto step 1

### CQS (Command Query Seperation)

Command query separation is the idea that each method call should either be

1. A command that performs an action or
2. A query which returns a value

This style naturally falls out of the [SRP](Single Responsibility Principle) as
methods should only be responsible for one thing. Following it helps avoid
violations of the [Principle of least surprise] as a query should never have
unxpected side effects (otherwise it wouldn't be a query) which in turn makes
the code easier to reason about.

###Immutability
Value types
pure functions idempotent

###Inheritance
    prefer composition
    design for it or prohibit
###Security
  Validate input at trust boundries
  white/black list
  clense input
  key management -its hard
##Techniques
  Refactoring
  Unit Testing
    Triple A
    mocks
    test first
    don't cross network/file system/database
    Mandatory before commit
    assert last
    assert once
    sut
    fast
    repeatable
  TDD
  Pair Programming
  Continuous Integration
  Code Reviews
##Code Smells
Code Smells
   "a code smell is a surface indication that usually corresponds to a deeper
   problem in the system" <fowler>
  Comments
    A bit of a controversial one, it's no so much that the comments themselves
    are rather the fact that you need them to explain what the code is doing,
    you could consider them deoderent <fowler>. Examples of comments which are
    fine are those describing the interface to something
    Avoid comments which describe what a line is doing they are just noise
    Write them using proper English
    Write the comment you would like to read if you were someone else
  Long class
  Long Method
##Style
bad statics
Practices
Inner classes, avoid except in tests
Files
  naming
    descriptive
    camelCase for variables
    PascalCase for everything else
    no underscore
    no all caps
    no hungarian
    avoid prefixes or suffixes like _ m_
  Braces
    new line
    never optional
  Enum
    singular name for type
    Do no add enu as a suffix (Hungarian notation)
  spaces vs tabs
    The most important thing is to be consistent, if you mix them and someone
    has a different setting for the tab width the code will look messed up. 
    Since this is a greenfield project we'll use spaces the primary reason for
    this it is far to easy to add a space to a tab indented file and make a
    mess whereas you can configure your ide to replace tabs with spaces and
    thus can avoid the converse problem.
  Indent Size
    Four characters. Personally I prefer two but four seems to be the standard
    here and consistancy is the most important thing
  line length
    You should strive to keep your lines under 80 characters, yes the ide can
    cope with more easily but you'd be surprised at where you have to look at
    your code (in a browser, in a text editor, when printed, in a terminal). 
    Life is just easier if you try to stick to this, however if it will really 
    impact the readability of the file then this isn't mandatory
usings
  within namespace
    there is a subtle bug which can occur if you declare usings outside of a
    namespace 

    // File1.cs
    using System;
    namespace Outer.Inner
    {
      class Foo
      {
        static void Bar()
        {
          double d = Math.PI;
        }
      }
    }

    // File2.cs
    namespace Outer
    {
      class Math
      {
        public static double PI = 314.0;
      }
    }

    The compiler will search for Math.PI in Outer before it looks in the global
    namespace and so will find the value of 314.0.

    // File1b.cs
    namespace Outer.Inner
    {
      using System;
      class Foo
      {
        static void Bar()
        {
          double d = Math.PI;
        }
      }
    }

    Now the compiler searches System before it searches outer and so finds the
    correct value. It's not a problem that's we're likely to encounter but it
    would be hellish to debug if we did so why ask for trouble.
    <http://stackoverflow.com/questions/125319/should-using-statements-be-inside-or-outside-the-namespace>
  ordering
    
  remove unused
regions
  minimise number of them
Interfaces
  Don't leak
  Would prefer no I prefix as it violates the no hungarian rule however MS
  prefix all of their interfaces like this and consistency is more important
Classes
  length
  order of members
    order by visibility
    order statics then normal
    order constructors, methods then fields
    order methods in the order they are called
  static
  singletons
  single responsibility
  naming
  LSP
Methods
  Tell don't ask
  Single Responsibility
  Express intention (don't optimise prematurely)
  length
  Don't change parameters
  Avoid unessary temporaries
  no defaults at boundries
Variables
  Use Var if you can
  Avoid arrays when there is an appropriate collection
  Avoid primatives if there is a better level of abstration available
Exceptions
  Define top level exceptions for libraries and have all other exceptions in
  that lib derive from it
  Dont have empty catch blocks
  Wrap and rethrow exceptions at abstration boundries

##refs

These references are where I came across a particular idea and are not
necessarily the idea's origin 

1 http://www.joelonsoftware.com/articles/fog0000000069.html
2 http://blog.codinghorror.com/coding-for-violent-psychopaths/
3 Clean Code Uncle Bob
4 http://codeblog.jonskeet.uk/2013/03/15/the-open-closed-principle-in-review/
