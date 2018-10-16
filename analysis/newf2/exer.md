#### Exercise

Smart Pointers allow you to access the underlying pointer, but they don't make you do it very often.  However, it's common for those recently acquainted with smart pointers to ignore the overloaded operators out of either ignorance or a misplaced fear of performance degradation.  Change the following code using `.get()` to have a more natural syntax:
```c++
  std::unique_ptr<Bar> pBar(new Bar);

  foo(*(pBar.get()));

  pBar.get()->x();

  if(pBar.get())
  {
    //Do some stuff...
  }
```
Becomes:
```diff
   std::unique_ptr<Bar> pBar(new Bar);
-  foo(*(pBar.get()));
+  foo(*pBar);
-  pBar.get()->x();
+  pBar->x();
-  if(pBar.get())
+  if(pBar)
   {
     //Do some stuff...
   }
```


####Question 1:
When should `.get()` be used on a smart pointer?
####Answer 1:
Usually when code outside your control requires a raw pointer -- but not ownership of the pointer.  Herb Sutter has also argued that in modern code, a raw pointer may indicate that the caller should do nothing with ownership (i.e., don't delete it).  Our codebase probably isn't modern enough to allow this -- too many instances of raw pointers still conveying ownership..


####Question 2:
`unique_ptr::release` has a similar signature to `unique_ptr::get`.  How is it different?
####Answer 2:
That should be called when passing ownership of the pointer elsewhere.  For example, the GUI toolkit wxWidgets expects heap-allocated[^0] objects which it will delete once it's done using them.  `release` on `unique_ptr` means that `unique_ptr` will no longer hold that pointer, nor attempt to delete it -- otherwise it wouldn't be unique.  See the next training for gotchas when releasing `unique_ptr`s.


####Question 3:
It would be possible to allow the following code to compile.  What could happen if `shared_ptr` or `unique_ptr` were changed to allow it?
```c++
void foo1(int*) {}

std::shared_ptr<int> pInt(new int(5));
foo1(pInt);
```
####Answer 3:
Firstly, calling `foo1` as shown will fail, since you can't implicitly convert `shared_ptr<T>` to `T*`.  If such a conversion were added, `foo1(pInt)` would compile & work, but `delete pInt;` would merely compile -- `delete` would be called twice on that pointer.  (It would probably work until your software was unserviceable -- e.g., deployed on a satellite.  This is in keeping with software's reputation for being spiteful.)  Accidentally calling `delete` as shown could happen if you were trying to replace raw pointers with smart pointers in legacy code.


####Question 4:
It would be possible to allow the following code to compile.  What could happen if `shared_ptr` or `unique_ptr` were changed to allow it?
```c++
std::unique_ptr<int> pInt2 = new int(5);
```
####Answer 4:
The previous code would require `unique_ptr<T>` to be implicitly constructable from T*.  Suppose you had legacy code like this:
```c++
foo2(int* pInt)
{...}

...

int* pMyInt = new int(42);
foo2(pMyInt);
*pMyInt = 43;
```
Then, using shiny new smart_ptrs, you changed part, but didn't see the later code:
```c++
foo2(const std::unique_ptr<int>& pInt)
{...}

...

int* pMyInt = new int(42);
foo2(pMyInt);
*pMyInt = 43; //Earth-shattering kaboom!
```

As designed, the compiler will produce an error in the second case.  It's still possible to have exactly these problems, but you need to go out of your way to get them.  Code should be written to protect against Murphy (accidental misuse), not Machiavelli (deliberate attempts to misuse) [^1].  (This is not speaking in reference to sanitizing user input, which you should usually assume to be dangerous.)

***

####Question 5:
The following contains a mistake.  Fix it, and explain why it was wrong.
```c++
std::shared_ptr<int> p1(new int(42));
std::shared_ptr<int> p2(p1.get());
```
####Answer 5:
`p1` and `p2` will both attempt to delete the same pointer, without knowing the other exists.  A `shared_ptr` counts the number of pointers referencing the same pointer and deallocates that pointer once no one is referencing it.  With code like the previous, we get the following:
```c++
std::shared_ptr<int> p1(new int(42));//p1 references the new pointer, reference count = 1
std::shared_ptr<int> p2(p1.get());//p2 thinks it's the only guy looking at the pointer, reference count = 1
//(automatically called) p2.~shared_ptr<int>() -- deletes the pointer in p2
//(automatically called) p1.~shared_ptr<int>() -- Oops, p1's pointer was already deleted!
```

