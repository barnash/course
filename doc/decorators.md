
# Decorators

* apply a specific policy or behavior to several functions
* aids in preventing code duplication
* sometimes misused

## Examples

	!python
	# raises an exception if the user is not logged in
	@login_required
	def change_password():
		pass

	# displays a warning when used for the first time
	@deprecated
	def func():
	    pass

	# python can be type safe!
	@typecheck(int, int)
	def add(x, y):
	    pass

---

## Syntax

	!python
	@decorator
	def func():
		pass

Equivalent to:

	!python
	def func():
	    pass
	func = decorator(func)

Definition:

**A decorator is a callable the receives a function and returns a callable**

Truth is, a decorator may return anything, but returning something other than a callable usually doesn't make sense.

---

## The debug decorator

A decorator that prints each time the function is called.

	!python

	def debug(func):
		def decorated(*a, **k):
			print "calling", func.__name__
			return func(*a, **k)
		return decorated

	@debug
	def foo():
	    pass

	>>> foo()
	calling foo

---

## Exercise 1 - a better debug decorator

This debug decorator also prints arguments, exceptions and return values.

Can you implement it using the information you see here?

	!python
	@debug
	def div0(n):
		return n / 0

	@debug
	def div2(n):
		return n / 2

    >>> div2(10)
    div2 got  (10,) {}
    returned: 5

    >>> div0(10)
    div0 got (10,) {}
	raised: integer division or modulo by zero
	Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    ZeroDivisionError: integer division or modulo by zero

---

## Execise 1 - solution

	!python
	def debug(func):
		def new_func(*a, **k):
			print "{} got ".format(func.__name__), a, k
			try:
				ret = func(*a, **k)
			except Exception as e:
				print "raised: {}".format(e)
				raise
			else:
				print "returned: {}".format(ret)
				return ret
		return new_func

---

## Decorators as classes

Useful when a function needs to save state

	!python
	class InvocationCounter(object):

		def __init__(self, func):
		    self._func = func
			self.count = 0

		def __call__(self, *a, **k):
		    self.count += 1
			return self._func(*a, **k)

	@InvocationCounter
	def foo():
		pass

	>>> foo()
	>>> foo()
	>>> foo.count
	2

---

## Decorator factories

A function that generates a decorator

	!python
	def ignore_exceptions(exception_types, default):
		def decorator(func):
			def decorated(*args, **kwargs):
				try:
					return func(*args, **kwargs)
				except exception_types:
					return default
			return decorated
		return decorator

	@ignore_exceptions([ZeroDivisionError], 0)
	def safediv(x, y):
		return x / y

    >>> safediv(3, 0)
	0

---

## Exercise 2 - type checker

Adding some type safety

	!python
	@typesafe(int, int)
	def add(x, y):
		return x + y

	>>> add(5, "bla")
	Traceback (most recent call last):
	  File "<stdin>", line 1, in <module>
    TypeError: invalid type for argument y. expected int, got str

---

## Exercise 2 - solution

	!python
	def typesafe(*types):
		def decorator(func):
			def decorated(*args):
				n_args = len(args)
				n_types = len(types)
				if n_args != n_types:
					raise ValueError("invalid num of args. expected {}, got {}".format(
					                                                  n_types, n_args))
				for arg, arg_type in zip(args, types):
					if not isinstance(arg, arg_type):
						raise TypeError("expected {} got {} for argument {}".format(
						                                  arg_type, type(arg), arg))
				return func(*args)
			return decorated
		return decorator

---

## Class decorator

A class decorator can manipulate the class upon creation.

For example, `decorate_methods` gets a class, iterates over its methods and decorates them with the `debug` decorator.

	!python

	@decorate_methods(debug)
	class Calculator(object):
		def add(self, x, y):
			return x + y
		def sub(self, x, y):
			return x - y

Note: methods in child classes won't be decorated (metaclasses can solve that).

---

## gotcha #1

Decorators usually replace a function with two nested ones and should be avoided where cpu is the bottleneck.

---

## gotcha #2

Don't decorate a function with a function that gets a different number of arguments.

An atrocity:

	!python

    # The retry decorator returns a function that gets a 'retries' argument indicating
    # how many times to invoke the decorated function in case an exception occured.

	@retry
	def foo():
	    pass

    >>> foo(retries=3) # how can a fresh coder know this argument exists?

---

## gotcha #3

Always use `functools.wraps` in order to preserve the function's name and documentation.

Without `functools.wraps`:

	!python

	def debug(func):
		def decorated(*a, **k):
			print "calling", func.__name__
			return func(*a, **k)
		return decorated

	@debug
	def foo():
		"""
		foo does this and that.
		"""
	    pass

	>>> foo.__name__
	decorated
	>>> foo.__doc__
	None

---

With `functools.wraps`:

	!python
	from functools import wraps

	def debug(func):
		@wraps(func)
		def decorated(*a, **k):
			print "calling", func.__name__
			return func(*a, **k)
		return decorated

	@debug
	def foo():
		"""
		foo does this and that.
		"""
	    pass

	>>> foo.__name__
	foo
	>>> foo.__doc__
	foo does this and that.


---

## Exercise 3 - cached

The `cached` decorator records the arguments and return values a function received and returned.

This method can be used to save cpu time in exchange for memory.

	!python

	@cached
	def calc(x, y):
		print "performing a long computation..."
		return x + y

	>>> print calc(10, 20)
	performing a long computation...
	30
	>>> print calc(10, 20) # cache hit!
	30
	>>> print calc(10, 50)
	performing a long computation...
	60

Bonus - add a 'timeout' parameter to the cached decorator, indicating how long should objects stay in the cache.

---

## Exercise 3 - solution

    !python
    def cached(func):
        args_to_result = {}
        def decorated(*args):
            if args not in args_to_result:
                args_to_result[args] = func(*args)
            return args_to_result[args]
        return decorated
