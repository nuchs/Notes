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

  public Lod (Component aThingy)
  {
    thingy = aThingy;
  }

  public void Rule1()
  {
    this.Rule2();
  }

  public void Rule2(Thing thing)
  {
    thing.DoSomething
  }

  public void Rule3()
  {
    var thing = new Thing();
    thing.DoSomething();
  }

  public void Rule4()
  {
    thingy.DoSomething();
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

In some senses giving a good name to a software entity is one of the most
important things we do when programming. A good name can make the purpose of an
entity immediately apparant wheras a bad name can obscure it or in the worst
case can completely mislead you.

Most people recognise that single character names are a Bad Thing except in
certain well defined situations where convention makes it clear what the
purpose of the variable is (for example using 'i' as the index in a for
loop). However it is still common to use cryptic names and abbreviations which
probably on make sense to the developer and even then only at the time of
writing. In essence you are trading clarity for brevity (dhh).

```csharp
// Admittedly I haven't seen code like this since at least 2007 but it 
// emphisizes the point
public void la (UID i, PWA pa)
{
  var u = gu(i);
  var p = dc(u.ep);

  if (pa == p)
  {
    l(user);
  }
}

// And the same code written more expressively
public void AttemptLogin (UserId id, Password passwordAttempt)
{
  var user = GetUserWith(id);
  var password = Decrypt(user.EncryptedPassword);
  if (passwordAttempt.Matches(password))
  {
    Login(user);
  }
}
```

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

Immutable types are,as their name suggest types whose state doesn't change
after they have been created. The advantage of this is that it is much easier
to reason about the behaviour of immutable objects as the behaviour does not
vary with time.

A somewhat silly example, consider two balls, the first is red and the second
changes colour depending on the time of the day. Which one is it easier to work
out the colour of?

Another advantage of immutable types is that they are safe to share between
threads as there are no synchronisation issues; they have the same state
whenever they are used.

Thus when programming it is worth examining if a type really needs to change or
not. A rule of thumb is that if the identity of the object is irrelevant then
it should be immutable e.g. even if two people have the same name they are
different people so their identity matters whereas two integers with the same
value are interchangeable hence their identity does not matter.
aliasing

### Inheritance
TODO
    prefer composition
    design for it or prohibit
    (GOF, Joshua Bloch)

### Security

Writing secure code is a massive topic which far exceeds the scope of this
guide. The following are just a handful of guidelines which when followed will 
help make you less likely to make the most obvious mistakes.

**Don't give away a free lunch**

Make sure you have an error handling strategy in place, stack traces should not
be reaching the users. Apart from the fact this makes for a poor user
experience it is also giving away information about the internal structure of
your system.

For the sake of a catch statement why make it any easier for potential
attackers?.

**Validate input at trust boundries**

Any data received from outside the system should be treated as suspect, whether
that's something a user has typed in, a request from an external system or an
event from a third party framework you're using. 

Any suspect data should be tested to make sure that it is valid and doesn't 
contain any nasty surprises

There are various techniques you can use when validating input, some of the
more common ones are:

1. Black List - specifically block known bad inputs
2. White List - Only allow known good inputs
3. Process input data in a way that any control statements embedded in the
   data are treated as data and not commands e.g. when with SQL use
   placeholders rather than creating query strings through concatenation e.g
   the following two statements will handle the input 

   > "bob"); DROP TABLE users; --"

   very differently.

```sql
INSERT INTO users (name) VALUES (?);
```

```csharp
var query = "INSERT INTO users (name) VALUES (" + name + ");"
```

**Be careful with temporary files**

If other users have write access to the directory the temporary file is being
written to they can attempt to create their own file with the same name into
the gap between when your file is created and written. This would let them see
the contents of your file or alternatively act as a way of injecting data into
your system.

On a simpler level you should consider any data written to a tempoary file as
exposed to anyone with read access to file.

**Be careful what you log**

Don't write sensitive data to the log file. This one is painfully obvious but
I've seen more credit card numbers and passwords written in the clear than I
care to think about

**Use the lowest privelge level you can**

Assume that your system will be compromised. If you limit what it can do to
what it actually needs to do then you minimise the amount of damage that can
occur when the compromise occurs.

### Globals are bad

It's rare nowadays that you would find a developer who would disagree with the
statement that global variables are a Bad Thing. They violate the Open Closed
principle, their clients depend on their iimplementation and, because
dependencies are transitive all the clients become coupled together. This makes
it both harder to understand the system and to change it.