The correct way to ensure the reference counts are correct is to copy the shared_ptr, not just the underlying pointer:
```c++
std::shared_ptr<int> p1(new int(42));//p1 references the new pointer, reference count = 1
std::shared_ptr<int> p2(p1);//p2 references p1's pointer & p1's reference count, reference count = 2
//(automatically called) p2.~shared_ptr<int>() -- decrements the reference count ( = 1 )
//(automatically called) p1.~shared_ptr<int>() -- decrements the reference count to zero and deletes the pointer.
```


####Question 6:
The following contains a mistake.  Fix it, and explain why it was wrong.
```c++
std::unique_ptr<int> p3(new int[42]);
```
####Answer 6:
`std::unique_ptr<T>` will call `delete` on its pointer -- that means the pointer should be allocated using `new`.  If we want to use `new[N]`, we should store it in `std::unique_ptr<T[]>`, which will correctly call `delete[]`.  Furthermore, it offers `operator[]` and disallows `operator*` & `operator->`, which makes it less awkward to index elements after the first.


####Question 7:
When would you use a `unique_ptr` versus a `shared_ptr`?
####Answer 7:
Use `shared_ptr` when you know that ownership of the pointer must be shared in more than one place.  Note that _ownership_ == _responsability to deallocate the pointer_.
Use `unique_ptr` when you don't know that ownership of the pointer must be shared.


####Question 8:
Suppose you have a class which returns a pointer which the user _may_ wish to store in a `shared_ptr`.  What type should you return?
####Answer 8:
Return a `unique_ptr` to that type [^2].  `unique_ptr`s are convertible to `shared_ptr`s, so returning a `unique_ptr` prevents memory leaks without restricting how that pointer is used later.
Returning a raw pointer instead would certainly work, but goes against, e.g., Scott Meyers's advice to "make interfaces easy to use correctly and hard to use incorrectly."  (See Item 18 of Effective C++.)

The following code opens a handle to a file using the C-style `FILE*` interface
and closes the handle before returning.
```c++

  FILE* pFin = NULL;
  try
  {
    std::string filename("c:\\Windows\\win.ini");
    pFin = std::fopen(filename.c_str(), "r");
    if (!pFin)
      return;

    char data[10];
    if (std::fread(data, 1, 10, pFin) <= 0)
    {
      std::fclose(pFin);
      return;
    }

    std::fclose(pFin);
  }
  catch (...)
  {
    if (pFin)
      std::fclose(pFin);
    throw;  //rethrow after closing file
  }
```
We could change it to use
iostreams, but for this exercise, change the code to just use a smart
pointer (`std::shared_ptr`) to `FILE` with a custom deleter, so that the
`fclose()` call doesn't have to appear in so many places, and eliminate the
try-catch block.
####Answer:
```c++
    std::string filename("c:\\Windows\\win.ini");
    std::shared_ptr<FILE> pFin(std::fopen(filename.c_str(), "r"), &std::fclose);
    if (!pFin)
      return;

    char data[10];
    if (std::fread(data, 1, 10, pFin.get()) <= 0)
    {
      return;
    }
```
Notice that not only do the `fclose()` calls go away, but also the try/catch block goes away, since there's no longer a need to call `fclose()` and then re-throw.

***

####Question:
What could you do if your deleter had a declaration that didn't match the 
`void(T*)` interface that the smart pointers expect?
####Answer:
Use the adapter pattern to wrap the deleter with the appropriate interface.  One easy way is to wrap it in a lambda which does match that declaration.
***

####Bonus Question:
How would you manage a resource that uses an int-type handle instead of a
pointer? Could a smart pointer still do the job, and if so, how?
####Answer:
Depending on how frequently that resource was used, you may wish to create
a RAII-type class to store it.  If it's infrequently used, you could use
a smart pointer.  For example:
```c++
typedef unsigned int HANDLE;//Like a Windows API identifier.
HANDLE allocate_mutex() { return /*OS-stuff here*/ static_cast<HANDLE>(std::rand()); }
void deallocate_mutex(HANDLE) {/*OS-stuff here*/ std::cout << "Called!" << std::endl; }

struct mutex_destructor
{
  typedef HANDLE pointer;//Just store an int, not a pointer.
  void operator()(HANDLE val)
  { deallocate_mutex(val); }
};
void example()
{
  std::unique_ptr<HANDLE, mutex_destructor> mutex(allocate_mutex());
  //Note that the unique_ptr size is unchanged, since mutex_destructor has no state.
  assert(sizeof(HANDLE) == sizeof(mutex));
}
```
***

