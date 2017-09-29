### What C++ Developers Should Know About Globals

#### (and the linker)



#### Or

### Things I never wanted to know about globals and linkers but was forced to find out anyway

Note:
I work on a trading team, in HFT
 - I'm not a build engineer; focus on low latency + modelling systems
 - bad part of me giving this talk: I'm not an expert in linkers, or globals
   (whatever that means)
 - good thing: neither are most C++ devs!
 - not going to tell you how awesome the linker is
 - just going to tell you how to avoid making me sad one day



### Globals
 - I've never looked at a codebase and thought: gee, this has fewer globals than it should
 - This ~~rant~~ talk is not an endorsement of using lots of globals
 - Still have uses:
   - Logging
   - Intrusive performance profiling
   - Polymorphic factories
   - Dealing with someone else's globals!

Note:
  All of the examples of useful logging have good excuses:
   - the first two don't affect behwhere did they hurt youavior, and you may want to switch
     them on/off seamlessly, so passing in loggers/profilers is a pain
   - polymorphic factory global: the price paid to allow self-registration


### Globals vs Singletons
 - Do not confuse them!
 - Singleton: a *class* that can have at most one instance
 - Global: a *variable* that has global scope
 - These are orthogonal, you can have them in any combination
 - Singletons are even more evil than globals (but still have uses)



### Compiler vs Linker

 - C++ devs like the compiler: there's a detailed spec, there's new language
   iterations...
 - But even code written in the 2017 edition of C++, could maybe be linked by a
 linker written before I was born
 - This leads to weirdness

<aside class="notes">
 - googling for compiler stuff: every conceivable question analyzed to the nth
 detail on SO by 8 people with 100K + rep
 - figuring out linker stuff: cricket
</aside>



### When good code goes bad


### A static library

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


### A shared library

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


### Building the example

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
# Link executable against static and shared libraries
clang++ -L./ -Wl,-rpath=./ main.x.o -lstatic -ldynamic
```


### This program segfaults!
 - It may seem contrived, but with enough developers with a modicum of freedom,
   this example is very commmon
 - a class static constant string seems innocent, right?


### What's happening?
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
```
int main() {
    std::cerr << g_informer.get() << "\n";
    std::cerr << globalGetter().get() << "\n";
    return 0;
}
```


### What now?

<section style="text-align: left;">
Running the program doesn't segfault (since we aren't doing anything
interesting) and prints:
</section>
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


### Examining a symbol table

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

<aside class="notes">
 U - undefined symbol "needed"
 W - defined weak symbol, because Informer::get is inline
 T - regular defined symbol
</aside>


### Implications for our example
 - The shared library gets a copy of the object code for `g_informer`
 - So does the executable (because we linked with static first)
 - What if we change link order?

```
# Link executable against shared and static libraries
clang++ -L./ -Wl,-rpath=./ main.x.o -ldynamic -lstatic
```
<div class="fragment">
<section style="text-align: left;">
New output:
</section>
<pre><code class="cpp">
ctor 0x601170
0x601170
0x601170
dtor 0x601170
</code></pre>
</div>

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


### Caveat coding
> One thing should be noted, though. If your application consists of more than
> one module (e.g. an exe and one or several dll's) that use Boost.Log, the
> library must be built as a shared object

<aside class="notes">
Boost documentation doesn't say (that I could see) why this is, but it does
ensure that "a global logger instance will be unique even across module
boundaries", so I suspect this is related
</aside>


### How about this?
> Your application can build against Boost.Log however you want and it will work
> regardless



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
 - Not backwards compatible (need extra `()`)
 - Worse performance
 - Work may happen at inconvenient times


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
 - Solves most issues of previous approach
 - Sometimes laziness can be an upside, though


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
<aside class="notes">
 - weird but wait: template classes have always needed
   `inline`
 - linker must know how to emit unique symbol already!
 - produces identical code to 17 `inline` keyword
</aside>



### Interdependent Globals
 - When globals depend on other globals, things can get nasty
 - Static Initialization Order Fiasco (SIOF)
 - Sadly: there is no silver bullet
 - We can spread some knowledge around though


### Best: avoid them!
 - Avoiding singletons helps

```
class IntrusiveProfileData {
  IntrusiveProfileData() {
    Logger::instance().log("Interdependent globals!");
    m_logger.log("Special logger instead!");
  }
};
```

<aside class="notes">
 - Avoiding singleton helps because we can use the class
 normally, by creating our own instance
 - Singletons lock useful functionality in only a global
   form
</aside>


### Define globals in the header
 - Initialization order between different translation units is unspecified
 - If your global is defined in `.cpp`, it will be initialized as part of its own TU
 - This means that it is not safe for a client global to use your global in its constructor/destructor
 - Unless you provide a function to force initialization (similar to lazy)


### Beware laziness and destructors
#### A noisy logger
```
#include <iostream>

struct Logger {
    Logger() { std::cerr << "Constructed "; }
    void log() { std::cerr << "logging " ; }
    ~Logger() { std::cerr << "Destructed "; }

    static Logger& instance() {
        static Logger l;
        return l;
    }
};
```


### Beware laziness and destructors
#### A confused program

```
struct Foo {
    ~Foo() { Logger::instance().log(); }
};

Foo f;

int main(int argc, char *argv[])
{
    Logger::instance().log();
    return 0;
}
```

<section style="text-align: left;">
Output:
</section>
```
Constructed logging Destructed logging
```

<aside class="notes">
- this is a valid order for program to execute; Foo TU then main
- f global gets created first, then we do logging in main
- so logger gets destroyed first, since constructed second
- f tries to log in dest, logger gone!
</aside>


### Include global-defining headers in headers
```
// Foo.h
struct Foo { Foo(); };

inline Foo f;

// Foo.cpp
#include <iostream>  // defines cerr global

Foo::Foo() { std::cerr << "hello world\n"; }

int main() { return 0; }
```
<section style="text-align: left;">
Output:
</section>
```
bash: line 7:  5940 Segmentation fault
      (core dumped) ./a.out
```
<aside class="notes">
- this is a valid order for program to execute; Foo TU then main
- f global gets created before cerr global
- but f global calls to Foo constructor, which uses cerr!
</aside>



### Conclusions

 - Know the very basics of linkers/symbol tables
 - Create all non-trivial globals using one of the techniques
   discussed to ensure they are safe
 - Prefer to do things through the header when globals are involved
 - Think very carefully about (transitive) dependencies between globals
