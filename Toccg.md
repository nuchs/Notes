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

  - That code is read far more often than it is written <1>
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
exercise judgment in deciding when to apply them (especially when they 
contradict each other). That said they are good advice in most cases so if you 
decide to go against their recommendations make sure you understand the 
tradeoff you are making.

### YAGNI (You ain't gonna need it)

This principle states that you shouldn't implement something until you actually
need it.

The rationale behind this is that it is hard to guess what features will be
needed in the future and how they should be built. 

If you guess wrong then you've wasted time building something which won't be
used and you've made the code base more complex unless you spend more time
removing it.

If you guess right but don't implement it correctly (say because the
requirements haven't been bottomed out because the feature is not yet a
priority) then you've spent time building something which isn't needed just now
instead of building something which is. Your code base is now more complex and
you will need to modify the code in the future to correct your mistake.

If you guess right and you implement it correctly then you've still spent time
on a feature which could have been spent on something else which is required.

So whenever you feel the need to make the code more 'flexible' because you're
sure that some feature will definitely be needed in the future stop for a
second and think if it's actually worth it.

### Abstraction

Most programs are too complex for anyone to hold the whole thing in their head
at the one time. We manage this by mentally structuring the code into a 
hierarchy of abstractions and only examining one level at a time e.g.

1. When reasoning about a service which uses a notification system you think of
   the notification system as a black box which is able deliver messages to 
   recipients you don't concern yourself with its implementation.

2. If you examine the notification system you see that it uses a queue that
   helps you to understand how the system works but you're not interested in
   how the queue works

3. If you then look at the queue you see that it is implemented using a singly
   linked list. At this point you don't care how the queue is used just how it
   works.

Each of these steps involves looking at the system at a particular level of 
abstraction. The key thing is that you ignore the details of everything at a
different level, this helps us manage the inherent complexity of a program.

Since this is a key way in which we are able to understand a system it is
important to design code in a way which supports our ability to form
abstractions.

**Single level of abstraction** <Uncle bob>

In a single method all statements should be at the same level of abstraction
otherwise you are requiring the reader to perform a context switch every time
the level changes

```csharp
// This method uses two levels of abstraction for the first part it is looking
// at how to validate particular part of a credit card while in the seond part 
// it is looking at how to look if a credit card is valid
public bool ValidateCreditCard(CreditCard card)
{
  var luhnNumbers = DoubleEveryOtherNumber(card.LongNumber); 
  var sum = (from digit in luhnNumbers select digit).Sum();
  var validNumber = (sum % 10) == 0;
  
  return validNumber &&
         ValidStartDate(card.StartDate) &&
         ValidSecurityNumber(card.SecurityNumber);
}

// Now everything is at the same level of abstraction
public bool ValidateCreditCard(CreditCard card)
{
  return ValidateCreditCardNumber(card.LongCardNumber)  &&
         ValidStartDate(card.StartDate) &&
         ValidSecurityNumber(card.SecurityNumber);
}

private bool ValidateCreditCardNumber (int[] longCardNumber)
{
  var luhnNumbers = DoubleEveryOtherNumber(longNumber); 
  var sum = (from digit in luhnNumbers select digit).Sum();
  return (sum % 10) == 0;
}
```

**Abstractions shouldn't leak (much<6>)**

An abstraction is said to be leaky if it reveals some of the implementation
details of the thing it is supposed to be hiding

```csharp
interface IEmailManager
{
  void SendEmail(FileInfo email); 
  List<FileInfo> GetMail();
}
```

The above interface provides an abstration to hide the details of an email
system. It lets the client focus on the two behaviours it's interested in,
sending and receiving emails. The problem is that it's apparant from the return
and parameter types that the emails are being stored on disk. This is irrelvant
to the client, it is an implementation detail that has leaked through the
abstraction.

The reason this is bad is that the client is now coupled to the implemenation
of the thing being abstracted. In this case if you wanted to change the email
manager so that it stored the emails on a database you would need to update all
of the clients too.

When designing interfaces (small i) care should be taken to prevent it from
leaking. The better you do at this, the better your abstraction will be and
thus it will be easier for readers to understand your code.

### SOLID <3>

SOLID is a mnemonic acronym for five design principles which when followed are
intended to help a developer write code which is easier to understand and
maintain. The five principles are

