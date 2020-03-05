# **_<center>Unit testing Guide</center>_**

<br />

## **1 - Testing Services and API calls**

the service we are going to test:

```typescript
export const fetchTodos = async (
  set: Function,
  get: Function
): Promise<object> => {
  set({ fetching: true });
  return todoReturn(await Axios.get(todoURLWithFilterAndSort(null, get)));
};
```

### _a - mock external modules_

To Test a service function or an API call, any module that is used to make a call
to the server or to alter sent and received data should be mocked.

```typescript
jest.mock("axios");
jest.mock("../helpers/urlHelpers");
```

### _b - clear all mocks after each test_

In order to avoid interference between different tests, mocks should be cleared
after each one.

```typescript
describe("test todo service", () => {
  afterEach(() => {
    jest.clearAllMocks();
  });
});
```

### _c - writing the test_

First, for typescript the mocks should be typed:

```typescript
// mock axios module
const MockAxios = axios as jest.Mocked<typeof axios>;
// mock urlHelpers module
const MockUrlHelpers = urlHelpers as jest.Mocked<typeof urlHelpers>;
```

then, any arguments passed to the service function should be defined as well as
any desired return value for the mock functions (if a passed argument is a
function, then a jest mock function should be passed instead):

```typescript
const axiosResponse = { data: { data: [], total: 0 } };
const get = jest.fn(),
  set = jest.fn();
```

after defining the potential arguments for the tested function and return values
for the mocked modules, now time to mock the return values for each mocked value:

```typescript
MockAxios.get.mockResolvedValueOnce(axiosResponse);
MockUrlHelpers.todoURLWithFilterAndSort.mockReturnValueOnce("");
```

after the setup is complete, we need to call the tested function to test its behavior:

```typescript
const res = await todoService.fetchTodos(set, get);
```

after the call, we start testing the expected behavior:

```typescript
expect(MockAxios.get).toHaveBeenCalledTimes(1);
expect(MockAxios.get).toHaveBeenCalledWith("");

expect(urlHelpers.todoURLWithFilterAndSort).toHaveBeenCalledTimes(1);
expect(urlHelpers.todoURLWithFilterAndSort).toHaveBeenCalledWith(null, get);

expect(set).toHaveBeenCalledTimes(1);
expect(set).toHaveBeenCalledWith({ fetching: true });

expect(MockFormatReturnedData.todoReturn).toHaveBeenCalledTimes(1);
expect(MockFormatReturnedData.todoReturn).toHaveBeenLastCalledWith(
  axiosResponse
);
```

we test each mocked module is being called as expected by the tested module by
testing the number of calls and passed arguments

finally, the complete test will look like this:

```typescript
import * as todoService from "../services/todoService";
import * as urlHelpers from "../helpers/urlHelpers";
import * as formatReturnedData from "../helpers/formatReturnedData";
import axios from "axios";

jest.mock("axios");
jest.mock("../helpers/urlHelpers");
jest.mock("../helpers/formatReturnedData");

describe("test todo service", () => {
  afterEach(() => {
    jest.clearAllMocks();
  });

  test("fetchTodos()", async () => {
    // mock axios module
    const MockAxios = axios as jest.Mocked<typeof axios>;
    // mock urlHelpers module
    const MockUrlHelpers = urlHelpers as jest.Mocked<typeof urlHelpers>;
    const MockFormatReturnedData = formatReturnedData as jest.Mocked<
      typeof formatReturnedData
    >;

    // define the response for the get request and the mocks for the state methods
    const axiosResponse = { data: { data: [], total: 0 } };
    const get = jest.fn(),
      set = jest.fn();

    // mock resolved and rejected values for axios get and the url helper
    MockAxios.get.mockResolvedValueOnce(axiosResponse);
    MockUrlHelpers.todoURLWithFilterAndSort.mockReturnValueOnce("");

    // trigger the fetchTodos function
    const res = await todoService.fetchTodos(set, get);

    // assertions
    // assert that axios have been called properly
    expect(MockAxios.get).toHaveBeenCalledTimes(1);
    expect(MockAxios.get).toHaveBeenCalledWith("");

    // assert that todoURLWithFilterAndSort have been called properly
    expect(urlHelpers.todoURLWithFilterAndSort).toHaveBeenCalledTimes(1);
    expect(urlHelpers.todoURLWithFilterAndSort).toHaveBeenCalledWith(null, get);

    // assert that state functions have been called properly
    expect(set).toHaveBeenCalledTimes(1);
    expect(set).toHaveBeenCalledWith({ fetching: true });

    // assert the proper value is returned
    expect(MockFormatReturnedData.todoReturn).toHaveBeenCalledTimes(1);
    expect(MockFormatReturnedData.todoReturn).toHaveBeenLastCalledWith(
      axiosResponse
    );
  });
});
```