####Exercise:
Using custom deleters with `std::unique_ptr` is a little more complicated
 than for `std::shared_ptr`, because the deleter becomes part of its type
 (the 2nd template argument). Try rewriting the above example to use
 `std::unique_ptr` instead of `std::shared_ptr`.
####Answer:
```c++
void example1unique()
{
  std::string filename("c:\\Windows\\win.ini");
  auto deleter = [](FILE* f){std::fclose(f);};
  std::unique_ptr<FILE,void(FILE*)> p_fin(std::fopen(filename.c_str(), "r"),&std::fclose);
  if (!pFin)
    return;

  char data[10];
  if (std::fread(data, 1, 10, pFin.get()) <= 0)
    return;
}
```
***

####Bonus Question:
Does `std::unique_ptr`'s size increase when you use a custom deleter?
If so, why?
####Answer:
Sometimes.  If the deleter need not be passed in as a constructor argument,
then the `unique_ptr` size will be unchanged.  If the deleter must be
referenced or otherwise stored by the `unique_ptr`, it will increase.
***

####Question:
Why is the `weak_ptr` class designed such that we have to convert the `weak_ptr`
into a `shared_ptr` before using the object the weak pointer refers to?
####Answer:
You must obtain a reference to the object to insure that it is not deleted
while the code is accessing it.
***

####Question:
Assuming that `p_int`'s lifetime is managed in a separate thread of execution, what could happen if the code were structured like this:
```c++
if (!p_weak_int.expired())
{
  auto p_tmp_copy = p_weak_int.lock();
  std::cout << "value = " << *p_tmp_copy << std::endl;
}
else
  std::cout << "expired\n";
```
####Answer:
Consider what would happen if instruction interleaving caused the pointer to be deleted
between the call to expired and the call to lock -- the next line would (hopefully) crash!
***

####Question:
In the un-threaded example above, a raw pointer would've served just as well as a
`std::weak_ptr`. However, suppose we had 3 threads, each owning one copy
of the `std::shared_ptr` (so its reference count is 3). Suppose each thread
could terminate at any time (thus decreasing the reference count until it
reaches 0 and the memory is freed). The main thread has a `std::weak_ptr`
copy and is monitoring/reporting its value periodically.

Now what benefit(s) might using a `std::weak_ptr` provide over using a copy
of the raw pointer?

