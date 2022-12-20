# Back-End (API): Integration Tests Guidelines

- [I) Strategy](#strategy)
  - [What is Integration Testing?](#what-is-integration-testing)
  - [What to test?](#what-to-test)
- [II) Testing Guidelines](#ii-testing-guidelines)
  - [Principles](#principles)
  - [Practices](#practices)
    - [Setup/Teardown](#setupteardown)
    - [Test Doubles](#test-doubles)
      - [Examples](#examples)
        - [1. Power Mockito](#1-power-mockito)
        - [2. Stubs](#2-stubs)
        - [3. Mocks](#3-mocks)

## Strategy

The testing strategy should inform the project mannagers, developers, testers
and other people involved. The test strategy should clearly define the
following:

- **Scope:** What to test, why to test, people involved
- **Testing environemnt:** Staging, dev, prod tests
- **Testing tools:** Junit, Mockito
- **Summary:** Test reports, coverage

### What is Integration Testing?

Integration testing revolves around assuring that two or more of the various
modules we have created will interact with each other properly.

> Our definition of integration test is: Tests which cover the interaction
> between two or more modules, this includes testing for code that talks to
> outside services.

### What to test?

- Test the interactions between various modules in our code.
- Test our interaction with outside code.
- We want our tests to be valuable. Tests need to be centred around vital
portions of the code.
- Test need not mock the behaviour of other modules rather call the actual
implementations.

> We want to avoid testing all potential scenarios in a desperate attempt to
> raise our code coverage. Code coverage is not an accurate measure for the
> quality of tests. Also it should be noted that the code coverge for a module
> should not be taken into account when testing outside of the module scope for
> tested module.

## II) Testing Guidelines

### Principles

As integration tests cover much more ground, it may not always be obvious as to
what is causing a failure.

- **Logging:** Thorough logging is the best way to help developers and QA
identify the source of the issues. Usage of logging libraries like `slf4j` and
`log4j` is recommended. These libraries allow to configure various output
sources and formats for properly formatting output to make the logs easier to
investigate and follow.

- **Understand the difference between Integration tests and Unit tests:** Unit
tests are usually smaller, encapsulated and much simpler. Unit test failures
should be related to the business logic not being sound. Integration tests on
the other hand should identify issues between our various modules or when
communicating to an outside source. Examples include calling an API or module.

- **Integration and Unit tests should run separately:** The business logic
should not be tested in Integration tests, business logic should always be
tested in the Unit tests. Integration tests are longer to run and therefore
testing business logic should not be dependent on these lengthy tests. Keeping
the two test suites separate ensures that we don’t slow down the process for
Unit tests. Ideally the integration tests should be performed after verifying
the unit tests.

- **Separate Unit tests and Integration tests:** Unit tests need to be able to
run quickly in order to give developers a quick response to their recent
changes, unlike integration tests which do not simulate the behaviour of an
implementation (using mocks and stubs) instead call the actual implementation.
If the two test types are not clearly separated testing will be overly
time-consuming which risks turning devs away from using unit tests for their
intended purpose.

- **Tests should be atomic:** In other words, tests should not be dependent on
previous test results in order to pass. Each test and its results should be
isolated from other tests. Use setup/teardown methods to ensure the required
setup before/after each test case.

- **Tests should not affect actual data:** Since integration tests make use of
external modules directly instead of relying on mocks or stubs, extra attention
should be given to tests to check if they are not modifying any existing data.
For this purpose tests should use an in-memory database or make use of temporary
files whenever required to make sure that the tests are not interfering with the
actual data or state. Some examples of tests modifying actual data include:
  - Creating/modifyng/deleting data from actual database
  - Changing config files
  - Changing data in a data store

### Practices

#### Setup/Teardown

- **We have the setup method annotated with `beforeClass` to create any objects
needed during the testing phase:**  In the setup method seen in the example
below, we can see that the user is signed in and the order to be used for
testing is created.

Tests need to be idempotent and isolated: Meaning that the order in which they
are run should have no effect on the outcome and the state should not be
persisted after the test is run.

The example below is of an integration test from setup to teardown. In this test
a test user is signed in and a new order is created. The test ensures that this
process functions correctly.

```java
public class OrderServiceTest {

    UserService userService = UserService.getInstance();
    OrderService orderService = OrderService.getInstance();
    UserDTO testUser;
    OrderDTO testOrder;

    @BeforeClass
    public void setup() {
        testUser = userService.signIn("userId",TEST_PASS);
        testOrder = orderService.newOrder(orderId, oldOrder, newOrder);
    }

    @AfterClass
    public void teardown() {
        testUser.signOut();
        orderService = null;
    }

    @Test
    public void getAllOrdersTest() {
        List<OrderDTO> orders = orderService.getAllOrders();
        Assert.assertNotNull(orders);
    }
}
```

#### Test Doubles

"Meszaros uses the term **Test Double** as the generic term for any kind of
pretend object used in place of a real object for testing purposes. The name
comes from the notion of a Stunt Double in movies. (One of his aims was to avoid
using any name that was already widely used.) Meszaros then defined five
[particular kinds of double](https://martinfowler.com/bliki/TestDouble.html):

**Dummy** objects are passed around but never actually used. Usually, they are
just used to fill parameter lists.

**Fake** objects actually have working implementations, but usually take some
shortcut which makes them not suitable for production (an [in-memory
database](https://martinfowler.com/bliki/InMemoryTestDatabase.html) is a good
example).

**Stubs** provide canned answers to calls made during the test, usually not
responding at all to anything outside what's programmed in for the test.

**Spies** are stubs that also record some information based on how they were
called. One form of this might be an email service that records how many
messages it was sent.

**Mocks** are what we are talking about here: objects pre-programmed with
expectations which form a specification of the calls they are expected to
receive.

> Of these kinds of doubles, only mocks insist upon behaviour verification. The
> other doubles can, and usually do use state verification.

 Mocks actually do behave like other doubles during the exercise phase, as they
 need to make the SUT believe it's talking with its real collaborators - but
 mocks differ in the setup and the verification phases.” (Martin Fowler, The
 Difference Between Mocks and Stubs)

##### Examples

###### 1. Power Mockito

We can use PowerMockito's 'doAnswer' method to intercept call to void method. In
the example below we are intercepting a web request and mocking the response
returned from the call.

```java
public class OderRestTest {
    @Test
    public void getOrdersByUserTest() {
        PowerMockito.doAnswer(
            (Answer<AsyncResult<Order>>) invocationOnMock -> {
                ((Handler<AsyncResult<Order>>) invocationOnMock
                    .getArgument(0))
                    .handle(Future.succeededFuture(mockOrderList));
            })
            .when(client)
            .get(GET_ORDERS_BY_USER);

        client
            .get(GET_ORDERS_BY_USER)
            .sendBuffer(body, response -> {
                Assert.assertTrue(response.succeeded());
            });
    }
}
```

###### 2. Stubs

Stubs allow us to reduce dependencies on outside resources during testing by
allowing us to return hard-coded responses. Below is an example of Stubbing, the
method `getTotalOrdersStub` returns the total number of orders when called.

```java
public class TotalOrdersTest {

    private Integer getTotalOrdersStub() {
        return TOTAL_ORDERS_CONST;
    }

    @Test
    public void getTotalOrdersTest () {
        PowerMockito.doAnswer(
            (Answer<AsyncResult<Integer>>) invocationOnMock -> {
                ((Handler<AsyncResult<Integer>>) invocationOnMock
                    .handle(Future.succeededFuture(getTotalOrdersStub()));
            })
            .when(client)
            .get(GET_TOTAL_ORDERS);

        client
            .get(GET_TOTAL_ORDERS)
            .sendBuffer(body, response -> {
                Assert.assertTrue(response.succeeded());
                Assert.assertEqual(
                    Integer.valueOf(response.result().bodyAsString()),
                    TOTAL_ORDERS_CONST
                );
            });
    }
}
```

###### 3. Mocks

Mocks are a bit more complex than stubs, they are created with some expectations
(expected method calls) and can then be verified to ensure those methods were
called. Below is an example of a Mocking taken from outside the code for
simplicity.

```java
public class TotalOrdersTest {

    @Test
    public void getTotalOrdersTest () {
        PowerMockito.doAnswer(
            (Answer<AsyncResult<Order>>) invocationOnMock -> {
                ((Handler<AsyncResult<Order>>) invocationOnMock
                    .handle(orderAsyncResultHandler);
            })
            .when(client)
            .get(GET_TOTAL_ORDERS);

        PowerMockito.when(orderAsyncResultHandler.succeeded()).thenReturn(true);

        client
            .get(GET_TOTAL_ORDERS)
            .sendBuffer(body, response -> {
                Assert.assertTrue(response.succeeded());
                Assert.assertEqual(
                    Integer.valueOf(response.result().bodyAsString()),
                    TOTAL_ORDERS_CONST
                );
            });
    }
}
```

### Tooling

Considering our current use of *JUnit* and *Mockito* for our unit tests it would
be logical to make the use of the same tool for our Integration tests.
