# TSTestCourse

This repository follows the course [Unit Testing for TypeScript & Node.js Developers with Jest](https://www.udemy.com/user/alexhorea/) by Alex Horea. It is a small TypeScript application used to learn testing one step at a time.

The examples live under `src/test`. They are lessons in the same project, not separate applications. Start with small functions and gradually test bigger parts of the server.

## Start Here

You need Node.js and npm installed. In the project folder, run:

```sh
npm install
npm test
```

Other useful commands:

```sh
npm run itest  # run the request tests selected by itest.config.ts
npm run build  # type-check and compile the TypeScript project
```

Jest tests usually follow three simple steps:

1. **Arrange:** prepare the input and anything the test needs.
2. **Act:** call the function or method.
3. **Assert:** check that the result or behavior is what you expected.

For example: give `toUpperCase` the word `abc`, call it, and expect `ABC` back.

## Learn In This Order

### 1. A tiny unit test

Open `src/test/Utils.test.ts` and find the `toUpperCase` test. A **unit test** checks one small piece of code by itself. Here, the input is a string and the output should be its uppercase version.

`describe` groups related tests. `it` and `test` both create one test; they mean the same thing. `expect(actual).toBe(expected)` checks a result. Read a test as a small sentence: “When I do this, I expect that.”

### 2. Checking strings, objects, and collections

The same file tests `getStringInfo`. Its result has several parts, so the tests check its length, upper- and lowercase versions, and character list.

- `toBe` is useful for simple values such as strings and numbers.
- `toEqual` checks the contents of objects and arrays.
- `toHaveLength`, `toContain`, and `toBeDefined` check common details.
- `expect.arrayContaining(...)` checks that certain items are present, even if you do not need to check the whole array in that order.

### 3. Errors and several inputs

Still in `src/test/Utils.test.ts`, look for tests that expect an invalid string to throw an error. `toThrow` needs a function to call later, so pass it a function rather than calling the function before `expect` gets it. The file shows function and arrow-function forms, plus a try/catch example.

The `it.each` section runs the same kind of test with several input/output pairs. This is called a **parameterized** or **table-driven test**. Use it when the rule stays the same but the examples change.

### 4. Testing rules and edge cases

Open `src/test/pass_checker/PasswordChecker.test.ts`. These are unit tests for business rules: passwords that are too short, missing uppercase or lowercase letters, and admin passwords missing a number. Each test changes one important condition and checks both the result and the reason.

This folder is a useful early lesson, but note that it is not currently included in either npm test command's Jest `testMatch` list. The same is true for `src/test/doubles` and `src/test/server_app3`; see [Which Tests Run?](#which-tests-run).

### 5. Callbacks, mocks, and spies

The examples in `src/test/doubles/OtherUtils.test.ts` and `MockModules.test.ts` show three related tools. They help answer different questions: “Was this function called?”, “What did it receive?”, and “Can I control what a dependency does?”

#### Callbacks: a function passed into another function

In `src/app/doubles/OtherUtils.ts`, `toUpperCaseWithCb` receives two things: a string and a callback. It calls the callback with an error message when the string is empty. For a non-empty string, it calls the callback with a status message, then returns the uppercased string.

The tests in `OtherUtils.test.ts` check both pieces of behavior. With an empty string, the function returns `undefined`, calls the callback with `Invalid argument!`, and calls it exactly once. With `abc`, it returns `ABC`, calls the callback with `called function with abc`, and again calls it once. The test is checking that the callback is used correctly, not just that the final string is right.

The file teaches two ways to keep track of callback calls:

- `jest.fn()` makes a tiny recorder for you. Jest stores its calls so the test can use `toBeCalledWith(...)` and `toBeCalledTimes(1)`.
- A hand-written callback can do the same job manually. The example pushes each message into `cbArgs` and increases `timesCalled`; after the test, `afterEach` empties both pieces of tracking state so the next test starts clean.

Use `jest.fn()` when you want Jest to record calls with little setup. The hand-written version is useful for seeing what a mock is doing behind the scenes.

#### Mock functions: record a call or choose the result

A **mock function** is a replacement function that Jest can inspect. For example, `jest.fn()` starts as a function with no special behavior, but Jest remembers when it was called and with what arguments. You can also teach it what to return.

`src/test/server_app/handlers/RegisterHandler.test.ts` uses this approach for `responseMock.writeHead`, `responseMock.write`, and `authorizerMock.registerUser`. The test arranges a fake request, then configures the collaborators for just that test:

- `getRequestBodyMock.mockResolvedValueOnce(someAccount)` says: “When the request-body helper is awaited next, give it this account.”
- `authorizerMock.registerUser.mockResolvedValueOnce(someId)` says: “When registration is awaited next, give it this ID.”

After `handleRequest()` runs, the assertions check the observable response: the created status code, JSON content-type header, and response body containing `userId`. This gives the handler controlled inputs and checks what it sends back, without needing a real database or network request.

The `Once` in `mockResolvedValueOnce` matters: that prepared result is for one call only. It is especially handy when one test makes several calls or when separate tests need different outcomes. The invalid-account test only prepares an empty request body, then checks for a bad-request response. The unsupported-method test checks that neither the response nor the request-body helper is touched.

#### Spies: watch a real function

A **spy** created with `jest.spyOn(object, 'method')` watches a method that already exists. By default, it still calls the real method; it simply lets the test inspect that call afterward.

In `OtherUtils.test.ts`:

- The first spy watches `sut.toUpperCase`. The test calls it with `asa` and checks that the method received that argument.
- The second watches `console.log`. Calling `sut.logString('abc')` uses the real implementation, which logs `abc`; the spy lets the test check that `console.log` received it.
- The third spy adds `.mockImplementation(...)`. That replaces the original method body for this test, so calling `sut.callExternalService()` runs the supplied implementation instead. This example demonstrates replacing behavior, but does not assert the spy was called.

The practical distinction: use a spy when you want to watch a real object's method (or temporarily replace it); use a standalone `jest.fn()` when you want a simple fake function to pass into the code. In both cases, check calls when calls are part of the behavior you care about.

#### Module mocks: replace a dependency before the test runs

`src/test/doubles/MockModules.test.ts` uses `jest.mock(...)` to replace functions at the module boundary:

- The `OtherUtils` mock spreads in the real module with `jest.requireActual(...)`, then replaces only `calculateComplexity` with a function that always returns `10`. This is a **partial mock**. Its test proves the fake result is returned, while another test proves the real `toUpperCase` function was kept.
- The `uuid` mock makes `v4()` always return `123`. Since `toLowerCaseWithId('ABC')` combines the lowercase text with that ID, the expected result is always `abc123`, never a random value.

Mocking a dependency makes the test repeatable and lets it focus on the code under test. It also means the test is not checking the real mocked dependency; the real dependency needs its own tests.

One learning trap in this folder: `OtherUtils.test.ts` has `describe.only` around its spy tests. Jest therefore runs only that suite from this file, so the callback tests later in the same file are skipped until `.only` is removed. Also, the `src/test/doubles` folder is not selected by either npm test command in the current configuration; see [Which Tests Run?](#which-tests-run).

### 6. Async work and data access

Tests that call asynchronous functions are marked `async` and use `await`. The `DataBase` and data-access tests under `src/test/server_app/data` are good examples. They check stored data and CRUD actions: create, read, update, and delete.

Some tests replace the database or ID generator with mocks/spies. Methods such as `mockResolvedValueOnce` teach a mock what to return for one awaited call. Tests then check both the returned value and whether the dependency received the right arguments.

### 7. Handlers and server routing

The tests in `src/test/server_app/handlers` check request handling without starting a real network server. They supply fake request/response objects and mocked collaborators, then check status codes, headers, response bodies, and calls to dependencies. These are often called **unit** or **component** tests: the main class is real, while neighbors are controlled.

`src/test/server_app/server/Server.test.ts` checks routing and server lifecycle in a similar isolated way. The HTTP module and handlers are replaced with fakes, so these tests can ask questions such as “Did `/login` go to `LoginHandler`?” without opening a network port.

### 8. Request-level tests with wrappers

The three tests in `src/test/server_app2` exercise register, login, and reservation requests. `RequestTestWrapper` and `ResponseTestWrapper` act like small pretend HTTP request and response objects. The tests mock the HTTP server and database, then check the behavior across more of the app than a single handler test.

These are useful **request-level/component tests**, but they do not send real network traffic. Despite the `itest` command name, `npm run itest` runs these mocked request tests; it does not run the live-server example described next.

### 9. A real server integration test

`src/test/server_app3/Itest.test.ts` starts the server, sends HTTP requests with `fetch` (and the `makeAwesomeRequest` helper), then checks registration, login, and reservation workflows. This is an **integration test** because several real parts of the application work together over HTTP.

The test uses `beforeAll` to start the server and `afterAll` to stop it. It also shares a token and reservation ID between steps, which shows how a longer workflow can be tested. Because it uses port `8080`, the server must be available for the duration of the test.

### 10. Snapshots and custom matchers

`src/test/server_app3/Itest.test.ts` includes a **snapshot test**. Jest saves an expected representation of the reservation in `src/test/server_app3/__snapshots__`. Later runs compare the result to that saved file. If the snapshot changes, inspect the difference and decide whether the new output is correct before updating it.

`src/test/server_app3/customMatchers.test.ts` demonstrates **custom matchers**: project-specific checks such as `toHaveUser`. They let you express a repeated rule in a readable way. This file also demonstrates adding TypeScript types for those matchers.

## Which Tests Run?

Jest only runs files matched by the configuration for the command:

- `npm test` uses `jest.config.ts`. It selects tests under `src/test/server_app` and `src/test/server_app2`, plus `src/test/Utils.test.ts`.
- `npm run itest` uses `itest.config.ts`. It selects tests under `src/test/server_app2` and loads `src/test/server_app3/utils/config.ts` as setup.
- The lessons in `src/test/doubles`, `src/test/pass_checker`, and `src/test/server_app3` are not selected by either current config. To run those lessons as part of a suite, add their paths to the appropriate Jest `testMatch` configuration.

There is also a `describe.only` in `src/test/doubles/OtherUtils.test.ts`. `.only` tells Jest to run only that suite within that test file, hiding the other tests in the same file. Remove `.only` when you want all its lessons to run.

The `coverage/` folder contains generated coverage reports; it is output from Jest, not where the lessons or source code live.

## A Good Practice Loop

For each lesson, pick one test and trace these three things:

1. What input does the test arrange?
2. What code does it act on?
3. What result, error, or interaction does it assert?

Then change one input or expected result, run the relevant tests, and notice what Jest reports. Put the expected result back afterward. That small loop teaches both what the code does and how each testing tool works.