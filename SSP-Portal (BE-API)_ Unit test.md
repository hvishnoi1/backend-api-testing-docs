# SSP-Portal (BE-API) Unit test

- [I) Strategy](#strategy)
  - [What to test?](#what-to-test)
- [II) Testing Guidelines](#testing-guidelines)
  - [Principles](#principles)
  - [Practices](#practices)
    - [Core Practices Concepts](#core-practices-concepts)
      - [Good practices](#good-practices)
      - [Bad practices](#bad-practices)
    - [Test for negative outcomes](#test-for-negative-outcomes)
    - [Cover edge cases](#cover-edge-cases)
    - [Setup/Teardown](#setupteardown)
    - [Test data](#test-data)
      - [Data set categories](#data-set-categories)
        - [1) No Data](#1-no-data)
        - [2) Valid Data](#2-valid-data)
        - [3) Invalid Data](#3-invalid-data)
        - [4) Illegal Data Format](#4-illegal-data-format)
        - [5) Boundary Condition Data Set](#5-boundary-condition-data-set)
        - [6) Decision Table Data Set](#6-decision-table-data-set)
        - [7) State Transition Test Data Set](#7-state-transition-test-data-set)
        - [8) Use Case Test Data](#8-use-case-test-data)
        - [9) Realistic Data](#9-realistic-data)
    - [Tooling](#tooling)

## Strategy

The testing strategy should inform the project mannagers, developers, testers
and other people involved. The test strategy should clearly define the
following:

- **Scope:** What to test, why to test, people involved
- **Testing environemnt:** Staging, dev, prod tests
- **Testing tools:** Junit, Mockito
- **Summary:** Test reports, coverage

### What to test

We need to ensure that we are covering the piece(s) of code we develop by tests.
These tests should measure the application’s behaviour and verify whether they
are expected or not. We need to test each business layer that we are
implementing and verify their behaviour. This involves layers such as DAO to be
verified for storage/retreival from a database, mapper should be verified to map
rows to DTO objects, service layer to sanitize proper and improper data and rest
layer to properly parse data and handle errors.

## Testing Guidelines

### Principles

- Tests should be **fast**
- Tests should be **simple**
- Tests should **not duplicate implementation logic**
- Tests should be **readable**
- Tests should be **deterministic**
- Make sure they are a **part of the build process**

### Practices

#### Core Practices Concepts

##### Good practices

A good, value-adding unit test should encompass even more, meaning the test
should:

- Follow consistent **naming convention** like test*Functionality*_*outcome*,
for example `testCheckValidUser_P`, `testCheckInvalidUser_N`.
- Clearly define the **expected and actual** testing values using variable names
prefixed with `expected` or `actual`, for example, `expectedUser` and
`actualUser`.
- Be **simple** testing only single functionality at a time.
- **Run in memory** and has no effect on the existing state providing no DB or
file access.
- **Consistently return the same result**.
- Have **full control for overall tested units** and should not rely on other
modules.
- **Use Mocks or stubs in isolation** when needed.
- Be **readable**, separate code inside a test into **arrange**, **act** and
**assert** sections.

> A unit test should be fully automated, fast, readable, maintainable and
> trustworthy. A unit test is ultimately about catching bugs in the code. A test
> suite that never goes red equates to unit testing that is actually not working
> and is therefore of limited or no value.

```java
public class UserTest {

    // ...
    @Test
    public void testValidatePassword_P() {
        UserDTO user = new UserDTO(
            "userId",
            "user@email",
            USER_ROLES.SYSTEM_ADMIN.getValue(),
            USER_STATUS.INACTIVE.toString()
        );
        user.setPassword(TEST_PASS);

        String result = service.validatePassword(user, VALID_PASS);

        Assert.assertEquals(Constants.VALIDATION_SUCCESS, result);    
    }
    // ...
}
```

The example above clearly states the test's intention, parameters are legibly
defined, and the assertions deliver a well-understandable message. The test name
follows a defined scheme and the test is devided into 3 parts: initialize the
required variables (arrange), call the implementations (act) and assertions
(assert).

##### Bad practices

A poor unit test can arise due to different attributes or practices, which
should be avoided and may include:

- **Non-deterministic factors in the code-base:** They are problematic since
they are difficult to test; for example, time as an authentication factor in
code can fail due to different time zones.
- **Side-effecting methods:** They are also difficult and problematic to test,
resulting in unwarranted complexities in the code.
- **Anti-patterns, secret dependencies, and other “back-door” scenarios** are
also highly problematic.

```java
public class Test {
    @Test
    public void empty() {
        assertNull(null, null);
    }

    @Test
    public void includes() {
        Integer userId = 9;

        List<Integer> userIds = service.getUserIdByRole(
            USER_ROLES.SYSTEM_ADMIN.getValue()
        );

        assertNotEquals(userIds.indexOf(userId), -1);
    }

    @Test
    public void testAllUsers() {
        List<User> inactiveUsers = service.getUsersByStatus(
            USER_STATUS.INACTIVE.toString()
        );
        List<User> activeUsers = service.getUsersByStatus(
            USER_STATUS.INACTIVE.toString()
        );
        List<User> adminUsers = service.getUsersByRole(
            USER_ROLES.SYSTEM_ADMIN.toString()
        );

        assertTrue(inactiveUsers.size() == 9);
        assertTrue(activeUsers.get(3).getRole() == USER_ROLES.SYSTEM_ADMIN);
        assertTrue(adminUsers.get(0).getStatus == USER_STATUS.ACTIVE.toString());
    }
}
```

The examples above are all bad practices. Their descriptions are vague,
assertions are not clear about what they are comparing, and complexities are
increased unnecessarily.

#### Test for negative outcomes

It is crucial to provide tests for both happy paths and negative outcomes. While
engineering a piece of code, it is easy for engineers to neglect the potential
negative sides of their development and focus on the happy path. Therefore, they
need to keep asking themselves important questions that challenge their code in
both ways and write tests for those questions.

#### Cover edge cases

This is also very important as covering edge cases could be tricky. We need to
spend a good quality of time thinking about the unseen parts of our work. Ask
ourselves questions such as: if 3x2=6, does 1+2x1+2=6 as well?

> It is a good idea to ask our peers to think about these cases as we might be
> missing something obvious due to being heavily drowned in the development
> itself.

#### Setup/Teardown

Setting up the environment for testing is an important step to guarantee all
necessary prerequisites are available before executing the tests. These setups
could include setting up a login session, feeding the database with sample data,
assigning roles to users, etc. Once the setup stage is done, we can start our
tests.

```java
public class Test {
    @BeforeClass
    public static void setUp(TestContext context) {
        vertx = Vertx.vertx();
        // initializes testing application-config. json informat
        ApplicationConfiguration.initialize(
            vertx,
            myHandler -> {
                if (myHandler.succeeded()) {
                    ApplicationConfiguration.getHttpServerOptions().setPort(8888);
                    // when initialization success deploys Verticle
                    DeploymentOptions options = 
                        ApplicationConfiguration.getDeploymentOptions();

                    vertx.deployVerticle(
                        UserRest.class.getName(), 
                        options, 
                        context.asyncAssertSuccess()
                    );
                    client = WebClient.create(vertx);
                }
            }
        );
    }
    
    @AfterClass
    public static void tearDown(TestContext context) {
        vertx.close(context.asyncAssertSuccess());
    }
}
```

In the end, tearing down the setup items is important for security reasons,
performance and re-execution of the tests. For instance, any existing sessions
must be terminated upon tests being finished.

#### Test data

The test data is the initial point of defining the test. The execution and
behaviour of the test are completely dependent on the test data, which serves as
input to the test case.

**Test data could be categorized into two types:**

1. Data generated at runtime.
2. Data picked from different data locations by application.

> Divide the data into different categories of data sets to attain max test
> coverage.

##### Data set categories

###### 1) No Data

Test the features without data.

###### 2) Valid Data

Test the features with valid data.

###### 3) Invalid Data

Test the features with invalid data.

###### 4) Illegal Data Format

Test the features with invalid data format.

Consider the following piece of code to be tested:

```java
public class UserDTO {
    
    private String userId;
    private String email;
    private String role;
    private String status;
    private String password;

    public UserDTO (String userId, String email, String role, String status) {
        this.userId = userId;
        this.email = email;
        this.role = role;
        this.status = status;
    }

    public String setPassword(String password) {
        this.password = password.hashCode();
    }
}
```

*Example Test Data for 1-4 data set categories:*

| No. | Test Case Data | No Data | Valid Data                                | Invalid/Illegal Data               |
|-----|----------------|---------|-------------------------------------------|------------------------------------|
| 1   | `UserDTO`      |         | "1234","user@email","sysadmin","INACTIVE" | 1234,"username","password","admin" |
| 2   | `setPassword`  |         | "passw0rd"                                | true                               |

###### 5) Boundary Condition Data Set

Test for boundaries inside or outside.

*Example Test Data for 5 data set categories:*

| Number of records to fetch | Action |
|----------------------------|--------|
| 4                          | Pass   |
| -4                         | Fail   |
| 4000                       | Fail   |
| 0                          | Pass   |

###### 6) Decision Table Data Set

It is the technique for qualifying your data with a combination of inputs to
produce various results.

*Example Test Data for 6 data set categories:*

| User name status | Password status | Results                       |
|------------------|-----------------|-------------------------------|
| F                | F               | Error: enter username         |
| T                | F               | Error: enter correct pwd      |
| F                | T               | Error: enter correct username |
| T                | T               | Logged in                     |

###### 7) State Transition Test Data Set

It is the testing technique that helps you to validate the state transition of
the feature under test by providing the system with the input conditions. For
example, a user tries to log in to the application with an incorrect username
and password with three attempts and is expected to be blocked.

*The table below indicates how the login feature would behave in multiple
attempts.*

| Application Login Attempts | Invalid Password   |
|----------------------------|--------------------|
| 1st                        | Denied             |
| 2nd                        | Denied             |
| 3rd                        | Denied and Blocked |

###### 8) Use Case Test Data

It is the testing method that identifies our test cases capturing the different
use-cases of a particular feature or a function.

*The tables below show the possible use-cases for testing the login feature.*

| Success Scenario | Step | Description                 |
|------------------|------|-----------------------------|
| User             | 1    | Enter Username and Password |
| System           | 2    | Check Password              |
| System           | 3    | Application Access granted  |

| Failure Scenario | Step | Description            |
|------------------|------|------------------------|
|User              | 1    | Enter Invalid Username |
|System            | 2    | Access denied message  |

###### 9) Realistic Data

Testers use data which is accurate in the context of real-life scenarios

Reference: [test data](https://www.softwaretestinghelp.com/tips-to-design-test-data-before-executing-your-test-cases/)

#### Tooling

At the moment, we use JUnit4 and Mockito for unit testing in the back-end.

> All unit tests are found under `src/test/java` directory for each module with
> the required resources under `src/test/resources` directory.

There are many best practices to follow when using jUnit4. Following are the
examples of assertions using jUnit4 and best practices.

##### 1) Assert Equals

Use `assertEquals` to compare two objects.

```java
@Test
public void testEqual() {
    UserDTO expectedUser = new UserDTO(
        "userId",
        "user@email",
        USER_ROLES.SYSTEM_ADMIN.getValue(),
        USER_STATUS.INACTIVE.toString()
    );

    UserDTO actualUser = service.findUserById("userId");

    assertEquals("Different user values", expectedUser, actualUser);
}
```

> Bad
>> `assertFalse(actual != expected)`
>>
>> `assertTrue(expected == actual)`
>
> Good
>> `assertEquals("Failure message", expected, actual)`

##### 2) Assert Not Equals

Use `assertNotEquals` to compare two objects for inequality.

```java
@Test
public void testNotEqual() {
    UserDTO expectedUser = new UserDTO(
        "userId",
        "user@email",
        USER_ROLES.SYSTEM_ADMIN.getValue(),
        USER_STATUS.INACTIVE.toString()
    );

    UserDTO actualUser = service.findUserById("userId2");

    assertNotEquals("Same values for users", expectedUser, actualUser);
}
```

> Bad
>> `assertFalse(actual == expected)`
>>
>> `assertTrue(expected != actual)`
>
> Good
>> `assertNotEquals("Failure message", expected, actual)`

##### 3) Assert True

Use `assertTrue` to check for boolean `true`.

```java
@Test
public void testTrue() {
    UserDTO user = new UserDTO(
        "userId",
        "user@email",
        USER_ROLES.SYSTEM_ADMIN.getValue(),
        USER_STATUS.INACTIVE.toString()
    );
    service.newUser(user);

    boolean userExists = service.userExists(user.getUserId());

    assertTrue("User not inserted", userExists);
}
```

> Bad
>> `assertTrue(expected == actual)`
>
> Good
>> `assertTrue("Failure message", booleanValue)`

##### 4) Assert False

Use `assertFalse` to check for boolean `false`.

```java
@Test
public void testFalse() {
    service.deleteUser("userId");

    boolean userExists = service.userExists(user.getUserId());

    assertFalse("User not inserted", userExists);
}
```

> Bad
>> `assertFalse(expected == actual)`
>>
>> `assertTrue(expected != actual)`
>
> Good
>> `assertFalse("Failure message", booleanValue)`

##### 5) Assert Null

Use `assertNull` for checking `null` values.

```java
@Test
public void testNull() {
    UserDTO user = service.findUserById("userId");

    assertNull("User is not null", user);
}
```

> Bad
>> `assertTrue(actual == null)`
>>
>> `assertFalse(actual != null)`
>
> Good
>> `assertNull("Failure message", nullValue)`

##### 6) Assert Not Null

Use `assertNotNull` for checking if values are **not** `null`.

```java
@Test
public void testNotNull() {
    UserDTO user = service.findUserById("userId");

    assertNull("User is null", user);
}
```

> Bad
>> `assertTrue(actual != null)`
>>
>> `assertFalse(actual == null)`
>
> Good
>> `assertNull("Failure message", nonNullValue)`

##### 7) Assert Same

Use `assertSame` for checking if values are same.

```java
@Test
public void testSame() {
    UserDTO user1 = service.findUserById("userId");
    UserDTO user2 = service.findUserById("userId");

    assertSame("Objects are not same", user1, user2);
}
```

> Bad
>> `assertTrue(value1 == value2)`
>>
>> `assertTrue(value1.equals(value2))`
>
> Good
>> `assertSame("Failure message", value1, value2)`

##### 8) Assert Not Same

Use `assertNotSame` for checking if values are **not** same.

```java
@Test
public void testNotSame() {
    UserDTO user1 = service.findUserById("userId");
    UserDTO user2 = service.findUserById("userId");

    assertSame("Objects are same", user1, user2);
}
```

> Bad
>> `assertFalse(value1 == value2)`
>>
>> `assertFalse(value1.equals(value2))`
>
> Good
>> `assertNotSame("Failure message", value1, value2)`

##### 9) Assert Array Equals

Use `assertArrayEquals` for checking array equality.

```java
@Test
public void testArrayNotEquals() {
    Integer[] expectedUserRoles = {0, 1, 2};

    Integer[] actualActiveUserRoles = service.getRoles();

    assertArrayEquals("Roles are not same", expectedUserRoles, actualUserRoles);
}
```

> Bad
>> `assertEquals(expected, actual)`
>>
>> `assertTrue(expected == actual)`
>>
>> `for(int i=0; i<actual.size(); i++) { assertEquals(expected[i], actual[i]); }`
>
> Good
>> `assertArrayEquals("Failure message", expectedArray, actualArray)`

##### 10) Assert Array Not Equals

Use `assertArrayNotEquals` for checking array inequality.

```java
@Test
public void testArrayNotEquals() {
    Integer[] expectedUserRoles = {0, 1, 2};

    Integer[] actualActiveUserRoles = service.getAdminRoles();

    assertArrayNotEquals("Roles are same", expectedUserRoles, actualUserRoles);
}
```

> Bad
>> `assertNotEquals(expected, actual)`
>>
>> `assertFalse(expected == actual)`
>>
>> `for(int i=0; i<actual.size(); i++) { assertNotEquals(expected[i], actual[i]); }`
>
> Good
>> `assertArrayEquals("Failure message", expectedArray, actualArray)`

Junit also provides annotations to modify behaviour of test cases and provide
various configurations for tests easily. Following are the examples of
annotations using jUnit4.

##### 1) Test

The `Test` annotation indicates that the method is a test. It helps
differentiate test methods from helper methods.

```java
@Test
public void testMethod() {
    Long count = service.countUsers();

    assertTrue(count > 0);
}
```

##### 2) Before

The `Before` annotation is used to run a method before each test.

```java
@Before
public void initializeService() {
    service.initialize();
}
```

##### 3) After

The `After` annotation is used to run a method after each test.

```java
@After
public void resetService() {
    service.reset();
}
```

##### 4) Before Class

The `BeforeClass` annotation is used to execute a method once before start of
all the tests.

```java
@BeforeClass
public void setup() {
    service = new UserService();
}
```

##### 5) After Class

The `AfterClass` annotation is used to execute a method once after all tests are
completed.

```java
@AfterClass
public void tearDown() {
    service = null;
}
```

##### 6) Ignore

The `Ignore` annotation is used to temporarily disable a test execution.

```java
@Ignore
@Test
public void ignoredTest() {
    assertTrue(true);
}
```

##### 7) Test Exception

Used to specify if a test will throw an exception. If the test does not throws
the exception, the test will fail.

```java
@Test(expected = UserNotFoundException.class)
public void testUdateUserRole() {
    boolean updateSuccess = service.updateUserRole(
        "userId", 
        USER_ROLES.SYSTEM_ADMIN.getValue()
    );

    assertFalse(updateSuccess);
}
```
