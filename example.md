### What C++ Developers Should Know About Globals

#### (and the linker)



### Globals
 - I've never looked at a codebase and thought: gee, this has fewer globals than it should
 - Nothing in this talk is an endorsement of using lots of globals
 - Try to overcome your global addiction. Or at least do globals in private, not at your desk
 - Still have uses:
   - Logging
   - Intrusive performance benchmarking
   - Polymorphic factories


### Globals vs Singletons
 - Do not confuse them!
 - A singleton is a *class* that can have *at most* one instance
 - A global is a *variable* that has global scope
 - These are orthogonal, you can have them in any combination
 - Singletons are even more evil than globals (but still have uses)



### Compiler vs Linker

 - C++ devs like the compiler: there's a detailed spec, there's new language iterations...
 - But even code written in the 2017 edition of C++, could perhaps be linked by a linker from 1977
 - This leads to weirdness



### When good code goes bad


### One library

```
// static.h
#pragma once

#include <string>

extern std::string g_str;

// static.cpp
#include "static.h"

std::string g_str = "string too long for short string"
                    "optimization";
```


### Another library

<section><pre><code data-trim data-noescape>
// dynamic.h
#pragma once

#include "static.h"
#include &lt;string>

std::string&amp; globalGetter();

// dynamic.cpp
#include "dynamic.h"
#include "static.h"

std::string& globalGetter() {
    return g_str;
}
</code></pre></section>


### An executable

```
// main.x.cpp
#include "static.h"
#include "dynamic.h"

#include <iostream>

int main() {
    std::cerr << g_str << "\n";
    std::cerr << globalGetter() << "\n";
    return 0;
}
```


### Putting it together

```bash
# Compile static translation unit
clang++ -std=c++14 -c -fpic static.cpp
# Produce a static library
ar rcs libstatic.a static.o
# Compile dynamic translation unit
clang++ -std=c++14 -c -fpic dynamic.cpp
# Produce dynamic library, linking against static
clang++ -shared -o libdynamic.so dynamic.o -L./ -lstatic
# Compile main executable translation unit
clang++ -std=c++14 -c main.x.cpp
# Link executable against shared and static libraries
clang++ -L./ -Wl,-rpath=./ main.x.o -lstatic -ldynamic
```


### This program segfaults!
 - It may seem contrived, but with enough developers with a modicum of freedom,
   this example is very commmon
 - a class static constant string seems innocent, right?


### What's happening
Let's try and change our string class to something else:
```
struct Informer {
  Informer() {
    std::cerr << "ctor " << this << "\n";
  }
  Informer* get() { return this; }

  ~Informer() {
    std::cerr << "dtor " << this << "\n";
  }
};
```

We'll also print addresses in main instead of the string itself.


### What now?

New output:

```
ctor 0x602193
ctor 0x602193
0x602193
0x602193
dtor 0x602193
dtor 0x602193
```



### Detour: How does linking work?

 - Linkers march through the link line, left to right
 - Each target/library provides some symbols, and needs others
 - Pick up needed symbols at each encountered target/library
 - Each time a provided symbol matches one that's needed:
   - If static, copy and paste the assembly into target
   - If dynamic, politely say that the loader will get this


### Examining the symbol table

```
~/D/g/informer ❯❯❯ nm --demangle main.x.o

                 U __cxa_atexit
0000000000000000 t __cxx_global_var_init
                 U __dso_handle
                 U g_informer
0000000000000050 t _GLOBAL__sub_I_main.x.cpp
0000000000000000 T main
                 U globalGetter()
0000000000000000 W Informer::get()
                 U std::ostream::operator<<(void const*)
                 U std::ios_base::Init::Init()
                 U std::ios_base::Init::~Init()
                 U std::cerr
```



### Implications for our example
 - The shared library gets a copy of the object code for `g_informer`
 - So does the executable (because we linked with static first)
 - What if we change link order?

```
# Link executable against static and shared libraries
clang++ -L./ -Wl,-rpath=./ main.x.o -ldynamic -lstatic
```
New output:
```
ctor 0x601170
0x601170
0x601170
dtor 0x601170
```
<aside class="notes">
  Left align new output. Use strike through to emphasize changed order of linking
</aside>


### Where is global initialized?
#### Shared Library: always

