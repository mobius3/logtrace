The logtrace logging library
============================

**logtrace** is a C++11 *single-header* library for adding logging capabilities to your application. It features multiple verbosity levels, nested function tracing (indented information when entering or leaving a function) and inclusive and exclusive filtering. It also features Android [logcat] support :)

It was designed to be as unobtrusive as possible and to keep your code free of such `LOGLIBRARY_LOGMESSAGE_ERROR("hi?")` macros. No explicit inicialization required.

Installation
------------
Drop logtrace.h into your project where your source files can see it.

Using
-----

Place a `#include "logtrace.h` wherever you want to use it.

By default, logging is **disabled** and all **logtrace** related calls will do nothing. To activate them, you'll need to define the `LOGTRACE` compile-time macro. You can do that by placing `#define LOGTRACE` *before* including the header, or you can pass that as a flag to your compiler (consult your compiler help to see how to do it).

Quickstart
----------

To log your stuff, create an instance of the `logtrace` class and use it to print messages, like this:



    #define LOGTRACE
    #include "logtrace.h"

    int main() {
        logtrace trace;
        trace("This is an information message");
        trace.w("This is a warning message");
        trace.e("This is an error message");
        return 0;
    }


Because the trace object was declared without parameters, its [log level](#log-levels) is *information* and its [module](#modules) is not set. The output is something like this:

    E : This is an error message
    
Every message that has a log level below the global level (which defaults to **error**) will not appear.

Building the message
----------

Each message logging function supports as many parameters as you want. The **logtrace** class will pass them directly to the output stream (which defaults to [`cout`][1]), so you might want to write "[output operators]" for your objects if you intend to trace them.

For instance, you can have something as:

    logtrace trace;
    int roses = 0xff0000, violets = 0x0000ff;
    trace.e("Roses are", roses, "violets are", violets);

Output:

    E : Roses are 16711680 violets are 255
    
Modules
-------

You can define **modules** for each trace object, so that later you can [filter](#filters) messages by them. To do that, simply place the module name when instantiating the **logtrace** class:

    int main() {
        logtrace trace("main"); /* trace messages from the "main" module */
        ...
    }

This would change the [first example](#quickstart)'s output to:

    E main: This is an error message
    
You can change the module of the trace object anytime by calling its `module()` method:

    trace.module("main"); /* this object module is now "main" */



Log levels
----------

**logtrace** has four log levels. From the least to most verbose (or from the most to least severe), these are:

- *error*
- *warning*
- *information*
- *call tracing*

The default **global** level is *error*, this means that only error messages will appear. You can change the global verbosity level of **logtrace** by using the following code:


    logtrace::globalverb() = logtrace::error;       /* only error messages */
    logtrace::globalverb() = logtrace::warning;     /* warning and error messages */
    logtrace::globalverb() = logtrace::info;        /* information, warning and error messages */
    logtrace::globalverb() = logtrace::maximum;     /* everything, including function tracing */

When instantiating your trace object you can set its default log level:

    logtrace trace("main", logtrace::warning);
    
Every message through this particular **logtrace** object will have the *warning* level, unless specified otherwise by using the `.i()`, `.w()` or `.e()` functions. If the global level is bigger than *warning*, all messages from this object will be shown.


If we changed the code in the [modules section](#modules) to have `logtrace::globalverb() = logtrace::warning;` before instantiating logtrace, its output would be:

    W main: This is a warning message 
    E main: This is an error message 

As you can see, log levels are incremental. The last level, `logtrace::maximum`, is very verbose and useful to trace function calls. When set, this would be the output of the above example:

    [ In main
      I main: This is an information message 
      W main: This is a warning message 
      E main: This is an error message 
    ] Leaving main
    
    
This is useful for recursive function calls and also to easily spot how deep in your code the funcion gets called. Consider:

    #define LOGTRACE
    #include "logtrace.h"
    
    void foo(int i);
    void bar(int i) {
      logtrace trace("bar");
      if (i == 0) trace("Recursion stop.");
      else foo(--i);
    }
    
    void foo(int i) {
      logtrace trace("foo");
      if (i == 0) trace("Recursion stop.");
      else bar(--i);
    }
    
    int main() {
        logtrace::globalverb() = logtrace::maximum;        
        logtrace trace("main");
        foo(2);
        return 0;
    }


This is the output:

    [ In main
      [ In foo
        [ In bar
          [ In foo
            I foo: Recursion stop. 
          ] Leaving foo
        ] Leaving bar
      ] Leaving foo
    ] Leaving main
    
    
Filters
-------

Besides changing the log level, you can set **logtrace** to print only messages from the modules you want from, using the exclusive and the inclusive filters. The inclusive filter will ignore everything except what its in there while the exclusive will print everything except for what it is in there.

These filters will make **logtrace** *completely* ignore what shouldn't be printed. This includes the tracing and error messages.

To set the ignore filter:
```cpp
logtrace::ignore().insert("foo"); /* will ignore messages from the 'foo' module */
```

The output of the previous example when ignoring messages from the `foo` module would be like this:

    [ In main
      [ In bar
      ] Leaving bar
    ] Leaving main


Similarly, to set the inclusive filter:
```cpp
logtrace::filter().insert("foo"); /* will print only messages from the 'foo' module */
```

Output:

    [ In foo
      [ In foo
        I foo: Recursion stop. 
      ] Leaving foo
    ] Leaving foo

Trace messages and helper macros
---------------------------------

You can also set a trace message when instantiating the logtrace object. It will be shown when entering and leaving the function, together with the module:

    logtrace trace("module", "trace");
    
Will show

    [ In module: trace
    ] Leaving module: trace

Making use of that feature, there are some helper macros that will declare a trace object for you and initialize it with a desired log level and will print the function name that they pertain to. These are

- `modinfo(module)`
- `modwarn(module)`
- `moderr(module)`

They will all create an object named `trace` with the specified module string. The *trace* message will be the function name. Replacing the `logtrace trace("foo")`, `logtrace trace("bar")` and `logtrace trace("main")` with the macro `modinfo()`, the output would be:

    [ In main: int main()
      [ In foo: void foo(int)
        [ In bar: void bar(int)
          [ In foo: void foo(int)
            I foo: Recursion stop. 
          ] Leaving foo: void foo(int)
        ] Leaving bar: void bar(int)
      ] Leaving foo: void foo(int)
    ] Leaving main: int main()
    
    
Output to a file
----------------

To output all the messages to a file, use the following code before you start logging stuff:

    logtrace::open("/path/to/log.txt");
    
The destination file will be truncated.
    
Android support
---------------

This library implements a streambuf that makes `cout` redirect its output to Android's [logcat] log viewer. This is automatically set when you use **logtrace** for the first time. This **streambuf** is largely based on the answer for [this][Android streambuf] StackOverflow question.

[1]: http://www.cplusplus.com/reference/iostream/cout/
[output operators]: http://www.parashift.com/c++-faq/output-operator.html
[logcat]: http://developer.android.com/tools/help/logcat.html
[Android streambuf]: http://stackoverflow.com/questions/8870174/is-stdcout-usable-in-android-ndk