####Answer:
If we had a raw pointer we wouldn't be able to test if it still points to
anything valid (the value of the pointer doesn't get changed), whereas with 
a `weak_ptr`, we can test if the pointer is still valid.
***

####Question:
```c++
auto p_unique = get_unique_ptr(my_lib.open_handle(),
[&](int* p_handle) { my_lib.close_handle(p_handle); });

//Now suppose we need to change to a shared_ptr:
std::shared_ptr<int> p_shared(p_unique.release());
```
What's wrong with the conversion to `shared_ptr` and why does it cause a
crash?
####Answer:
The custom deleter is not transferred from the `unique_ptr` to the `shared_ptr` when you use `release()`.  Therefore, when the `shared_ptr` goes out of scope, it calls delete on the raw pointer instead of the custom deleter.
***

####Question:
The fix for the above looks like this:
```c++
std::shared_ptr<int> p_shared(std::move(p_unique));
```
Rewrite the code above. Why do you have to use `std::move()`?
####Answer:
`unique_ptr` cannot be copied, as that would not provide unique ownership
of the pointer.  `unique_ptr` can be moved from.  This allows `unique_ptr` to
be returned from functions, used in std containers, and more.  `std::move`
explicitly moves the pointer out of `p_unique` into `p_shared`.
***

####Question:
When we copied a `shared_ptr` to another `shared_ptr`, we didn't have to use
`std::move()`. Why not?
####Answer:
Because `shared_ptrs` are copy assignable whereas `unique_ptr`s are not.  With
a `unique_ptr`, we have to help transfer ownership through the move because
ownership can not be copied.

It seems that explicitly calling `std::move` is usually only necessary when
writing move constructors and move assignment operators.  (There are other
cases, like when writing algorithms, but moves are usually safest
(and fastest!) when implicitly generated.)
***

####Question:
```c++
//We have a unique_ptr to const and we want to switch to a unique_ptr to
// non-const:
std::unique_ptr<const int> p_const_unique(new int(5));
//std::unique_ptr<int> p_my_non_const_int(std::move(p_const_unique));
```
If you uncomment the line above, it won't compile.  Why not?
####Answer:
Just as with raw pointers, converting a pointer to `const` to a pointer to
non-`const` will fail.

From a higher-level point of view, if this conversion were allowed, 
then the following could happen.
```c++
std::unique_ptr<const int> p_const_unique(new int(5));
std::unique_ptr<int> p_my_non_const_int(std::move(p_const_unique));
// We are changing the value of something originally declared const!
*p_my_non_const_int = 4;
```
***
####Question:
```c++
//We have a shared_ptr and we want to convert it to a unique_ptr:
std::shared_ptr<int> p_shared_int(new int(6));
//std::unique_ptr<int> p_unique_int(std::move(p_shared_int));
```
If you uncomment the line above, it won't compile. Why not?
You can create a `shared_ptr` from a `unique_ptr`, so why can't you create a
`unique_ptr` from a `shared_ptr`?
####Answer:
Simple answer: It would require that all others who have a shared_ptr to a
resource to give it up, which isn't realistically possible in C++.
Less-simple answer: While `shared_ptr` has a `unique` member function which indicates if more than one copy of it exists, it seems the committee didn't find once-shared-but-now-unique a common enough use-case to merit a conversion.  Since you can check uniqueness at runtime, nothing stops you from acting as if the `shared_ptr` were a `unique_ptr` once you've checked.

####Question 1:
Consider the following code:
```c++
boost::shared_ptr<int> int1(new int(42));
```

How many allocations were required?  What was the size of each allocation?

####Answer 1:
For 32-bit VC10, two allocations were required, 4 bytes for the first, 16 bytes for the second.
For 64-bit VC10, two allocations were required, 4 bytes for the first, 24 bytes for the second.
There are two allocations for a single `shared_ptr`, one for the data being pointed to (what's returned by `std::shared_ptr::get`) and one for the reference count (so co-owning shared pointers know where to look to see how many other referrers there are).

***

####Exercise:
Create a int2 `shared_ptr` using `make_shared`.
```c++
auto int2 = std::make_shared<int>(42);
```

***

####Question 2:
How many allocations were required?  What was the size of each allocation?

####Answer 2:
`make_shared` allocates the object pointed to and the reference count at the same time.  Essentially, it creates one `struct` which holds both pieces of data.  Thus:

For 32-bit VC10, one 16-byte allocation was required.
For 64-bit VC10, one 24-byte allocation was required.
If you used any other standard library, you probably got one 24-byte allocation (on 32-bit systems) or one 32-byte allocation (on 64-bit systems). Why are none of the sizes equal to the sum of the individual allocations?  See below.

***

####BONUS Question 3:
Be assured that `make_shared` is allocating enough space for the same items allocated for int1.  There should be something somewhat surprising about the sum of bytes allocated with `make_shared` versus the original version.  Explain why the surprising result is okay.

####BONUS Answer 3:
This was a harder question than I intended.  First, if you're using standard library from anyone but Microsoft, you probably noticed that it allocated MORE than the sum of the individual allocations.  This is due to how platforms pad their structs -- it's more efficient on most platforms to load aligned data.  (In fact, some platforms crash unless this is the case!)  Without delving into excessive detail, rather than allocating 20 bytes for a single struct holding the pointer and the reference count, the compiler allocates 24, so that the next struct is also aligned.

If that's true, then why does the MS version do what it does?  The STL maintainer uses what he calls the ["we know where you live optimization"](http://boost.2283326.n4.nabble.com/Smart-Ptr-make-shared-slower-than-shared-ptr-new-on-VC-9-and-10-with-fix-td4587958.html).  See his presentation for details, but he essentially saves allocating a pointer by knowing that the allocated pointer was created via `make_shared`.  This allows the `make_shared` struct to be one pointer smaller, which in turn allows the struct to be more compact, all without hurting alignment.

***
####Question 4:
Consider the following code:
```c++
int foo() { throw std::runtime_error("Normally conditional");  return 5; }

void bar(std::shared_ptr<int> int, int val)
{}

void example2()
{
  bar(std::shared_ptr<int>(new int(42)), foo());
}
```
Why could calling example2 result in a memory leak?
####Answer 4:
C++ arguments may be evaluated in unspecified order.  In the previous, the `shared_ptr` constructor and `foo()` must both be evaluated before `bar` is called, but there is no requirement that the `shared_ptr` constructor be called before `foo()` is evaluated.  Thus the order of operations could be:
`new int(42)`
`foo()`
and the `shared_ptr` constructor might never be called, as `foo()` threw and exception.

This is [explicitly warned about](http://www.boost.org/doc/libs/1_54_0/libs/smart_ptr/shared_ptr.htm#BestPractices) in the `shared_ptr` docs.  So two solutions are:
```c++
std::shared_ptr<int> int(new int(42)); //Less freedom to reorder statements.
bar(int, foo());
```

or

```c++
bar(std::make_shared<int>(42), foo());//No problems, make_shared won't be split to call foo().
```

***

####Question 5:
`std::make_shared` always uses the `std::allocator` to retrieve memory.  What should we use if we have a different allocator?
####Answer 5:
[std::allocate_shared](http://www.boost.org/doc/libs/1_54_0/libs/smart_ptr/make_shared.html#functions)

***
####Question 6:
Consider the following two ways to allocate an array:
```c++
void example3()
{
  int* int1 = new int[42];
  delete [] int1;

  std::shared_ptr<int> int2(new int[42], [](int* p) { delete [] p; });
}
```

Both have drawbacks.  Using `boost::shared_ptr` as of 1.53 you can write:
```c++
boost::shared_ptr<int[]> int3(new int[42]);
```
or
```c++
auto int4 = boost::make_shared<int[]>(42);//make_shared_noinit is actually functionally equivalent. . .
```

The drawbacks to the code for int1 in example3 are obvious, since RAII classes protect against memory leaks.  int2 has drawbacks that aren't quite as obvious.  Name at least one drawback to storing an array in a `shared_ptr`.
####Answer 6:
Drawback #1: How do you access the second element of the array?  `int2[1]` doesn't compile, as there's no overloaded `operator[]`.  You end up having to write something like `int2.get()[1]`, which is probably why languages without pointers are so popular these days.  On the flip side, it can be argued that being able to access `*` and `->` can lead to issues for arrays -- it might be better to require `[0]` instead.

Drawback #2: 
The code for specifying an array deleter is verbose.  It can be slightly prettier if you write `std::default_delete<int[]>`, but it's long enough to be a pain to type much.

Drawback #3 (subtle):
Consider the following code
```c++
struct B {};
struct D : public B { std::array<int,2> vals };

std::shared_ptr<B> p_B(new D);//Okay. . .
std::shared_ptr<B> bs(new D[24]);//Awful!
```

If you're not seeing what's wrong immediately, that's because it's subtle.  In More Effective C++, Scott Meyers goes farther than his normal counsel to "avoid" things.  He states "never treat arrays polymorphically." [^1]  And he's right!  Moving from one element to the next in an array requires pointer arithmetic.  If each element is actually further apart than `sizeof(B)`, then the pointer arithmetic will be wrong, and you won't access D[1] when you write pB[1].

Happily, with the new `boost::shared_ptr` type [^2] code like that above will fail to compile:
```c++
boost::shared_ptr<B[]> bs(new D[24]);//Compile error!
```
while allowing other, safe conversions to take place:
```c++
boost::shared_ptr<const B[]> bs(new B[24]);//non-const to const okay. . .
```

Using smart pointers can dramatically improve the safety of your code.  It allows you to convey ownership semantics using something more than comments, while preventing memory leaks and potential double-deletions that come from improper memory leak fixes.

***
####Notes:
[^1] See More Effective C++ item 3

[^2] http://www.boost.org/doc/libs/1_55_0/libs/smart_ptr/shared_ptr.htm