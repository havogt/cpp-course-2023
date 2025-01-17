---
marp: true
theme: cscs
# class: lead
paginate: true
backgroundColor: #fff
backgroundImage: url('../slides-support/common/4k-slide-bg-white.png')
size: 16:9
---

# **C++ Course**
![bg cover](../slides-support/common/title-bg3.png)
<!-- _paginate: skip  -->
<!-- _class: titlecover -->
<!-- _footer: "" -->

## Templates Part I

### everyone/JB/Mauro

--- 

## Overview
* Template basics, function and class templates
* Overloads
* Argument deduction
* Partial specialization
* Default template parameters
* Non type template parameters
* SFINAE and enable_if
* Class template type deduction
* Basic intro to meta-programming _ integral constant
* Alias templates
* Variadic templates
* Fold Expressions
* Structured Bindings
* ->Tuples


--- 

## Functions and Classes

* A function is defined as return type, name, and arguments

```c++
    int add(int x, int y) { return x+y; }

    int main() { add(65, 35);
```

* A class is defined as a name after class (or struct)

```c++
    class name1;  // by default members are private
    struct name2; //                        public 
```

* Definition contains type, data and function members

```c++
    class name1 {
        // insert interesting things here ...
        int thing1;
        int thing2(float);
    };
    // here we instantiate a variable of type 'name1'
    int main() { name1 x; }
```

--- 

## Function templates

* A function template is not a function
  * It needs to be *instantiated*
  * Mental model: substitute type text to template argument

```c++
    template <typename T>
    void foo(T x) {
        std::cout << x << "\n";
    }

    int main() {
        foo<int>(65);
        foo<char>(65);
        foo(3.14159); // Argument Deduction
        foo(std::string("string"));
    }
```

---

## Overloads

* Among the options the most specialized is chosen
  * Include ADL available candidates

```c++
    template <typename T> 
    void foo(T x) {
        std::cout << x << "\n";
    }
    
    void foo(std::string const& x) {
        std::cout << "ooh! a string! " << x << "\n";
    }
    
    int main() {
        foo<int>(65);
        foo<char>(65);
        foo(3.14159); // Argument Deduction
        foo(std::string("string"));
    }
```

--- 

## Order Matters

```c++
    template <typename T, typename U>
    void foo(T, U) {}

    int main() {
        foo<std::string, double>("hello", 4.5);
        foo<std::string>("hello", 4.5);
        foo<double>("hello", 4.5);
    }
```

`cannot convert '"hello"' (type 'const char [6]') to type 'double'`

--- 

## Template Argument Deduction

* To instantiate a template all the arguments must be known!
* Sometime they can be deduced

```c++
    template <typename To, typename From> 
    To convert(From f);   

    void g(double d) { 
        int i = convert<int>(d);    // To = int,  deduced From = double
        char c = convert<char>(d);  // To = char, deduced From = double
        int(*ptr)(float) = convert; // To = int,  From = float
    }
```

```c++
    template <typename From, typename To> 
    To convert(From f);   

    void g(double d) { 
        int i = convert<double, int>(d);   // To = int,  From = double
        char c = convert<double, char>(d); // To = int,  From = double
        int(*ptr)(float) = convert;        // To = int,  From = float
    }
```

--- 

# Class templates

* A class template is not a class
  * It needs to be instantiated
  * Only then can an object of that type exist

```c++
    template <typename T> 
    class templ_name {
        T member;
        T operator()(T a, T b) const {…}
    };

    int main() {
      using class_name = templ_name<int>;
      class_name x;
    }
```

--- 

## Partial specializations

* For template functions - these are really just **OVERLOADS**
  * The more specialized is chosen

