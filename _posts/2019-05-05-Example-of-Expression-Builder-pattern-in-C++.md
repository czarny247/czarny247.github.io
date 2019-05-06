---
layout: post
title:  "[C++] Expression Builder pattern"
date:   2019-05-05
categories: design_patterns c++ builder_pattern fluent_interface
---

Link to a repository: <https://github.com/czarny247/ExpressionBuilderExample>

## Introduction

**Expression Builder pattern** is just a simple combination of two widely known patterns - the **Builder pattern** and the **Fluent Interface pattern**. 

In this article I'd like to show you how to implement it in **C++** language.

It will also make you familiar with **CRTP idiom** which is great C++ feature which allows us to use **Expression Builder pattern** to define Builders for inherited types.

## Developing a Builder

### Builder pattern 

As you probably already know, the Builder pattern belongs to the group of creational design patterns. Its role is to allow the client to create a complex object (product) in a step-by-step manner. Allowing the client to avoid using a bloated constructor of the product class could be an additional profit.

There's a simple example how basic builder could look like:

```c++
//class defining a "product"
class Horse
{
public:
	Horse(std::string name, 
          CoatColor coatColor, 
          HairColor maneColor, 
          HairColor tailColor);

//<...>
    
private:
	std::string name_;
	CoatColor coatColor_;
	HairColor maneColor_;
	HairColor tailColor_;    
};

//class defining a "builder"
class HorseBuilder
{
public:
    HorseBuilder();
    ~HorseBuilder();
    
    void name(std::string name);
    void coatColor(CoatColor coatColor);
    void maneColor(HairColor maneColor);
    void tailColor(HairColor tailColor);
   	
    std::shared_ptr<Horse> build();
    
private:
	std::string name_;
	CoatColor coatColor_;
	HairColor maneColor_;
	HairColor tailColor_;    
};

//finally how does it works
//<...>
HorseBuilder horseBuilder;
horseBuilder.name("Roach");
horseBuilder.coatColor(horse::CoatColor::Brown);
horseBuilder.maneColor(horse::HairColor::Black);
horseBuilder.tailColor(horse::HairColor::Black);
auto builtRoach = horseBuilder.build();

//instead of directly calling bloated and unreadable c-tor
Horse roach("Roach", 
            horse::CoatColor::Brown,
           	horse::HairColor::Black,
           	horse::HairColor::Black);
//<...>
```

Without going in implementation details it looks like clearer alternative, especially for classes which uses c-tors with many arguments.

Unfortunately the part where builder methods are called one after another does not look very convincing.

How about making it look like this:

```c++
HorseBuilder horseBuilder;
auto roach = 
    horseBuilder
	.name("Roach")
	.coatColor(horse::CoatColor::Brown)
	.maneColor(horse::HairColor::Black)
	.tailColor(horse::HairColor::Black)
	.build();
```

Definitely better! How it is achieved?

Well.. That's the moment when the Fluent interface pattern enters the stage!

### Builder + Fluent Interface = Expression Builder

This pattern is often utilized to design the API based on method chaining "with the goal of making the readability of the source code close to that of ordinary written prose, essentially creating a domain specific language within the interface".

Here's how builder has been changed:

```c++
//class defining an "expression builder"
class HorseBuilder
{
public:
    HorseBuilder();
    ~HorseBuilder();
    
    HorseBuilder& name(std::string name);
    HorseBuilder& coatColor(CoatColor coatColor);
    HorseBuilder& maneColor(HairColor maneColor);
    HorseBuilder& tailColor(HairColor tailColor);
   	
    std::shared_ptr<Horse> build();
    
private:
	std::string name_;
	CoatColor coatColor_;
	HairColor maneColor_;
	HairColor tailColor_;    
};
```

This kind of builder pattern variant is often called **Expression Builder**.

After these modification we can finally create horses like this:

```c++
HorseBuilder horseBuilder;
auto roach = 
    horseBuilder
	.name("Roach")
	.coatColor(horse::CoatColor::Brown)
	.maneColor(horse::HairColor::Black)
	.tailColor(horse::HairColor::Black)
	.build();
```

I'll later show you how to improve this implementation to make this call look even more like the expression which is desired when we use the **Expression Builder pattern**.

At this point let's take care of something else.

Let's introduce the class **Unicorn** which inherits from **Horse**.

It will of course lead us to implementing **Unicorn Builder** class and that's where the problem begins.

### Expression Builder and the inheritance problem

Let's see how the **Unicorn** class looks like:

```c++
class Unicorn : public Horse
{
public:
	Unicorn(std::string name, 
			CoatColor coatColor, 
			HairColor maneColor, 
			HairColor tailColor, 
			HornColor hornColor);
	//<...>

private:
	HornColor hornColor_;
};
```

As we can see, the **Unicorn** "extends" **Horse** by adding the **HornColor**.

**UnicornBuilder** will look like this:

```c++
class UnicornBuilder : public HorseBuilder
{
public:
	UnicornBuilder();
	~UnicornBuilder();

	UnicornBuilder& hornColor(HornColor hornColor);

	std::shared_ptr<Horse> build() override;

private:
	HornColor hornColor_;
};
```

So the **Unicorn** creation will look like this:

```c++
UnicornBuilder unicornBuilder;
auto rainbowHorn = 
	unicornBuilder
		.hornColor(horse::unicorn::HornColor::Rainbow)
		.name("Rainbow Horn")
		.coatColor(horse::CoatColor::Pale)
		.maneColor(horse::HairColor::Pale)
		.tailColor(horse::HairColor::Pale)
		.build();
```

