---
layout:     post
title:      constexpr explained
date:       2015-04-16 20:47:01
summary:    The new standard introduces the <i>constexpr</i> keyword. In this post I will try to explain when to use it, and provide some examples. 
categories: programming c++11 
---

### Introduction

The new standard lets us use the **constexpr** keyword to create _constant expressions_. What's a constant expression? Allow me to quote the author of a well known bestseller, _C++ Primer_: 
> A constant expression is an expression whose value cannot change and that can be evaluated at compile time.

### _constexpr_ variables

The first question that comes to mind is: _is a **const** variable a constant expression_? As mentioned before, to render a variable a constant expression, its value needs to be known at compile time. Let's look at an example:
{% highlight c++ linenos %}
const int literal_int = 20;
const int file_size = literal_int + 1;
int max_file_size = 30;
const int file_no = get_file_no();
{% endhighlight %} 
 * A _literal_ (the number 20 in the first example) is a constant expression. The compiler knows the value, and the value cannot change. So when we initialize `const int literal_int`, we know it is a constant expression
 * `const int file_size` is initialized with a sum of a literal, and a constant expression. Each of them is a constant expression, therefore `file_size` is a constant expression
 * `int max_file_size` is initialized with a constant expression. But the variable itself isn't const. That means - its value can, and probably will, change along the way. That is why it is **not** a constant expression.
 * The last variable is _const_, we know its value won't change. But is the init value know at compile time? No. The value will be know at run time. That is why it is **not** a constant expression (with one exception, which I will describe later).    
 
At some point we might want to use an initializer that we _think_ is a constant expression. We might want to use a variable in a context, where we **need** it to be constant. The program might prove us wrong, providing a value which we didn't expect. So, to be extra sure, we can prepend the variable definition with the _constexpr_ word. That way, should the variable be something other than a const expression, the compiler will mark the definition as an error. 
 
#### When can I use it?
 
You can use _constexpr_ with **literal types**. A literal type is a type simple enough, to have literal values. 
>Arithmetic, reference, and pointer types are literal types.

_std::string_ is **not** a literal type. If we define a class, its type is also not a literal type (although there are _constexpr_ classes).    
We can initialize a _constexpr_ pointer with limited number of objects, that is:

 * `nullptr` literal or `0` literal
 *  an object that remains at a fixed address 
 
Local variables do not reside at a fixed address - they cannot be defined as constant expressions. On the other hand `static` variables do. Therefore, a _constexpr_ pointer can be initialized with (and a reference can be bound to) its address:
{% highlight c++ linenos %}
void f() 
{
	 static int i = 10;
	 int j = 0;
	 constexpr int *pi = &i;		// OK - i is static
	 constexpr int *pj = &j;		// Error - j is not a constant expression
	 constexpr int &ri = i;		 // OK  
	 constexpr int &rj = j;		 // Error!
};
{% endhighlight %} 

#### Pointers and _constexpr_

Our beloved specifier **imposes** a top-level const on the pointer it applies to, and **not** on the object it points to. Therefore, in the example:
{% highlight c++ linenos %}
const int *pa = nullptr;
constexpr int *pb = nullptr;
{% endhighlight %}
the first pointer is a pointer to const. That means we cannot change the state of the object using the pointer.    
The second pointer is equivalent to `int * const pb = nullptr` - a const pointer. We cannot change what it points to. Do you see the difference?    

A constexpr pointer can be used with both - const and non-const object. One thing to remember is that such a pointer is a const pointer - we cannot assign a new address value to it.

{% highlight c++ linenos %}
const int i = 0;
int j = 1;
constexpr const int *pi = &i;    // const pointer to const int
constexpr int *pj = &j;		 	// const pointer to int 
pj = &i;						  // Error - trying to change a read-only object
{% endhighlight %}

### _constexpr_ functions
In the first example I used the `get_file_no()` function. We could use it to initialize a constant expression if, and only if, it is a _constexpr function_. Such a function can be used to initialize an expression if:

 * its body contains exactly **one** return statement
 * its return type and the the type of each parameter is a literal type
 