```c++
    template <typename T, typename U>
    struct X {};                        // 1 (Primary template)

    template <typename T>
    struct X<T, int> {};                // 2 (Specialization of arg1)

    template <typename U>
    struct X<float, U> {};              // 3 (Specialization of arg2)

    int main() {
        X<char,  double> a; // choose 1 
        X<char,  int>    b; // choose 2
        X<float, double> c; // choose 3
        X<float, int>    d; // all 3 match. what to do ???
    }
```

`error: ambiguous partial specializations of 'X<float, int>'`

--- 

## Full specialization

```c++
    template <typename T, typename U>
    struct X {};                        // 1 (Primary template)

    template <typename T>
    struct X<T, int> {};                // 2 (Specialization of arg1)

    template <typename U>
    struct X<float, U> {};              // 3 (Specialization of arg2)

    template <>
    struct X<float, int> {};            // 4 (Full Specialization)

    int main() {
        X<char,  double> a; // choose 1 
        X<char,  int>    b; // choose 2
        X<float, double> c; // choose 3
        X<float, int>    d; // choose 4
    }
```

--- 

## Pattern Matching

```c++
    template <typename T, typename U>
    struct X {};              // 1

    template <typename W, typename T, typename U>
    struct X<W, X<T,U>> {};   // 2

    template <typename T>
    void foo(X<T,T>) {}       // F1

    int main() {
        X<int, X<int, float>> b;         // specialization<int, primary>
        X<int, X<char, X<int, void>>> c; // specialization<int, primary>
        X<double, double> a; 
        foo(a); // ok F1
        foo(c); // no :(
    }
```

`candidate template ignored: deduced conflicting types for parameter 'T' ('int' vs. 'X<char, X<int, void>>') -> void foo(X<T,T>) {}`

--- 

## Pattern Matching

```c++
    template <typename T, typename U>
    struct X {};              // 1

    template <typename W, typename T, typename U>
    struct X<W, X<T,U>> {};   // 2

    template <typename T>
    void foo(X<T,T>) {}       // F1

    template <typename T, typename U>
    void foo(X<T,U>) {}       // F2

    int main() {
        X<int, X<int, float>> b;         // specialization<int, primary>
        X<int, X<char, X<int, void>>> c; // specialization<int, primary>
        X<double, double> a; 
        foo(a); // ok F1
        foo(c); // yay \o/ F2  
    }
```

--- 

## Default template arguments

* Template argument can be defaulted
* From C++11 this is also possible on function templates
* Default args can only be to the right the arguments on the right **if not deduced**

```c++
    template <typename T, typename Result=char>
    Result foo(T x) {
        return static_cast<Result>(x);
    }


    int main() {
        std::cout << foo(65) << "\n"; 
        std::cout << foo<int>(65) << "\n";
        std::cout << foo<int, int>(65.3) << "\n";
    }
```

`A A 65`

--- 

## Default args (... on the right)

* No args on the right of the Default args ___unless they can be deduced___

```c++
    template <typename T, typename Result=char, typename Another>
    Result foo(T x, Another a) {
        std::cout << typeid(T).name() << " " << typeid(Another).name() <<" ";
        return static_cast<Result>(x);
    }

    int main() {
        // T deduced as int, Another deduced as float
        std::cout << foo(65, 3.5) << "\n"; // Result = char
        // but like this we explicitly state
        std::cout << foo<int, char, float>(65, 3.5) << "\n"; // Result = char
        // in this case Another is deduced as float 
        std::cout << foo<int, float>(65, 3.5) << "\n"; // Result = float
    }
```

`i d A`
`i f A`
`i d 65`
* 1 The compiler knows `T` and `Another` so it is ok with defaulting `Result`
* 2 We tell the compiler all 3
* 3 The compiler can deduce `Another` but we forced `Result` to be float

--- 

## Non type template arguments

* Integral values can be used as template arguments
  * also char, int, pointers and arbitrary types
* Non type templates can also have default values

```c++
    template <int I>
    struct static_int {
        // a data value hardcoded for this type 
        static constexpr int value = I;
    };

    int main() {
        std::cout << static_int<5>::value;
        std::cout << static_int<6>::value;
    }
```