Current implementation forces us to call **hornColor** method as first because calling rest of the methods will return **HorseBuilder** instead of **UnicornBuilder**. It's an example of *Object slicing* which results in losing all **UnicornBuilder** "extensions" to **HorseBuilder**. It is considered as drawback since **Fluent Interface** should allow us to call its methods in any order (*exception for builder - build step should be the last one, depending on the implementation, but the other "setter" methods should have possibility to be called in any order - I will show later how to improve it*).

Fortunately C++ allows us to solve this problem with use of **CRTP idiom**.

### Fixing the ExpressionBuilder with the help of CRTP idiom

**CRTP** stands for **Curiously Recurring Template Pattern** idiom.

Don't be scared with the complicated name. CRTP simply *"is an idiom in C++ in which a class `X` derives from a class template instantiation using `X` itself as template argument."*

Its general form looks like this (from *Wikipedia*):

```c++
// The Curiously Recurring Template Pattern (CRTP)
template<class T>
class Base
{
    // methods within Base can use template to access members of Derived
};
class Derived : public Base<Derived>
{
    // ...
};
```

In my example **CRTP** is used to allow *Polymorphic method chaining* which means that we will avoid *Object slicing* mentioned in the previous paragraph.

This approach will change inheritance arrangement between Builders.

**HorseBuilderBase** class will be an *abstract base class* which defines all common parts of "real" Builders (the ones which produce objects of specific classes - *Unicorns* and *Horses*).

Here comes the code for "Base Builder":

```c++
template <typename TBuilderType>
class HorseBuilderBase
{
public:
	HorseBuilderBase();
	virtual ~HorseBuilderBase();

	TBuilderType& name(std::string name);
	TBuilderType& coatColor(CoatColor coatColor);
	TBuilderType& maneColor(HairColor maneColor);
	TBuilderType& tailColor(HairColor tailColor);

    //pure virtual build() method since this class don't build any real objects
	virtual std::shared_ptr<Horse> build() = 0;

protected:
	std::string name_;
	CoatColor coatColor_;
	HairColor maneColor_;
	HairColor tailColor_;

private:
    //method implementing cast of 'this' to proper 'TBuilderType'
	TBuilderType& builder();
};
```

```c++
template<typename TBuilderType>
TBuilderType& HorseBuilderBase<TBuilderType>::builder()
{
	return *(static_cast<TBuilderType*>(this));
}
```

And "Real builders":

```c++
class HorseBuilder : public HorseBuilderBase<HorseBuilder>
{
public:
	HorseBuilder();
	~HorseBuilder();

	std::shared_ptr<Horse> build() override;
};
```

```c++
class UnicornBuilder : public HorseBuilderBase<UnicornBuilder>
{
public:
	UnicornBuilder();
	~UnicornBuilder();

	UnicornBuilder& hornColor(HornColor hornColor);

	std::shared_ptr<Horse> build() override;

private:
	HornColor hornColor_;
};
```

And finally the client code:

```c++
std::vector<std::shared_ptr<horse::Horse>> stables;

horse::HorseBuilder horseBuilder;

stables.emplace_back(
	horseBuilder
		.name("Roach")
		.coatColor(horse::CoatColor::Brown)
		.maneColor(horse::HairColor::Black)
		.tailColor(horse::HairColor::Black)
		.build());

horse::unicorn::UnicornBuilder unicornBuilder;

stables.emplace_back(
    unicornBuilder
		.name("Rainbow Horn")
		.hornColor(horse::unicorn::HornColor::Rainbow)
		.coatColor(horse::CoatColor::Pale)
		.maneColor(horse::HairColor::Pale)
		.tailColor(horse::HairColor::Pale)
		.build());
```

## Further development Ideas

### Builder improvements

I've found some improvements for Builder while digging the web.

1. **Restrict construction to the Builder.**

   Its very simple improvement to implement but I think it make client code to be uniform since object's creation will be achievable only one way - through Builders.

   Although its simplicity for non-inheritance case, it will be hard to implement it for the one including inheritance among product classes.

   more: <https://riptutorial.com/cplusplus/example/30166/builder-pattern-with-fluent-api>

2. **Add Director class.**

   Director class will manage the builders and choose proper one depending on client's needs.

   more: <https://refactoring.guru/design-patterns/builder>

## Summary 

//Add summary

More on Expression Builder / Fluent interface / Method chaining:

<https://www.martinfowler.com/bliki/ExpressionBuilder.html>
<https://www.martinfowler.com/dslCatalog/expressionBuilder.html>
<https://en.wikipedia.org/wiki/Method_chaining>

More on CRTP:

<https://stackoverflow.com/questions/52530440/boostpolycollection-stdvariant-or-crtp>
<https://stackoverflow.com/questions/8113878/c-crtp-and-accessing-deriveds-nested-typedefs-from-base/8113956>
<https://blog.galowicz.de/2016/02/26/how_to_use_crtp_to_reduce_duplication/>
<http://www.modernescpp.com/index.php/c-is-still-lazy>
<https://marcoarena.wordpress.com/2012/04/29/use-crtp-for-polymorphic-chaining/>
<https://www.fluentcpp.com/2017/05/12/curiously-recurring-template-pattern/>

More on Builder Pattern:

<https://medium.com/beingprofessional/think-functional-advanced-builder-pattern-using-lambda-284714b85ed5>
<https://dzone.com/articles/design-patterns-the-builder-pattern>
<https://stackoverflow.com/questions/17937755/what-is-the-difference-between-a-fluent-interface-and-the-builder-pattern>
<https://medium.com/@sawomirkowalski/design-patterns-builder-fluent-interface-and-classic-builder-d16ad3e98f6c>
<https://gist.github.com/pazdera/1121152>
<https://softwareengineering.stackexchange.com/questions/278354/builder-design-patterns-passing-parameters-from-client-to-the-builder>
<https://stackoverflow.com/questions/17554618/c-builder-pattern-with-inheritance>