Such a function is implicitly _inline_.    
The function body may contain other (than the mentioned `return`) statements, as long as its a _null_ statement, type alias, or a _using_ declaration. These statements do not generate any action at run time. 
{% highlight c++ linenos %}
constexpr int f() 
{ ; 
  typedef int myint; 
  using myint2 = int; 
  return 2; 
};
{% endhighlight %}

Furthermore, the function **doesn't have to return a constant expression**. We can use such functions in different contexts. Let's write a function with one parameter, and use it to define an array, and initialize a plain `int` variable. 
{% highlight c++ linenos %}
constexpr int f(int p) 
{  
  return p + 2; 
};
{% endhighlight %}
You can see that it has only one `return` statement, and one parameter. So for it to be a constant expression, the parameter also has to be one (if you recall, `2` is also a constant expression). 
{% highlight c++ linenos %}
int i = 2;
int j = 5;
int first_array[f(10)];	 // OK - 10 is a constant expression
int second_array[f(j)];		// Error - the parameter is not a constant expression
j = f(i);					// OK - the function doesn't have to be constexpr
{% endhighlight %}
To define the `first_array`, we need to pass a _constexpr_ to the `f()` function. The `second_array`'s size parameter is a plain `int`, so it will not compile. The size needs to be determined at compile time.    
And what about the `j` variable? Its initializer doesn't need to be constant. It is a simple variable, initialized with the value of `i` - 2.

There's one more thing. _constexpr_ functions can be defined **multiple** times, like `inline` functions, but each definition must be exactly the same.

#### But wait, this code compiles!

If you are using GCC it is possible that the example above will compile. GCC supports C-style [Variable Length Arrays](https://gcc.gnu.org/onlinedocs/gcc/Variable-Length.html). The compiler will warn you if you use `-Wvla` or `-pedantic` flags. Normally, the C++11 would not allow it.

### _constexpr_ classes

A class is a literal class if it's an aggregate class whose data members are all of literal type. A non-aggregate class is literal if it follows certain restrictions.

#### Aggregate classes

A class is aggregate if:
 
 * all of its members are of literal type
 * it doesn't define any constructors
 * it has no in-class initializer
 * it has no base classes or _virtual_ functions
 
This is an example of an aggregate class:
{% highlight c++ linenos %}
class MyClass 
{
	int a;
	std::string b;
};
{% endhighlight %}

We can use objects of such classes as constant expressions.

#### Non-aggregate classes

A non-aggregate class is literal if:

 * all of its members are of literal type
 * it has at least one _constexpr_ constructor
 * if it has an in-class initializer for a member, the initializer must itself be a constant expression
 * it uses a default destructor (either synthesized by the compiler, or defined as `= default`)
 
In the previous paragraph, I mentioned that a _constexpr_ function can have only one `return` statement. Constructors don't have a return type. This has one important implication: the constructor's body is usually empty.

_constexpr_ constructors have to initialize all members of the class. They can do this either by using a _constexpr_ constructor for a member, or by using constant expressions. An example of a literal class:

{% highlight c++ linenos %}
class CEClass 
{
public:
	constexpr CEClass(int mode) : cmode(mode) { };  // one constexpr constructor
	constexpr int getMode() { return cmode; };		 // constexpr member
	~CEClass() = default;							  // default destructor
private: 
	int cmode;
	int init = 2;						// initializer is a constant expression
};

constexpr CEClass ob1(1);
CEClass ob2(1);
int arr[ob1.getMode()];		 // OK - getMode is a constexpr member
int arr2[ob2.getMode()];	 	 // Error - ob2 is not constexpr
{% endhighlight %}


### Summary

_constexpr_ can help us in two things:
 
 * by adding the specifier to a variable, we can be sure that  at run time the value will always be the same
 * the values are computed at compile time. This has an impact on the performance. Using _constexpr_ objects and their member functions can eliminate the overhead that in some cases could slow down the program
 