* Be aware that `static_int<5>` and `static_int<6>` are not the same *type*

--- 

## Default Template arguments for classes

```c++
    template <typename T=double, int Size=10>
    class my_container {};

    int main() {
        // x is a my_container of 10 doubles
        // <> required because container is templated and needs default types 
        my_container<> x; 

        // how does the compiler know what type the elements are?
        my_container<100> y; // ERROR

        // STL uses this approach for std::array (replaces double[N] syntax from C)
        std::array<double, Size> lovely;

        // Nicer than the old way (no size info carried)
        double deprecated[Size]
    }
```

---

## Evaluation order

* Template arguments are evaluated left to right
* So we can get things from the first and use them in the second, third ...

```c++
    template <typename T, typename U = typename T::type, int X = U::value>
    struct Order {};

    struct B {
        using type = B;
        static const int value = 100;
    };

    int main() {

        Order<B> x;
    }
```

---

## Evaluation order (wrong)

* You can't use a derived type/value before it is defined
* Swapping U/X breaks the compilation

```text
    template <typename T, int X = U::value, typename U = typename T::type>
    struct Order {};

    struct B {
        using type = B;
        static const int value = 100;
    };

    int main() {

        Order<B> x;
    }
```

---

## Default Template Arguments and Specializations

```c++
template <typename T=double, int Size=10>
struct my_container {};                   // always searched first

template <typename T>
struct my_container<T, 10> {};            // specialization 1

template <typename T>
struct my_container<T, 15> {};            // specialization 2

int main() {
    my_container<char, 10> z;             // uses specialization 1
    my_container<float,30> u;             // uses primary
    my_container<int,  15> v;             // uses specialization 2
    my_container<int>      y;             // uses specialization 1 (Size=10)
    my_container<>         x;             // primary (both default)

}
```

---

## SFINAE
* Substitution Failure Is Not An Error
  * When looking for specialization some substitution may fail
    * It's the backbone of templated code
    * Without it, nothing would work (compile)
```c++
    // some generic struct that by default will use int
    template <typename T, typename U = int>
    struct X {};

    // a specialization for the struct *IF* it has an extra_type definition
    // When extra_type is not there, clearly this would be a compilation 'fail'
    template <typename T>
    struct X<T, typename T::extra_type> {};
```

---

## Specialization in Action
```c++
    template <typename T, typename U = int>
    struct X {
        X() {
            std::cout << "Primary " << typeid(T).name() << " "  << typeid(U).name() << "\n";
        }
    };

    template <typename T>
    struct X<T, typename T::extra_type> {      
        X() {
            std::cout << "Specialization " << typeid(T).name() << " " << typeid(typename T::extra_type).name() << "\n";
        }
    };

    struct A { using value_type = int; };
    struct B { using extra_type = int; };
    struct C { using extra_type = float; };
    struct D { using extra_type = char; };

    int main() {
        X<A> a;         // uses primary - A::value_type not a match
        X<B> b;         // uses specialization - B::extra_type = int
        X<C> c;         // uses primary - C::extra_type not int
        X<B,char> b1;   // uses primary - B::extra_type not char
        X<D,char> d;    // uses specialization, D::extra_type = char
    }
```

`Primary 1A` `Specialization 1B` `Primary 1C` `Primary 1B` `Specialization 1D`  
* use specialization If `T::extra_type` matches `U`

---

## SFINAE: `std::enable_if``

```c++
    template <typename T, typename Enable = void>
    struct A;

    template <typename T> // does not compile if is_same fails 
    struct A<T, typename std::enable_if<std::is_same<T, int>::value, void>::type> {
        A() { std::cout << "int!\n"; }
    };

    template <typename T> // // does not compile if is_same succeeds
    struct A<T, typename std::enable_if<!std::is_same<T, int>::value, void>::type> {
        A() { std::cout << "not int\n"; }
    };

    int main() {
        A<int> a1;
        A<float> a2;
    }
```