```
~/D/g/informer ❯❯❯ objdump -CS libdynamic.so

0000000000000ab0 <__cxx_global_var_init.1>:
  ...
  callq  a00 <Informer::Informer()@plt>
  mov    0x20152d(%rip),%rdi
                 # 201ff8 <_DYNAMIC+0x230>
  mov    0x2014fe(%rip),%rsi
                 # 201fd0 <_DYNAMIC+0x208>
  lea    0x20157f(%rip),%rdx
                 # 202058 <__dso_handle>
  callq  9a0 <__cxa_atexit@plt>
```


### Where is global initialized?
#### Executable: sometimes

```
~/D/g/informer ❯❯❯ objdump -CS a.out

0000000000400a90 <__cxx_global_var_init.1>:
  ...
  lea    0x2016f4(%rip),%rdi
                # 602193 <g_informer>
  callq  400c70 <Informer::Informer()>
  lea    0x215(%rip),%rdi
                # 400cc0 <Informer::~Informer()>
  lea    0x2016e1(%rip),%rsi
                # 602193 <g_informer>
  lea    0x2015af(%rip),%rdx
                # 602068 <__dso_handle>
  callq  400970 <__cxa_atexit@plt>
```



### Hard to misuse
 - So we get segfaults conditional on linking order!
 - Could blame it on mixing shared + static linking
 - Not a good/constructive strategy in many environments
 - Better idea: Write code that *always* works


### What to do?
 - C++17 brings us the keyword `inline`

```
// static.h
#pragma once

#include <string>

extern std::string g_str;

// static.cpp
#include "static.h"

inline std::string g_str = "...";
```


### Us peons not yet on 17?
#### Lazy (really) solution
```
// static.h
#pragma once

#include <string>

std::string& g_str() {
  static std::string s = "...";
  return s;
}
```
 - not backwards compatible (need extra `()`)
 - worse performance
 - laziness means work may happen at inconvenient times
 - not good when multiple globals involved (later)


### Us peons not yet on 17?
#### Stop being lazy
```
// static.h
#pragma once

#include <string>

namespace detail {
std::string& g_str() {
  static std::string s = "..."
  return s;
}
}
static auto& g_str = detail::g_str();
```
 - solves most issues of previous approach
   (unless you really want to be lazy)


### Us peons not yet on 17?
#### Marginally faster
```
// static.h
#pragma once

#include <string>

namespace detail {
template <class=void>
struct my_globals {
    std::string g_str;
};
template <>
std::string my_globals<>::g_str = "...";
}
static auto& g_str = detail::my_globals<>::g_str;
```
 - weird but wait: template classes have always needed
   `inline`
 - linker must know how to emit unique symbol already!
 - produces identical code to 17 `inline` keyword



### Interdependent Globals
 - When globals depend on other globals, things can get nasty
 - To prevent nastiness, I will convince you of three things:
   - Always define globals in the header
   - Avoid laziness in globals
   - Always include headers defining globals, in the header


### Define globals in the header
 - We know from Static Initialization Order Fisaco (SIOF) that the initialization order between
   different translation units is unspecified
 - If your global is defined in `.cpp`, it will be initialized as part of its own TU
 - This means that it is *never* safe for a client global to use your global in its constructor/destructor


### Avoid laziness in globals
An example where laziness can cause conditional segfaulting


### Include global-defining headers in headers
```
// Foo.h
struct Foo {
    Foo();
};

inline Foo f;

// Foo.cpp
#include <iostream>

Foo::Foo() {
    std::cerr << "hello world\n";
}

int main() {
    return 0;
}
```
Petitioning to have this be the new official C++ hello world program. It prints:
```
bash: line 7:  5940 Segmentation fault      (core dumped) ./a.out
```


### The DAG of globals
 - All of the header files in a program form a DAG, which is very useful

<table>
<thead>
<tr>
	<th>Guideline</th>
	<th>Guarantee</th>
</tr>
</thead>
<tbody>
	<tr>
		<td>Always define globals in the header </td>
		<td>All global definitions present in DAG</td>
	</tr>
	<tr>
		<td>Avoid laziness in globals</td>
		<td>Initialized when its header is read</td>
	</tr>
	<tr>
		<td>Always include headers defining globals, in the header</td>
		<td>Headers of a globals dependency read before its own header</td>
	</tr>
</tbody>
</table>

 - Net result: complete avoidance of SIOF
 - Cost:
   - Tiny, tiny amount of link time (more symbols in more places)
   - Can't have circularly dependent globals