- **S**ingle responsibility principle
- **O**pen/closed principle
- **L**iskov substitution principle
- **I**nterface segregation principle
- **D**ependency inversion principle

They are described below.

**Single Responsibility Principle (SRP)**

This principle states that a class should conceptually be responsible for one
thing

Following this principle provides a number of advantages

- Because classes are focused on only doing one thing they become much easier
  to reuse. This has a further benefit in that if code is reused it becomes
  easier to update the system as there are less places where changes need to be
  applied
- Classes tend to be shorter and thus are easier to understand (less to read
  usually means less to think about).
- Classes are also easier to work with because there is only one concept you
  need to wrap your head around. Every part of the class should contribute to
  your understanding of that concept.
- Classes become easier to change. If a class has more than one responsibility the implementation of these responsibilites are often entangled (If they were clearly segregated then the class would probably have been split up already). This means when one responsibility changes the other will be affected. If a class has a single responsibility this problem is side stepped entirely.

The SRP can also be applied to methods and many of the same benefits apply.

There is a certain amount of subjectivity in deciding how to apply this
principle; one person's single responsibility is another person's multiple. A
good rule of thumb when trying to work this out is to see if you describe what
a class does without using any conjuctives ('and', 'or') <growing software>

**Open/Closed Principle (OCP)**

The principel as stated is

> Software entities should be open for extension but closed for modification.

In this context open can be interpreted to mean 'will accept changes of the
specified type' and closed means the opposite. Thus the open closed principle
recommends that it should be able to add new behaviour to a particular entity
without modifying it. 

On first reading this sounds impossible but it is actually quite straight
forward, as long as the software entity depends on a abstraction 

```csharp
// In this first implementation CookedBreakfast depends on the concrete class
// Fryer to cook the breakfast. If we decide that we'd rather start grilling
// our food then we need to change its implementation, thus this class does not
// follow the open closed principle
class CookedBreakfast
{
  public CookedBreakfast()
  {
    fryer = new Fryer();
  }

  public Cook(List<Food> ingrediants)
  {
    fryer.Cook(ingrediants);
  }
  
  private Fryer fryer;
}

// In the second implementation CookedBreakfast now depends on the ICook
// abstraction. If we need to switch from a Fryer to a Griller then we just
// need to construct a new instance with a Griller object. Thus we are able to
// change its behaviour without modifying the class, thus this class does
// follow the open closed principle
class CookedBreakfast
{
  public CookedBreakfast(ICook cooker)
  {
    this.cooker = cooker;
  }

  public Cook(List<Food> ingrediants)
  {
    cooker.Cook(ingrediants);
  }
  
  private ICook cooker;
}

class Fryer : ICook 
{
  //implementation
}

class Griller : ICook 
{
  //implementation
}
```

By consistantly using abstractions to mediate communications between software 
entities (Such as using the ICook interface to mediate between the
CookedBreakfast and Fryer classes) you create code in which the entities are
all isolated from each other. This has two main advantages

1. Your code becomes easier to understand because you can reason about a
   particular block without having to understand the parts that it depends on
2. The abstractions act as a firewall to change, preventing a change to one
   part of a system rippling out and requiring other parts to also change. This
   makes the code easier to maintain.

**Liskov Substitution Principle (LSP)**

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

**Interface Segregation Principle (ISP)**

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
but this can (and does) create situations where libraries are having to be 
redeployed even though they have functionaly not changed.

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

**Dependency Inversion Principle (DIP)**

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

### DRY (Don't repeat yourself)

> Every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

One of the key reasons is to do with change. If you have mutiple
representations of something and you need to change that thing then you have to
update *all* of the representation, otherwise Bad Things will happen.

An example of an update error in database caused by duplication.

---------------------------------------
EmployeeId | Address      | Description
---------------------------------------
0001       | 1 Big Road   | Wage
---------------------------------------
0002       | 2 Big Road   | Wage
---------------------------------------
0002       | 4 Small Road | Bonus
---------------------------------------

Here there are multiple represntations of employee 0002's address in the pay
table and one of them hasn't been updated which is going to cause some problems
with their pay packet this month. The relational database normal forms exist 
precisely to prevent this sort of duplication.

In code update errors will usually result in bugs. Take for example FooBar inc.
who calculates important metrics related to Foo and Bar as follows.