`int!` `not int`  

* The failed version will not compile so the good version is selected - SFINAE

---

## SFINAE: `std::enable_if_t` + `std::is_same_v` (less ___cruft___)

```c++  
    template <typename T, typename Enable = void>
    struct A;

    template <typename T> // does not compile if is_same fails 
    struct A<T, std::enable_if_t<std::is_same_v<T, int>, void>> {
        A() { std::cout << "int!\n"; }
    };

    template <typename T> // // does not compile if is_same succeeds
    struct A<T, std::enable_if_t<!std::is_same_v<T, int>, void>> {
        A() { std::cout << "not int\n"; }
    };

    int main() {
        A<int> a1;
        A<float> a2;
    }
```

* Just like the previous version, but slightly easier to understand

---

## SFINAE: `void_t`
```c++
    template <typename ...T>
    using void_t = void;

    template <class C>
    using can_add = decltype(std::declval<C>() + std::declval<C>());

    template <typename T, typename VOID=void>
    struct sum {
        static T add(T const&,T const&) { return -1; }
    };

    template <typename T>
    struct sum<T, void_t<can_add<T>>> {
        static T add(T const& a, T const& b) {
            return a+b;
        }
    };    
```

---

## An example 1/6

* my_container is not a template
  * my_container is not flexible

```c++
    struct my_container {
        using value_t = int;
        using container_t = vector<value_t>;
        container_t C;

        my_container(size_t s) : C(s) {}
    };
      
    my_container my_c(42);
```

---

## An example 2/6

* Customizing the basics

```c++
    template <typename VT>
    struct my_container {
        using value_t = VT;
        using container_t = vector<value_t>;
        container_t C;

        my_container(size_t s) : C(s) {}
    };

    my_container my_c<int>(42);
```

---

## An example 3/6

* Customizing the allocator

```c++
    template <typename VT, typename Allocator = allocator<VT>>
    struct my_container {
        using value_t = VT;
        using container_t = vector<value_t, Allocator>;
        container_t C;

        my_container(size_t s) : C(s) {}
    };

      
    my_container<int> my_c(42);
    my_container<int, std::allocator<int>> my_c(42);
    my_container<int, std::allocator<double>> my_c(42);
```

---

## An example 4/6 – Template Template Arguments

* Avoiding redundancies

```c++
    template <typename VT, 
              template <typename> class Allocator = allocator>
    struct my_container {
        using value_t = VT;
        using container_t = vector<value_t, Allocator<value_t>>;
        container_t C;

        my_container(size_t s) : C(s) {}
    };
      
    my_container<int> my_c(42);
    my_container<int, std::allocator> my_c2(42);
```

---

## An example 5/6

* Being completely explicit


```c++
    template <typename Container>
    struct my_container {
        using value_t = typename Container::value_type;
        using container_t = Container;
        container_t C;

        my_container(size_t s) : C(s) {}
    };

    my_container<vector<int, allocator<int>>> my_c(42);
```

---

## An example 6/6

* Customizing the whole

```c++
    template <typename VT, 
              template <typename, typename> class CT = vector,
              template <typename> class Allocator = allocator>
    struct my_container {
        using value_t = VT;
        using container_t = CT<value_t, Allocator<value_t>>;
        container_t C;

        my_container(size_t s) : C(s) {}
    };

    my_container<int> my_c(42);
    my_container<int, vector> my_c2(42);
    my_container<int, vector, allocator> my_c3(42);
```

---

## Class Template Type Deduction (C++17)

<table> 
<tr>
<th> A </th>
<th> B </th>
</tr>
<tr>
<td>

```c++
    template <typename T>
    class A {
        T x;
      public:
        A(T x) : x{x} {}
    };

    int main() {
        A<int> x(3);
        A y(3); // A<int>
    }
```

</td>
<td>