C# doesn't support global variables directly but it does have support for other
types of global data which share many of the same issues.

**Singletons**

Singletons are like diet globals, they might not be quite as bad as the full
fat version but they're not far off it.

Clients of Singletons almost always use the concrete implementation hence they 
are coupled tightly to it and, because dependency is transitive, to each other.

This makes the code hard to maintain because changes in one area can have
unexpected effects elsewhere.

Because you can no longer consider client code in isolation it becomes harder
to understand.

The code becomes harder to test because you no longer isolate the client code
from its dependency on the Singleton and by the same logic the code becomes
harder to reuse.

**static members**

Static memebers occupy a strange place in C#, they are not a object orientated
concept:

1. They are not polymorphic

* Static members cannot be accessed from an instance reference, they must
  always be accessed via the class name
* Static members cannot be part of an interface
* Static members cannot be virtual

Thus they are always accessed in the same way i.e they are monomorphic

2. Static members do not play nicely with inheritance. As stated above they
   cannot be virtual and so cannot be overridden, they can also behave in non
   obvious ways.

```csharp
public class Parent
{
  public static int number = 1;
}

public class Child : Parent
{
  static Child () { number = 2; }
}

class Program
{
  static void Main ( string[] args )
  {
    Console.WriteLine("Parent number is: " + Parent.number);
    Console.WriteLine("Child number is: " + Child.number);
    var runStaticConstructor = new Child();
    Console.WriteLine("Now Child number is: " + Child.number);

    // Outputs
    // Parent number is: 1
    // Child number is: 1
    // Now Child number is: 2
  }
}
```

3. They weaken or break encapsulation. 

Static methods provide a way to modify the behaviour of an object without going
through the object's (as opposed to class's) interface. This means that objects
cannot be reasoned about in isolation. An example of where this causes problems
is in testing.

```csharp
class Multiplier
{
  private static coefficient = 2;

  public void SetCoefficient (int value) { coefficient = value; }
  public int Multiply(int value) { return value * coefficient; }
}

class TestMultiplier
{
  [Test]
  settingTheCoefficientShouldChangeTheValueTheMultiplierMultipliesBy ()
  {
    var sut = new Multiplier();
    sut.SetCoefficient(3);
    Assert.That(sut.Multiply(2), Is.EqualTo(6));
  }

  [Test]
  theDefaultCoefficientShouldBeTwo ()
  {
    var sut = new Multiplier();
    Assert.That(sut.Multiply(2), Is.EqualTo(4));
  }
}
```

What will happen in the above code is the second test might fail as the order
that the tests run is not deterministic, it is up to the testing framework.
That means that the test might always pass, then fail when you upgrade the
framework, it might intermittently fail or worst of all it might always pass
and you would never realise you had a bug.

So if static members are OO then what are they? 

* Static methods are procedural code.
* Static fields are a form of global state 

Using procedural code is not inherantly wrong but it is worth considering that
C# is an object oriented language with a smattering of support for other
styles. If you're not writing in an OO style then you're going against the
grain of the language, losing all of the benefits it gives you (and if you
don't think those benefits are worth it then perhaps you should consider using
a different language).

A static method is often a hint that the is an object waiting to be born that
will fulfil the responsibilites of that method.

*Recommendation*: Do not make a method static just because it does not access
an instance variable.
*Recommendation*: Avoid using static methods if there is acceptable alternative
*Recommendation*: As the code base grows look to see if new classes can be
introduced which can take over the responsibilities of existing static methods.

As noted static fields are global state, and thus have all the problems that
global variables do (With the exception of namespace collisions, as the class
name effectively acts as a namespace). The problems can be mitagted somewhat by
reducing the access level of the field and by making the field immutable.

A private constant static field only introduces a light amount of coupling
between all the instances of a class but as soon as the field is mutable you
get problems like in the example above and things like threading become an
order of magnitude more complex.

*Rule*: Do not use public mutable static fields
*Recommendation*: Do not use mutable static fields or non private
immutable static fields.

## Techniques

### Refactoring

### Unit Testing

#### DAMP (Descriptive And Maintainable Procedures) <jay fields>

DRY is a useful principle when dealing with production code but it needs to be
applied with care when writing test code. 

One of the key principles when writing tests is that they should be 
[independent](independent tests link) of each other, indiscriminatly applying
DRY will result in code from multiple tests being extracted behind a common
abstraction, thus coupling them together. Sometimes this is harmless but
sometimes it can cause the tests to interfere with each other, this makes the
tests both harder to maintain as a change to one can cause an unexpected
failure in another.