```csharp
double AverageFoo (List<Foo> foos)
{
  double sum = 0.0;
  foreach (var foo in foos)
  {
    // Do some complicated stuff related to foo
    sum += fooAmount;
  }

  return sum / foos.Length;
}

double AverageBar (List<Bar> bars)
{
  double sum = 0.0;
  foreach (var Bar in bars)
  {
    // Do some complicated stuff related to bar
    sum += barAmount;
  }

  return sum / bars.Length;
}
```

For the metrics to be meaningful the averages for Foo and Bar must be
calculated in the same way. Suppose now FooBar inc decide that the median would
be a more useful way to caculate the average and so update their code as
follows.

```csharp
double AverageFoo (List<Foo> foos)
{
  // Do some complicated stuff related to foo
  int midPoint = // work out midpoint

  return foos[midPoint];
}

double AverageBar (List<Bar> bars)
{
  double sum = 0.0;
  foreach (var Bar in bars)
  {
    // Do some complicated stuff related to bar
    sum += barAmount;
  }

  return sum / bars.Length;
}
```

Now there is a bug in their code because the developers didn't realise the 
average calculation happened in more than one place.

Duplication is often a hint that there is a class or method waiting to be born.
The new entity will act as the authoritative representation of the duplicated
concept and be used in place of instances where the duplication occurs. So to
continue our example.

```csharp
double AverageFoo (List<Foo> foos)
{
  // Do some complicated stuff related to foo

  return CalculateAverge(fooValues);
}

double AverageBar (List<Bar> bars)
{
  // Do some complicated stuff related to bar

  return CalculateAverge(barValues);
}

double CalculateAverge (List<double> values)
{
  double sum = 0.0;
  foreach (var value in values)
  {
    sum += value;
  }

  return sum / values.Length;
}
```

Comments describing what code is doing are another example; it is not
uncommon in code bases which have been around for a while to see comments which
bear no relation to the code. While this may not be as serious as invalidate
the data in db or introducing a bug it does make code much harder to
understand.

To sum up, duplication is bad; don't do it, and remove it where you find it.

<The Pragmatic Programmer>

### DAMP (Descriptive And Meaningful Phrases) <jay fields>

The DAMP principle is applicable to unit tests and it suggests that when
writing tests it is more important for the code to 

### Tell don't ask (And the Law of Demeter)

The "Tell don't ask" principle states that you should tell objects what to do not
ask them for data and take actions based on it. 

```csharp
// Here we ask the object numbers for a part of its internal represntation
// and perform an operation on it.
public Ask (SomeNumbers numbers)
{
  int[] numbers = someNumbers.getTheNumbers();

  for (int i = 0; i < numbers.Length; i++)
  {
     numbers[i] = numbers[i] * 2;
  }
}

// Here we tell the numbers object what we want done but we let it handle doing
// actual work.
public Tell (SomeNumbers numbers)
{
  numbers.ForEach(x => x *2);
}
```

The difference between these two methods is that if we need to change how
SomeNumbers stores it's collection of numbers it will be simpler to do in the
latter case. This is because in Ask method has become coupled to implementation
of the SomeNumber class.

The aim of this principle therefore is to reduce coupling between classes
thus making the system easier to change. 

"The Law of Demeter" which is an alternative formulation of "Tell don't ask"
provides a simple set of rules to follow

A method "M" of an object "O" should invoke only the the methods of the
following kinds of objects:

1. Itself
2. Its parameters
3. Any objects it creates/instantiates
4. Its direct component objects 

```csharp
class LoD
{
  Component thingy

  public void Rule1()
  {
  }

  public void Rule1()
  {
  }

  public void Rule1()
  {
  }

  public void Rule1()
  {
  }
}
```

https://pragprog.com/articles/tell-dont-ask
http://www.bradapp.com/docs/demeter-intro.html

### Principle of least surprise (or astonishment)

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

### Expressive Naming


### Premature optimisation == Math.Sqrt(evil);

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

### Immutability
Value types
aliasing
pure functions idempotent

### Inheritance
    prefer composition
    design for it or prohibit
### Security
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

1. http://blog.codinghorror.com/when-understanding-means-rewriting/
2. http://blog.codinghorror.com/coding-for-violent-psychopaths/
3. Clean Code Uncle Bob
4. http://codeblog.jonskeet.uk/2013/03/15/the-open-closed-principle-in-review/
5. uncle bob open closed princple article
6. http://www.joelonsoftware.com/articles/LeakyAbstractions.html