```c++
  template <typename F>
  struct B {
      F f;
      B(F&& f) : f{std::move(f)} {}

      template <typename ...Args>
      void call(Args&&... args) {
          f(std::forward<Args>(args)...);
      }
  };

  int main() {
      auto f = [](int i, int j) {cout << i+j << "\n"};
      B<decltype(f)> a{std::move(f)};

      B b{[](int i, int j) {cout << i+j << "\n";}};

      a.call(3,4);
      b.call(3,4);
  }
```

</td> </tr>
</table>
---

## Title

```c++
```

---

## Class Template Type Deduction (C++17)

<table> 
<tr>
<th> A </th>
<th> B </th>
</tr>
<tr>
<td>

```c++
    template <typename T>
    class A {
        T x;
      public:
        template <typename U>
        A(T x, U, int) : x{x} {}
    };

    int main() {
        A<int> x(3, 3.4, 7);
        A y(3, 3.4, 7); // A<int>
        A z{y}; // A<int>
    }
```

</td>
<td>

```c++
    template <typename T , typename U >
    A<T> make_A(T a , U x, int y) {
        return A<T>{a, x, y};
    } // Fictional function template

    template <typename T>
    A<T> make_A(A<T> a) {
        return A<T>{a};
    } // Fictional function template

    int main() {
        A<int> x(3, 3.4, 7);
        auto y = make_A(3);
        auto z = make_A(y);
    }
```

</td> </tr>
</table>
---

---

## Classes and Meta-Programming

* Kinds of members
  * [Static] Function
  * [Static] Data
  * Constexpr function
  * Static const/Constexpr data
  * Type (nested type names)

* Meta-programming is manipulating types
  * And static const/constexpr values
* The main mechanism is using class templates
* Single applications rarely need TMP
* Useful when building abstraction layers
  * E.g., header-only libraries

---

## Step Back

* Type members are possible
* Access like static members
  * `X::type_t<U>`
* Visibility rules as normal
* Constexpr variables visible at translation unit level

```c++
    class X {
     /*public/private/*/protected:
        using type = ...;

        template <typename T, ...>
        using type_t = ...;

        static const int a = 10;
        static constexpr int b =10;

        X(…);

        void member(…);
    };
```

---

## A simple example

* A complex way to say 
  ```asm
    movl $120, %esi
  ```

```c++
    template <unsigned char N>
    struct factorial {
        static constexpr unsigned value = N * factorial<N-1>::value;
    };

    template <>
    struct factorial<1> {
        static constexpr unsigned value=1;
    };


    int main() {
        std::cout << factorial<5>::value << "\n"; 
    }
```

