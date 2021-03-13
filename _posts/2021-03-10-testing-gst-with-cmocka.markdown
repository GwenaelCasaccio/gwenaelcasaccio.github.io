---
layout: post
title:  "Testing GST (aka GNU Smalltalk) with cmocka"
date:   2021-03-10 09:01:28 +0100
categories: gst cmocka tests c
---
[GST][gst] is an gently fork of the open source smaltalk implementation: [GNU Smalltalk][gnu-smalltalk].
While working on it by doing some improvement or some bug fixes, I've faced another issue: *the lack of tests*
There are some kinds of integrations testing with Smalltalk tests, but no unit tests are done for the virtual machine itself.
After looking to some C framework, I choose  [cmocka][cmocka] because it provides:
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
of a function by a function that simulate that function. 

For instance we want to test a function that will lock a mutex and unlock a mutex and inside increment a variable:

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

With the mock functions we can simulate the pthread\_mutex\_lock, pthread\_mutex\_unlock, perror, fail.

To enable working with mocks we can change the linker and add an option to "wrap" the methods to mock.
We have to define in the test source:

{% highlight c %}
int _wrap___pthread_mutex_lock(pthread_mutex_lock *mutex) {
  check_expected(mutex);

  return mock_type(int);
}

int _wrap__pthread_mutex_unlock(pthread_mutex_lock *mutex) {
  check_expected(mutex);

  return mock_type(int);
}
{% endhighlight %}


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

In that example the method pthread\_mutex\_lock is mocked by the method __wrap_pthread_mutex_lock, 
the function expected_value(\_\_wrap\_pthread\_mutex\_lock, mutex, &mutex); ensures that the mocked method
is well called by the mutex argument variable (checked with the check\_expected function).

and will_return will\_return (the result is used with the corresponding function mock\_type(int))
unfortunatelly there is a restriction with cmocka with cannot check if a mock function is never called.
So we cannot check the perror and fail mock functions are never called (after we trick with a global static variable).

What is really interesting with mock objects is that I can control functions and in the previous example I can return
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

So that's it I hope that you're convince to add mock functions and use [cmocka][cmocka] testing framework!

[gst]: https://github.com/GwenaelCasaccio/smalltalk
[gnu-smalltalk]: https://github.com/gnu-smalltalk/smalltalk
[cmocka]: https://cmocka.org/
