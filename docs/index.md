# pytest-steps

*Create step-wise / incremental tests in `pytest`.*

[![Build Status](https://travis-ci.org/smarie/python-pytest-steps.svg?branch=master)](https://travis-ci.org/smarie/python-pytest-steps) [![Tests Status](https://smarie.github.io/python-pytest-steps/junit/junit-badge.svg?dummy=8484744)](https://smarie.github.io/python-pytest-steps/junit/report.html) [![codecov](https://codecov.io/gh/smarie/python-pytest-steps/branch/master/graph/badge.svg)](https://codecov.io/gh/smarie/python-pytest-steps) [![Documentation](https://img.shields.io/badge/docs-latest-blue.svg)](https://smarie.github.io/python-pytest-steps/) [![PyPI](https://img.shields.io/badge/PyPI-pytest_steps-blue.svg)](https://pypi.python.org/pypi/pytest_steps/)

!!! success "New 'generator' style is there, [check it out](#1-usage-generator-mode) !"

Did you ever want to organize your test is incremental steps, for example to improve readability in case of failure ? Or to have some optional steps, executing only conditionally to previous steps' results?

`pytest-steps` leverages `pytest` and its great `@pytest.mark.parametrize` and `@pytest.fixture` decorators, so that you can **create incremental tests with steps** without having to think about the pytest fixture/parametrize pattern that has to be implemented for your particular case. 

This is particularly helpful if:
 
 - you wish to **share a state / intermediate results** across test steps
 - your tests **already rely on other fixtures and/or parameters**, such as with [pytest-cases](https://smarie.github.io/python-pytest-cases/). In that case, finding the correct pytest design, that will ensure that you have a brand new state object at the beginning of each test suite while ensuring that this object will actually be shared across all steps, might be very tricky. 

With `pytest-steps` you don't have to care about the internals: it just works as expected.

!!! note
    `pytest-steps` has not yet been tested with pytest-xdist. See [#7](https://github.com/smarie/python-pytest-steps/issues/7)

## Installing

```bash
> pip install pytest_steps
```

## 1. Usage - "generator" mode

This new mode may seem more natural and readable to non-pytest experts. However it may be harder to debug when used in combination with other `pytest` tricks. In such case, do not hesitate to switch to good old ["explicit" mode](#2-usage-explicit-mode).

### a- Basics

Start with you favorite test function. There are two things to do, to break it down into steps:

 - decorate it with `@test_steps` to declare what are the steps that will be performed, as strings. 
 - insert as many `yield` statements in your function body as there are steps. The function should end with a `yield` (not `return`!). Code written after the last yield with not be executed. 

For example we define three steps:

```python
from pytest_steps import test_steps

@test_steps('step_a', 'step_b', 'step_c')
def test_suite():
    # Step A
    print("step a")
    assert not False  # replace with your logic
    intermediate_a = 'hello'
    yield

    # Step B
    print("step b")
    assert not False  # replace with your logic
    yield

    # Step C
    print("step c")
    new_text = intermediate_a + " ... augmented"
    print(new_text)
    assert len(new_text) == 56
    yield
```

That's it! If you run `pytest` you will now see 3 tests instead of one:

```
============================= test session starts =============================
(...)
collected 3 items                                                           

(...)/test_example.py::test_suite[step_a] <- <decorator-gen-3> PASSED [ 33%]
(...)/test_example.py::test_suite[step_b] <- <decorator-gen-3> PASSED [ 66%]
(...)/test_example.py::test_suite[step_c] <- <decorator-gen-3> PASSED [100%]

========================== 3 passed in 0.06 seconds ===========================
```

!!! note "Debugging note"
    You might wish to use `yield <step_name>` instead of `yield` at the end of each step when debugging if you think that there is an issue with the execution order. This will activate a built-in checker, that will check that each step name in the declared sequence corresponds to what you actually yield at the end of that step.

### b- Sharing intermediate results between steps

By design, all intermediate results created during function execution are shared between steps, since they are part of the same python function call. You therefore have nothing to do: this is what is shown above in step c where we reuse `intermediate_a` from step a. 

### c- Optional steps and dependencies

In this *generator* mode, all steps **depend on all previous steps** by default: if a step fails, all subsequent steps will be skipped. To circumvent this behaviour you can declare a step as optional. This means that subsequent steps will not depend on it except explicitly stated. For this you should:

 - wrap the step into the special `optional_step` context manager,
 - `yield` the corresponding context object at the end of the step, instead of `None` or the step name. This is very important, otherwise the step will be considered as successful by pytest!

For example:

```python
    # Step B
    with optional_step('step_b') as step_b:
        print("step b")
        assert False
    yield step_b
```

If steps *depend on an optional step* in order to execute, you should make them optional too, and state it explicitly:

 - declare the dependency using the `depends_on` argument.
 - use `should_run()` to test if the code block should be executed. 

The example below shows 4 steps, where steps a and d are mandatory and b and c are optional with c dependent on b:

```python
from pytest_steps import test_steps, optional_step

@test_steps('step_a', 'step_b', 'step_c', 'step_d')
def test_suite_opt():
    # Step A
    assert not False
    yield

    # Step B
    with optional_step('step_b') as step_b:
        assert False
    yield step_b

    # Step C depends on step B
    with optional_step('step_c', depends_on=step_b) as step_c:
        if step_c.should_run():
            assert True
    yield step_c

    # Step D
    assert not False
    yield
```

Running it with `pytest` shows the desired behaviour: step b **fails** but does not prevent step d to execute correctly. step c is marked as **skipped** because its dependency (step b) failed. 

```
============================= test session starts =============================
(...)
collected 4 items                                                              

(...)/test_example.py::test_suite_opt[step_a] <- <decorator-gen-3> PASSED [ 25%]
(...)/test_example.py::test_suite_opt[step_b] <- <decorator-gen-3> FAILED [ 50%]
(...)/test_example.py::test_suite_opt[step_c] <- <decorator-gen-3> SKIPPED [ 75%]
(...)/test_example.py::test_suite_opt[step_d] <- <decorator-gen-3> PASSED [100%]

================================== FAILURES ===================================
_______________ test_suite_optional_and_dependent_steps[step_b] _______________
(...)
================ 1 failed, 2 passed, 1 skipped in 0.16 seconds ================
```

### d- Compliance with the other pytest mechanisms

Under the hood, the `@test_steps` decorator simply generates a wrapper function around your function and mark it with `@pytest.mark.parametrize`, that is equivalent to this:

```python
import pytest
from pytest_steps.steps_generator import get_pytest_id

step_ids = [get_pytest_id(f) for f in steps]

@pytest.mark.parametrize('________step_name_', step_ids, ids=str)
def step_function_wrapper(________step_name_, request, *args, **kwargs):
    # here we create an execution monitor on first step 
    # by calling f(*args, **kwargs) and saving the resulting generator,
    # and we reuse the same generator for subsequent steps
    ...
``` 

The function wrapper is created using the excellent [`decorator`](https://github.com/micheles/decorator) library, so all marks that exist on it are kept in the process, as well as its name and signature. Therefore `@test_steps` should be compliant with all native pytest mechanisms. 

For exemple you can use decorators such as `@pytest.mark.parametrize` before or after it in the function decoration order (depending on your desired resulting test order):
 
```python
@test_steps('step_a', 'step_b')
@pytest.mark.parametrize('i', range(2), ids=lambda i: "i=%i" % i)
def test_suite_p(i):
    # Step A
    print("step a, i=%i" % i)
    assert not False  # replace with your logic
    yield

    # Step B
    print("step b, i=%i" % i)
    assert not False  # replace with your logic
    yield
```

If you execute it, it correctly executes all the steps for each parameter value:

```
============================= test session starts =============================
(...)
collected 4 items                                                              

(...)/test_example.py::test_suite_p[i=0-step_a] <- <decorator-gen-3> PASSED [ 25%]
(...)/test_example.py::test_suite_p[i=0-step_b] <- <decorator-gen-3> PASSED [ 50%]
(...)/test_example.py::test_suite_p[i=1-step_a] <- <decorator-gen-3> PASSED [ 75%]
(...)/test_example.py::test_suite_p[i=1-step_b] <- <decorator-gen-3> PASSED [100%]
========================== 4 passed in 0.07 seconds ===========================
```

You can also use fixtures as usual, but special care has to be taken about **function-scope fixtures**. Let's consider the following example:

```python
class MyFixture(object):
    def __init__(self):
        print("created new fixture %i" % id(self))
        self.i = 0
    
    def call(self):
        self.i += 1
        print("%i called for the %i-th time" % (id(self), self.i))

@pytest.fixture
def my_fixture():
    """Simple function-scoped fixture that return a new instance each time"""
    return MyFixture()

@test_steps('step_a', 'step_b')
def test_suite_fix(my_fixture):
    # Step A
    print("step a")
    assert not False  # replace with your logic
    my_fixture.call()
    yield

    # Step B
    print("step b")
    assert not False  # replace with your logic
    my_fixture.call()
    yield
```

Here, and that can be a bit misleading, `pytest` will call `my_fixture()` twice (because there are two pytest function executions, one for each step), but the second instance will never be passed to our code: instead, the `my_fixture` instance that was passed as argument in the first step will be used by all steps.

If we look at pytest results using the `-s` flag to see the prints, we can see it clearly:

```
(...)/test_example.py::test_suite_fix[step_a] <- <decorator-gen-3>
created new fixture 60339872
step a
60339872 called for the 1-th time
PASSED
(...)/test_example.py::test_suite_fix[step_b] <- <decorator-gen-3>
created new fixture 60341832
step b
60339872 called for the 2-th time
PASSED
```

Indeed, the second fixture instance `60341832` is created but never used.

It is possible to circumvent this behaviour, for the cases where you would like each step to have its own fixture instance. Simply decorate your fixture with `@one_per_step`:

```python
from pytest_steps import one_per_step

@pytest.fixture
@one_per_step
def my_fixture():
    """Simple function-scoped fixture that return a new instance each time"""
    return MyFixture()
```

Each step will now use its own fixture instance:

```
(...)/test_example.py::test_suite_fix[step_a] <- <decorator-gen-3>
created new fixture 60456520
step a
60456520 called for the 1-th time
PASSED
(...)/test_example.py::test_suite_fix[step_b] <- <decorator-gen-3>
new fixture 60567792
step b
60567792 called for the 1-th time
PASSED
```

## 2. Usage - "explicit" mode

In "explicit" mode, things are a bit more complex to write but can be easier to understand because it does not use generators, just simple function calls.

### a- Basics
 
Like for the other mode, simply decorate your test function with `@test_steps` and declare what are the steps that will be performed. In addition, put a `test_step` parameter in your function, that will receive the current step.

The easiest way to use it is to declare each step as a function:

```python
from pytest_steps import test_steps

def step_a():
    # perform this step ...
    print("step a")
    assert not False  # replace with your logic

def step_b():
    # perform this step
    print("step b")
    assert not False  # replace with your logic

@test_steps(step_a, step_b)
def test_suite_1(test_step):
    # Optional: perform step-specific reasoning, for example to select arguments
    if test_step.__name__ == "step_a":
        print("calling step a")
    
    # Execute the step by calling the test step function
    test_step()
```

Note: as shown above, you can perform some reasoning about the step at hand in `test_suite_1`, by looking at the `test_step` object.

!!! note "Custom parameter name"
    You might want another name than `test_step` to receive the current step. The `test_step_argname` argument can be used to change that name.

#### Variants: other types of steps

This mechanism is actually nothing more than a pytest parameter so it has to requirement on the `test_step` type. It is therefore possible to use other types, for example to declare the test steps as strings instead of function:

```python
@test_steps('step_a', 'step_b')
def test_suite_2(test_step):
    # Execute the step according to name
    if test_step == 'step_a':
        step_a()
    elif test_step == 'step_b':
        step_b()

...
```

This has pros and cons:

 - (+) you can declare the test suite *before* the step functions in the python file (better readability !)
 - (-) you can not use `@depends_on` to decorate your step functions: you can only rely on shared data container to create dependencies (as explained below)


### b- Auto-skip/fail

In this *explicit* mode **all steps are optional/independent** by default: each of them will be run, whatever the execution result of others. If you wish to change this, you can use the `@depends_on` decorator to mark a step as to be automatically skipped or failed if some other steps did not run successfully.

For example:

```python
from pytest_steps import depends_on

def step_a():
    ...

@depends_on(step_a)
def step_b():
    ...
```

That way, `step_b` will now be skipped if `step_a` does not run successfully. 

Note that if you use shared data (see below), you can perform similar, and also more advanced dependency checks, by checking the contents of the shared data and calling `pytest.skip()` or `pytest.fail()` according to what is present. See `step_b` in the example below for an illustration.

!!! warning
    The `@depends_on` decorator is only effective if the decorated step function is used "as is" as an argument in `@test_steps()`. If a non-direct relation is used, such as using the test step name as argument, you should use a shared data container (see below) to manually create the dependency.


### c- Shared data

In this *explicit* mode, by default all steps are independent, therefore they **do not have access to each other's execution results**. To solve this problem, you can add a `steps_data` argument to your test function. If you do so, a `StepsDataHolder` object will be injected in this variable, that you can use to store and retrieve results. Simply create fields on it and store whatever you like:

```python
import pytest
from pytest_steps import test_steps

@test_steps('step_a', 'step_b')
def test_suite_with_shared_results(test_step, steps_data):
    # Execute the step with access to the steps_data holder
    if test_step == 'step_a':
        step_a(steps_data)
    elif test_step == 'step_b':
        step_b(steps_data)

def step_a(steps_data):
    # perform this step ...
    print("step a")
    assert not False  # replace with your logic

    # intermediate results can be stored in steps_data
    steps_data.intermediate_a = 'hello'

def step_b(steps_data):
    # perform this step, leveraging the previous step's results
    print("step b")
    
    # you can leverage the results from previous steps... 
    # ... or pytest.skip if not relevant
    if len(steps_data.intermediate_a) < 5:
        pytest.skip("Step b should only be executed if the text is long enough")
    
    new_text = steps_data.intermediate_a + " ... augmented"  
    print(new_text)
    assert len(new_text) == 56
```

### d- Compliance with the other pytest mechanisms

You can add as many `@pytest.mark.parametrize` and pytest fixtures in your test suite function, it should work as expected: a **new** `steps_data` object will be created everytime a new parameter/fixture combination is created, and that object will be **shared** across steps with the same parameters and fixtures.

!!! info
    If you wish to share data between steps but you **also** wish to collect the whole dataset at the end of the tests, do not use `steps_data`. Instead, use `results_bag` from [pytest-patterns](https://smarie.github.io/pytest-patterns), it will work similarly but you will be able to collect it when pytest session ends.


## Main features / benefits

 * **Split tests into steps**. Although the best practices in testing are very much in favor of having each test completely independent of the other ones (for example for distributed execution), there is definitely some value in results readability to break down tests into chained sub-tests (steps). The `@test_steps` decorator provides an intuitive way to do that without forcing any data model (steps can be functions, objects, etc.).
 * **Multi-style**: an *explicit* mode and a *generator* mode are supported, developers may wish to use one or the other depending on their coding style or readability target.
 * **Steps can share data**: In *generator* mode this is out-of-the-box. In *explicit* mode all steps in the same test suite can share data through the injected `steps_data` container (name is configurable).
 * **Steps dependencies can be defined**: a `@depends_on` decorator (*explicit* mode) or an `optional_step` context manager (*generator* mode) allow you to specify that a given test step should be skipped or failed if its dependencies did not complete. 

## See Also

 - [pytest documentation on parametrize](https://docs.pytest.org/en/latest/parametrize.html)
 - [pytest documentation on fixtures](https://docs.pytest.org/en/latest/fixture.html)
 - [pytest-cases](https://smarie.github.io/python-pytest-cases/), to go further and separate test data from test functions

### Others

*Do you like this library ? You might also like [my other python libraries](https://github.com/smarie/OVERVIEW#python)* 

## Want to contribute ?

Details on the github page: [https://github.com/smarie/python-pytest-steps](https://github.com/smarie/python-pytest-steps)