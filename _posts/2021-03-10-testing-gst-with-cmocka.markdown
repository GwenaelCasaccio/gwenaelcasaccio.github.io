---
layout: post
title:  "Testing GST (aka GNU Smalltalk) with cmocka"
date:   2021-03-10 09:01:28 +0100
categories: gst cmocka tests c
---

# What is GST
[GST][gst] is an gentle fork of the open source Smalltalk-80 implementation: [GNU Smalltalk][gnu-smalltalk].
I started locally to improve and fix some parts of it for my own interest and decided to continue o work on
it mostly for the fun.

What improvements:
  * Object layout is much more flexible: adding a new slot (for learning, hacking, ...) is easier and require only localized changes (developper is warmed with a static_assert message)
  * Object table in 64 bits is improved and the stress test is green - you can allocate lot of object before running out of object pointers
  * Some misc issues raised by running the GCC sanitizer
  * Recenlty added an initial support of a multithreaded VM support - it's basic but hey it works
  * Adding vm tests with mock

# Testing the GST virtual machine
While working on the virtual machine I was frustrated by the difficulty to test it. Basically the virtual machine code was not tested in anyway.
So I've started to add some basic tests but I hadn't choose any testing framework for the C language. I've find multiple framwork but most of them
lacks to support mock objects. And for the some of the framwork supporting mocking the implementation was relying on assembly and don't work on some
OSes and architectures - as an hacker point of view certainly a lot of stuff to learn and really interesting - but I want a simple and relyable mocking
support. That's why I choose [cmocka][cmocka], it provides some really nice features:
 * Support for mock objects.
 * Test fixtures.
 * Only requires a C library
 * Exception handling for signals (SIGSEGV, SIGILL, ...)
 * No use of fork()
 * Testing of memory leaks, buffer overflows and underflows.
 * A set of assert macros.
 * Several supported output formats (stdout, TAP, JUnit XML, Subunit)
 * License: Apache License 2.0

The point where I want to focus here is the support for mock objects.

# What is a mock?

A mock object is a simulated object of a real object. In C we want to simulate the behavior 
of a function by another function that will simulate that function.

## Mock example

Here is a small example of a function that will lock a mutex and unlock a mutex and inside increment a variable:

{% highlight c %}
void increment_by_one(void) {
  if (!pthread_mutex_lock(&mutex)) {
    perror("cannot lock the mutex");
    fail();
  }

  counter++;

  if (!pthread_mutex_unlock(&mutex)) {
    perror("cannot unlock the mutex");
    fail();
  }
}
{% endhighlight %}

This simple example highlight the need of mock objects: how can we simulate an error when calling **pthread_mutext_lock** 
and ensure that **perror, fail** functions are called for instance. And we control what is the behavior (the result for instance)
when the mock function **pthread_mutext_lock** is called

## How to mock a function

There is the **wrap** option available in the GNU or LLVM linker which wrap the function pass as an argument to the wrapped function 
**__wrap_original_name_of_the_function** and the original function is still accessible with the **__real_original_name_of_the_function**.

{% highlight bash %}
ld -Wl,-wrap=pthread_mutex_lock -Wl,-wrap=pthread_mutex_unlock -Wl,-wrap=perror  -Wl,-wrap=fail
{% endhighlight %}

Now we can define in the test source:

{% highlight c %}
int _wrap___pthread_mutex_lock(pthread_mutex_lock *mutex) {
  check_expected(mutex);

  return mock_type(int);
}

int _wrap__pthread_mutex_unlock(pthread_mutex_lock *mutex) {
  check_expected(mutex);

  return mock_type(int);
}

...
{% endhighlight %}


## write a test with mock objects

In the testing method the mock will be configured to check the arguments and returns the expected values:
{% highlight c %}
static void test_increment(void **state) {
  expected_value(__wrap_pthread_mutex_lock, mutex, &mutex);
  will_return(__wrap_pthread_mutex_lock, 0);

  expected_value(__wrap_pthread_mutex_unlock, mutex, &mutex);
  will_return(__wrap_pthread_mutex_unlock, 0);

  increment_by_one();

  assert_true(counter == 1);
}
{% endhighlight %}

As I previously say the method **pthread_mutex_lock** is mocked by the method **__wrap_pthread_mutex_lock**, 
the function expected_value(__wrap_pthread_mutex_lock, mutex, &mutex); ensures that the mocked method
is well called by the mutex argument variable (this is checked with the check_expected function in the mocked function).
So the argument of the mock is configured te same is to be done for the returned value with ****will_return*** 
(the result is used with the corresponding function mock_type(int)).

What is really powerful with mock objects is that I can control functions and in the previous example I can return
an error when the pthread_mutex_lock is called and check that perror and fail functions are called. And the value is still
the same.

{% highlight c %}
static void test_increment(void **state) {
  expected_value(__wrap_pthread_mutex_lock, mutex, &mutex);
  will_return(__wrap_pthread_mutex_lock, 1);

  expected_value(__wrap_perror, message, "...");
  
  expected_function_calls(__wrap_fail, 1);

  assert_true(counter == 0);
}
{% endhighlight %}

# Conclusion

So that's it I hope that you're convince to add mock functions and use [cmocka][cmocka] testing framework! You can find a real
example of mock in the [forward_object.c][forward_object.c]. Of course the [API][cmocka-api] provides more functions like controlling the number of times a mock function is called.

[gst]: https://github.com/GwenaelCasaccio/smalltalk
[gnu-smalltalk]: https://github.com/gnu-smalltalk/smalltalk
[forward_object.c]: https://github.com/GwenaelCasaccio/smalltalk/blob/main/tests/vm/forward_object.c
[cmocka]: https://cmocka.org/
[cmocka-api]: https://api.cmocka.org/
