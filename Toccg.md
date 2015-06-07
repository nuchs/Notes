# The Opinionated CSharp Coding Guidelines

## Introduction

This guide is written based on the following assumptions 

  - That code is read far more often than it is written 
  - That the requirements for a project always change
  - That the next person who has to understand what you've written is a trigger happy pyschopath with a shotgun and your home address <2>

It therefore recommends a style of coding which prioritises readability (Or
maybe more accurately understandability) over other factors such as ease of 
writing or [performance](#Premature optimisation  == Math.Sqrt(evil);)

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
**Interface Segregation Principle**
**Dependency Inversion Principle**

###YAGNI (You ain't gonna need it)

This principle states that you shouldn't implement something until you actually
need to.
###DRY (Don't repeat yourself)
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
###Premature optimisation  == Math.Sqrt(evil);
###Command/Query Segregation
###Immutability

Pure functions, side effects idempotence
Value types
bad statics
Practices
  TDD
  Pair Programming
  Continuous Integration
  Code Reviews
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
  Readability
    read much more often
    changed much more often after release
    optimise for reading
    Assume the next person to read your code is a psychopath with a shotgun and
    your home address. Do not piss him off
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
Complexity
  manage it 
  cohesion
  coupling
Interfaces
  Don't leak
  Would prefer no I prefix as it violates the no hungarian rule however MS
  prefix all of their interfaces like this and consistency is more important
  keep it small
  dependency inversion or why the top layer owns the interface
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
  Immutability
  Inheritance
    prefer composition
    design for it or prohibit
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
Frameworks
  Hexagonal
  Encapsulate
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
Code Reviews
Continuous Integration
Pair Programming
Security
  Validate input at trust boundries
  white/black list
  clense input
  key management -its hard

refs

These references are my best effort to attribute where the various pieces of
advice came from but given my minimal access to the internet their accuracy
should be taken with a pinch of salt.

1 http://www.joelonsoftware.com/articles/fog0000000069.html
2 http://blog.codinghorror.com/coding-for-violent-psychopaths/
3 Clean Code Uncle Bob
4 http://codeblog.jonskeet.uk/2013/03/15/the-open-closed-principle-in-review/