---
layout: post
title:  "Verifying constraint violations on mocked APIs"
date:   2015-10-11
---
When using mocks in unit testing there is a case where subtle bugs can occur. Maybe they will raise in integration 
testing or even worse later in production. Usually, one verifies that a mock has or hasn't been called during a unit
test. In mockito it's accomplished with ```verify``` and can be used to check the given parameters certain constraints.
For example to verify that a method must not be called with null value one can use the matcher ```notnull()```:

    Mockito.verify(mock).methodCall(notnull())
    
This approach has some draw backs because the verification checks are tightly coupled to the preconditions of an API. It
is easy to forget a check to not null. Or more subtle, an API of an object could be changed after the time of test
writing. If one is lucky then there will be other unit test that fails after changing the API. But if not, the bug is
set.
 
The project [mockito-precond-verifier](https://github.com/Stefku/mockito-precond-verifier) provides a way to verify that 
certain constraints are not violated when a mock has been called. It makes
use of parameter annotations with runtime retention. These annotations inspected and the calls to the mock will be
validated with respect to these preconditions.
 
Example
-------

Let's say there is a class MainService that makes use of a OtherService

```java
public class MainService {
    private OtherService otherService;
    
    public MainService(OtherService otherService) {
        this.otherService = otherService;
    }
    
    public long square(Long input) {
        return otherService.square(input);
    }
}
```
     
The called method of the other service only takes values that are not null and would otherwise throw a 
NullPointerException

```java
public class OtherService {
    /**
     * This method throws a NPE if {@code input} is {@code null}.
     *
     * @param input {@code null} is not allowed
     * @return square of input.
     */
    @SuppressWarnings("UnnecessaryUnboxing")
    public long square(@NotNull Long input) {
        return input.longValue() * input.longValue();
    }
}
```
    
In a test that uses real objects a call to MainService#square() would lead to a NPE

```java
@Test(expected = NullPointerException.class)
public void test_exception_on_null() throws Exception {
    OtherService otherService = new OtherService();
    MainService sut = new MainService(otherService);

    sut.square(null);
}
```
    
But if using a mock of OtherService the NPE would not be thrown

```java
@Test
public void test_exception_on_null() throws Exception {
    MainService sut = new MainService(otherService);

    try {
        sut.square(null);
    } catch (NullPointerException ex) {
        throw new NullPointerException("Will never be thrown because the mock is not aware of precondition @NotNull");
    }
}
```
    
Using the constraint verification of this project the illegal call leads to an AssertionError. So this unit test

```java
@Test
public void test_that_constraint_verifier_throws_Error_on_null() throws Exception {
    MainService sut = new MainService(otherService);

    sut.square(null);

    ValidationConstraintVerifier verifier = new ValidationConstraintVerifier();
    verifier.verifyConstraints(otherService);
}
```

will raise

    java.lang.AssertionError: Constraint Validation Error: Called @NotNull parameter with null
    
To make this validation available in all tests the verification can be made in tear down method

```java
private ValidationConstraintVerifier verifier = new ValidationConstraintVerifier();

@Before
public void setUp() throws Exception {
    otherService = mock(OtherService.class);
}

@After
public void tearDown() {
    verifier.verifyConstraints(otherService);
}
```

To configure this every time is kind of unhandy. For production readyness it would be good to have this configured 
oncy for a whole project.