A second reason is that a unit test is supposed to describe a single piece of
behaviour and how it can fail. If the test does fail then ideally you should be
able to look at it and, independent of its surroundings, understand what has
gone wrong.

```csharp
[Test]
public void aThingShouldDoThis ()
{
  runTest(x => x.DoThis());
}

[Test]
public void aThingShouldDoThat ()
{
  runTest(x => x.DoThat());
}

[Test]
public void aThingShouldDoTheOther ()
{
  runTest(x => x.DoTheOther());
}
```

The above example is somewhat extreme of how over applying DRY (Although based on real 
code), after a certain point the test methods stop carrying any useful
information 

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
### TDD

### Continuous Integration


### Code Reviews

### Pair Programming

Pair programming takes the idea that code reviews are good and cranks that up a
notch so that your code is being constantly reviewed. You do this by developing
in pairs with one typing and the other watching and commenting. The roles
should regularly switch throughout the session and the produced code should be
a collaberative effort.

Some of the benefits

* With two eyes on the code you are less likely to make mistakes
* You are more likely to follow the guidelines and less likely to take
  shortcuts with someone watching you
* It is a great way to share knowledge amongst the team
* It increases the sense of ownership in the team (No more, "Oh the UI is Bob's
  code")

*Suggestion*: Give it a try and see if it works for you.

(Kent Beck XP Explained)

## Style

There are lots of different styles of writing code and generally their aim is
all the same, to make the code as clear and easy to read as possible. There is
a lot of disagreement over what is the best way to do that but in the end
people can get used to just about any sensible style.

Thus the prime style directive is: be consistent

If the style keeps changing then you are forcing the user to think about the
style of the code rather than the code itself.

Thus whereas the first part of this document provides guidelines on how code
should be structured the following are rules which must be followed to ensure
that everyone in the team is producing code with a consistant style.

### General

**Spaces vs tabs**

The arguments over which is better can reach nearly religious proportions and
about the only thing people can ever agree with is that mixing the two is bad
as it will mess up the formatting.

Rule: For greenfield projetcs use spaces (You can configure your IDE to replace
tabs with spaces). For existing projects consistency is more important so use
whatever is already in use.

**Indent Size**

*Rule*: Indent size will be set to 4 spaces, this is to be consistent with the
style used in our libraries.

**Brace Style**

*Rule*: Braces should appear on a new line, this is to be consistent with the
style used in our libraries

e.g.

void MyFunc ()
{
}

rather than

void MyFunc () {
}

*Rule*: Braces are never optional, this prevents errors when extra statements are
added to a control block.

// Not allowed
if (statement)
  DoSomthing();

// Do it this way
if (statement)
{
  DoSomething();
}

**Naming**

*Rule*: unless stated differently in another rule all names should be in
PascalCase
*Rule*: do not use underscore in names
*Rule*: do not use all caps names for constants

This is to be consistant with the Microsoft naming conventions used in the C# 
libraries.

*Rule*: Do not use Hungarian notation

This violates both the DRY principle and makes the name less expressive (as it
is describes what something is, not what it does).

**Comments**

*Recommendation*: XML comments should appear in the following places
1. At the start of each class, enum or interface, a short description of the
   purpose of the item.
2. At the start of each public or protected method or property, a short
   description of the purpose of the item.

*Recommendation*: Do not put XML comments

This minimizes the amount of noise introduced by the comments while still
providing enough useful information to have documentation automatically
generated by a tool.

### Files

**Contents and Name**

People expect to find the definition of a class/interface/enum in a file with
the same name, thus by the principle of least surprise:

*Rule*: Each file should contain one public class/interface/enum
*Rule*: Each file should be named after the public entity it contains

**Line Length**

Modern monitors have no issue with displaying long lines so suggesting a cap on
line length may seem archaic but you don't always know where you will need to
view the code (In a browser, in a remote shell, on a print out etc) and
restricting yourself to 80 characters will make it display correctly in most
environments. There is also a question of how readable your code is if you're
writing 200 character lines.

*Consider*: Limit line length to 80 characters unless it will seriously impact
readability

**Regions**

Regions are not part of the C# language they contribute nothing to your product
and are stripped out by the preprocessor before the compiler even sees your
code. So what is the point of them?

Region exist to allow you to specify custom fold points in your source code
that your IDE can use. That is, they are a tool to help make your code more
readable in an IDE which supports this feature.

Each region adds a fair amount of visual noise and bulk to your code so to be 
worth using the code that they wrap must be of a reason length. In fact it must
be long enough that it is impacting your ability to understand what's going on
in the class (Otherwise why bother using the region).

So to put it another way regions help you hide the fact that the class is badly
written from a readability perspective.

*Rule*: Do not use regions, make your code readable instead

**Usings**

*Rule*: unrequired using statements should be deleted
*Rule*: using statements should be sorted according to the default Visual
Studio sort order.

This just makes it easier to see what is being included in the file. The right
click menu in VS provides an option to do this automatically for you.

*Rule*: using declarations should be placed within the files namespace 
declaration.

This is to avoid the following bug:

```csharp
// File1.cs
using System;

namespace Outer.Inner
{
  class Foo
  {
    void Bar()
    {
      double d = Math.PI;
    }
  }
}

// File2.cs
namespace Outer
{
  public class Math
  {
    public static double PI = 314.0;
  }
}
```

The compiler searches for the definition of entities starting in the current
namespace and works its way outwards. Thus it will search for Math.PI in Outer 
before it looks in the global namespace and so will find the value of 314.0.

```csharp
// File1b.cs
namespace Outer.Inner
{
  using System;

  class Foo
  {
    void Bar()
    {
      double d = Math.PI;
    }
  }
}
```

Now the compiler searches System before it searches outer and so finds the
correct value. It's not a problem that's we're likely to encounter but it
would be hellish to debug if we did so why ask for trouble.

<http://stackoverflow.com/questions/125319/should-using-statements-be-inside-or-outside-the-namespace>

**Dead Code**

*Rule*: delete it.

Dead code, whether it is commented out, '#if false'd or in a conditional branch
which will never be executed, contributes nothing to your system.  There is no
need to 'save' the code in case you need it later, any revision control
system will take care of that for you.

Dead code is purely noise, it just makes your code harder to read.

###Classes

**Nested Types**

*Recommendation* avoid using nested types.

Break encapsulation
Add complexity

**Enums**

*Rule*: Do not use the word Enum as either a prefix of suffix for the name. An
enum should provide an abstraction over a set of values and this is leaking how
the abstraction is implemented.
*Rule*: use singular names for enums, it reads more naturally when the enum is
used.

```csharp
enum Colour  { Red, Green, Blue};
enum Colours { Red, Green, Blue};

public void PaintItRed ()
{
  // Singular name reads better here
  screen.Paint(Colour.Red);
  screen.Paint(Colours.Red);
}
```

**Interfaces**

*Rule*: Prefix interface names with a capital I e.g.

```csharp
public interface ICook {}
```

Although this violates the rule on not using Hungarian Notation, this is the
style used in all the Microsoft libraries. To not follow it would cause some
serious consistancy issues

**Member Order**

Members of classes should be ordered as follows

1. First by visibility
2. Then by type: constructors, methods, properties, fields
3. Statics before regular members

```csharp
class MyClass
{
  public MyClass ()
  {
  }

  public void MyPublicMethod ()
  {
  }

  public int MyProperty { get; private set; }

  protected
}
```

###Methods

**Parameters**

*Recommendation*: Try to minimise the length of parameter lists, the more
parameters a method has the harder it is to understand.
*Recommendation*: Avoid boolean parameters, they don't provide any useful
information when reading code containing calls to the method.

```csharp

// Lots of parameters, lots of booleans, lots of fun working out what they're
// all for
Excel.Workbook excelWorkbook = excelApp.Workbooks.Open(workbookPath,
        0, false, 5, "", "", false, Excel.XlPlatform.xlWindows, "",
        true, false, 0, true, false, false);
```

**Spacing**

*Rule* There should be one space between the method name and the opening 
parenthesis for its arguments in the method definition.

*Rule* There should be no spaces between the method name and the opening 
parenthesis for its arguments in method calls.

*Rule* In control statements there should be one space between the control
statement keyword and the opening parenthesis.

*Rule* There should be one blank line between methods definitions

```csharp
public void MyMethod (int anInt, bool aBool)
{
  for (int i = 0; i < anInt; i++)
  {
    // DoSomething
  }
}

public void AnotherMethod ()
{
}
```

There are many perfectly valid styles that could be used, one has been chosen
purely to ensure consistency across the code base.

**Vertical Length**

TODO: Short and sweet


### Fields, Properites, Parameters and Variables

**Naming**

*Rule*: use camelCase when writing names for variables and parameters, this is 
to remain consistent with the Microsoft libraries

*Rule*: Do not use prefixes or suffixes to indicate the scope of a field,
parameter, property or variable.

This is a form of Hungarian notation and it will impact the users ability to
read the meaningful part of the name (when recognising words we look at their
shape, especially the first and last parts, adding suffixes and prefixes will
interfere with this).

**Inferred type**

*Recommendation*: use the var keyword when declaring local variables as long as
the type is clear from the context.

```csharp

// The following two statements are identical in terms of effect

List<string> words = new List<string>();
var words = new List<string>();

```

Using var follows the DRY principle as you are not writing the variables type
out multiple times. The code is also slightly more readable as there is less
visual noise on the page.

**Default parameters**

Default parameters provide a nice way of simplfying method interfaces and
avoiding noise introduced by the telescoping constructor pattern but there are
some gotchas to do with versioning.

```
// Version 1 of SomeMethod in the library SomeLibrary.dll
public void SomeMethod (string param1="one", string param2="two") 
{ 
  /* Do something */ 
}

// meanwhile in AnotherAssembly.dll
SomeMethod("yarr");
```

In the above example SomeMethod is part of the public interface for
SomeLibrary.dll and at the end of the example we can see it being used in 
AnotherAssembly.dll. Assume not that we need to add a new parameter to 
SomeMethod

```
// Version 2 of SomeMethod
public void SomeMethod (string param1="one", string param2="two", string param3="three") 
{ 
  /* Do something */ 
}
```

This will compile fine but unless we can recompile AnotherAssembly then then
next time AnotherAssembly runs it will fail to call SomeMethod. This is because
it is still looking for the two parameter version.

This issue only occurs when a method is called from an external library which
was not compiled at the same time as the library containing the method. 

*Rule*: Methods that form the part of the public interface of a library must
not use default values for parameters.

http://codeblog.jonskeet.uk/2015/06/03/backwards-compatibility-is-still-hard/

**Const values**

A very similar problem to that with default parameters occurs with constant
values. 

Const members are replaced by their value in the compiled code (rather than a
member access). If you have an access in another assembly this value will only
be updated when the assembly is recompiled. Thus if you change the value of the
const without recompiling the external assembly, the external assembly will
contain an out of date value.

*Rule*: const fields must not form part of the public interface of an assembly

http://www.stum.de/2009/01/14/const-strings-a-very-convenient-way-to-shoot-yourself-in-the-foot/

**Primitive Obesseion**

Primitives are, almost by definition, an implementation detail. They do not
help us to understand the problem domain. Thus primitves should only be used at
the lowest level of abstraction, at higher levels objects representing concepts
from the domain should be used instead.

```csharp
// Too primitive
double GetDiscountPrice (Product product, double discount)
{
  double discountPrice = product.Cost * discount;

  return discountPrice;
}

// Better
Money GetDiscountPrice (Product product, Discount discount)
{
  Money discountPrice = discount.Apply(product.Cost);

  return discountPrice;
}
```

*Recommendation*: Avoid using primitive types where a domain object would make
more sense.

###Exceptions

**Exception Hierarchy**

*Recommendation*: For each library that throws an exception define a top level
exception and have all other exceptions thrown by the library dervive from it.

This is to make it easier for clients to use the libary.

**Empty Catches**

*Rule*: don't leave catch blocks empty (or just containing a log line). Follow
this procedure:

1. If you can resolve the issue that caused the exception, do so.
2. If you can't do step 1. but you can provide more information about the cause
   of the exception (for example by logging some contextual information), do 
   so. Then rethrow the exception.
3. If you can't do either 1. or 2. then don't catch the exception in the first
   place.

```csharp
// A note on rethrowing exceptions
try
{
  // Do something which might throw an exception
}
catch(ExceptionBad e)
{
  // If you try to rethrow the exception e like this you'll destroy its
  // stacktrace
  throw e;
}
catch(ExceptioinGood e)
{
  // Rethrowing it this way preserves the stacktrace
  throw
}
```

##refs

These references are where I came across a particular idea and are not
necessarily the idea's origin 

1. http://blog.codinghorror.com/when-understanding-means-rewriting/
2. http://blog.codinghorror.com/coding-for-violent-psychopaths/
3. Clean Code Uncle Bob
4. http://codeblog.jonskeet.uk/2013/03/15/the-open-closed-principle-in-review/
5. uncle bob open closed princple article
6. http://www.joelonsoftware.com/articles/LeakyAbstractions.html