[CompilerExplorer link](https://godbolt.org/z/9KPa4rqYe)

---

## A Convention for TMP

* TMP is still an “accident” in C++
* Boost::MPL conventions partially adopted by ISO C++
* A meta-function returning a type has a public ::type
* A meta-function returning a value has a public ::value
* Or both
```c++
           template <Arguments…>
           struct meta_function {
               using type = … ;
               static constexpr … value = … ; 
           };
```

* Type members with arbitrary names are called traits 

---

## std::integral_constant

```c++
template<class T, T v>
struct integral_constant {
    typedef T value_type;
    static constexpr value_type value = v;
};


//use:
static_assert(integral_constant<int, 7>::value == 7, “Error”)

```

---

## Building abstractions: std::rank example

* Type of `T[3][4]` is `(T[3])[4]`

```c++
    template<class T>
    struct rank 
        : public integral_constant<size_t, 0>
    {};

    template<class T, size_t N>
    struct rank<T[N]> // (int[3])[4] => T[4] where T = int[3]
        : public integral_constant<size_t, rank<T>::value + 1>
    {};

    template<class T>
    struct rank<T[]>
      : public integral_constant<size_t, rank<T>::value + 1>
    {}; 
```

---

## An Example with Types: If on Types

* If (boolean value) then type1, else type2
* A shorter version
```c++
    template <bool Pred, typename T1, typename T2>
    struct select_first {
        using type = T2;
    };

    template <typename T1, typename T2>
    struct select_first<true, T1, T2> {
        using type = T1;
    };
```

* But why do we need this?

---

## One (Maybe) Silly Example

```c++
    template <bool WithRef>
    typename select_first<WithRef, int&, int>::type
    with_ref (typename select_first<WithRef, int&, int>::type x) {
        x += 1;
        return x;
    }

    int main() {
        int x = 1;
        with_ref<true>(x);
        with_ref<false>(x);
    }
```

---

## Alias templates

* Typedefs on steroids!
  * But they are still really just typedefs
  * ```c++
      using integer_type = int;
      // same as
      typedef int integer_type;
    ```
  * But you can template them, which is really helpful
  ```c++
    // create an alias for a std::vector  
    template <typename T> 
    using my_type = std::vector<T>;

    // and then later on
    my_type<double> x(100);
  ```
  * 
* Many template arguments and defaults are allowed
  ```c++
    // create an alias for an STL-like container (which is itself templated) 
    template <template <typename, typename> typename T, typename U,
            template <typename> typename Alloc = std::allocator>            
    using my_container = T<U, Alloc<U>>;

    my_container<std::vector, int> x(100);
  ```

---

## Alias templates (it's an alias, not a new type)

```c++
template <typename T>
using my_vec = std::vector<T, my_allocator<T>>;

template <typename T> // defintion 1
std::size_t size_of(my_vec<T> const& v) { return v.size(); }

template <typename T> // defintion 2 - this is the same as 1
std::size_t size_of(std::vector<T, my_allocator<T>> const& v) {
    return v.size();
}
```

`error: redefinition of 'size_of' std::size_t size_of(std::vector<T, my_allocator<T>> const& v)`

---

## Alias templates (it's an alias, not a new type)

```c++
template <template <typename, typename> class V>  // 3
std::size_t size_of(V<int, std::allocator<int>> const& v) { return v.size(); }

template <template <typename> class V>            // 4
std::size_t size_of(V<int> const&v) { return v.size(); }

int main() {
    std::cout << size_of(my_vec<int>(23,0)) << std::endl; // uses 3, 4 doesn't match
}
```

* even though we've aliased the vector to a single template param, it really still has 2, using #4 we get  
  `template template argument has different template parameters than its corresponding template template parameter`

---

## Variadic Templates

* A template parameter pack accepts zero or more arguments!
  * Using `...` to express packs


```c++
    // a function tanking zero or more arguments 
    template <typename ...Ts>  
    void foo(Ts ...args) {}   // parameter pack

    // a class templated over zero or more types
    template <typename ...Ts>
    class A {};
```
```c++
    template <typename ...Ts>
    void foo(Ts ...args) { 
        function(args...);  // expands to arg1, arg2, arg3 ...
        pattern(args)...;   // expands to pattern(arg1), pattern(arg2) ... 
        function(&args...); // expands to function(&arg1, &arg2, &arg3 ...); 
    }
```
---

## Title

```c++
```

---

## Title

```c++
```

---

## Title

```c++
```

---

## Title

```c++
```

--- 

<!-- # Ideas

Variant with a visitor - using `auto`
 ```c++
    // get the i-th param from the variant algorithm list
    auto p = std::visit([=](auto const& obj) { 
        return obj.params[i];
    }, alg);
    // draw datasets in left column, params in right
    int column = std::visit([&](auto const& v) { 
        return get_column(v); 
    }, std::get<1>(p));  ```
``` -->

---

|First Image|Second Image|
|:-:|:-:|
|![First Image](https://images.pexels.com/photos/585759/pexels-photo-585759.jpeg?h=128)|![Second Image](https://images.pexels.com/photos/1335115/pexels-photo-1335115.jpeg?h=128)|

