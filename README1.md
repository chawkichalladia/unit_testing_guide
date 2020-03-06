# **_Unit testing Guide_**

<br />

## **1 - Testing Services and API calls**

the service we are going to test:

```typescript
import Axios from "axios";
import { todoURLWithFilterAndSort } from "../helpers/urlHelpers";
import { todoReturn } from "../helpers/formatReturnedData";

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

### _b - adding the afterEach hook to clear the mocks_

to avoid interference between different tests, mocks should be cleared
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

<br />

## **2 - Testing Zustand store**

the zustand store we are going to test:

```typescript
import create from "zustand";
import { fetchTodos } from "../services/todoService";

const [useTodoStore] = create((set, get) => ({
  todos: [],
  total: 0,
  fetching: false,
  getTodos: (): void => {
    fetchTodos(set, get)
      .then(data => set({ ...data, fetching: false }))
      .catch(err => console.log(err));
  }
}));

export default useTodoStore;
```

the first 2 steps to test the store is the same as the ones used to test the service:

- mocking the external modules

```typescript
jest.mock("../services/todoService");
```

- adding the afterEach hook

```typescript
describe("test todo store", () => {
  afterEach(async () => {
    await cleanup();
    jest.clearAllMocks();
  });
});
```

the difference between this test and the previous test is that in this case the
store is a react hook, so we need to use the `@testing-library/react-hooks`
testing library to do our tests.

so our setup would look like this in the test file:

```typescript
import { cleanup } from "@testing-library/react-hooks";

jest.mock("../services/todoService");

describe("test todo store", () => {
  afterEach(async () => {
    await cleanup();
    jest.clearAllMocks();
  });
});
```

where cleanup is an API used to unmount any rendered hooks.

now to writing the test itself.

first step is to type the mocks:

```typescript
const MockTodoService = todoService as jest.Mocked<typeof todoService>;
```

then we mock the returned value. to mock the value we use a helper function that
returns data in the expected form from the service.

```typescript
const todos = generateTodos(5);
MockTodoService.fetchTodos.mockResolvedValueOnce(todos);
```

in here we generate 5 todos and provide the data as returned value from the
mocked service function.

after establishing our mocked value, we render our hook:

```typescript
const { result } = renderHook(() =>
  useTodoStore(state => ({
    todos: state.todos,
    total: state.total,
    getTodos: state.getTodos
  }))
);
```

then we call the action we want to test:

```typescript
await act(async () => await result.current.getTodos());
```

the action we are calling is a state action, so we need to wrap our call in an
act closure which will wait for those state changes before returning the result
we need.

after the action we want to test is called, we begin writing our assertions:

```typescript
expect(todoService.fetchTodos).toHaveBeenCalledTimes(1);
expect(todoService.fetchTodos).toHaveBeenCalledWith(
  expect.any(Function),
  expect.any(Function)
);

expect(result.current.total).toBe(todos.total);
expect(result.current.todos).toEqual(todos.todos);
```

first, we start by asserting that the mocked service function has been called correctly,
then we assert that the state values are what we expect them to be.

in this case, we are expecting that the service function will be called a single
time, and will receive 2 functions are arguments.

on the other hand, we are expecting that the values saved to the store would match
the values passed to the service mock.

at the end, this how the complete test would look like:

```typescript
import { renderHook, act, cleanup } from "@testing-library/react-hooks";
import * as todoService from "../services/todoService";
import useTodoStore from "../stores/TodoStore";
import { generateTodos } from "../helpers/testUtils/dataGenerators";

jest.mock("../services/todoService");

describe("test todo store", () => {
  afterEach(async () => {
    await cleanup();
    jest.clearAllMocks();
  });

  it('"todos" and "total" state can be updated with "getTodos"', async () => {
    // mock todoService
    const MockTodoService = todoService as jest.Mocked<typeof todoService>;
    const todos = generateTodos(5);
    MockTodoService.fetchTodos.mockResolvedValueOnce(todos);
    const { result } = renderHook(() =>
      useTodoStore(state => ({
        todos: state.todos,
        total: state.total,
        getTodos: state.getTodos
      }))
    );

    await act(async () => await result.current.getTodos());

    // assertions
    expect(todoService.fetchTodos).toHaveBeenCalledTimes(1);
    expect(todoService.fetchTodos).toHaveBeenCalledWith(
      expect.any(Function),
      expect.any(Function)
    );

    expect(result.current.total).toBe(todos.total);
    expect(result.current.todos).toEqual(todos.todos);
  });
});
```

<br />

## **3 - Testing React View component**

### a - testing a simple component

simple components are components that do not use other components in their structure.

an example of a simple component is this `HeaderView` component:

```typescript
import React from "react";
import Styles from "../../Styles";
import { PageHeader, Typography } from "antd";
import { HeaderViewProps } from "../../models/component-props/HeaderProps";

const HeaderView: React.FC<HeaderViewProps> = ({ count }) => {
  return (
    <PageHeader
      title={
        <Typography.Title type="secondary" style={Styles.HeaderTitle}>
          Todos
        </Typography.Title>
      }
      style={Styles.Header}
      extra={
        <div style={Styles.Counter}>
          <span>pending({count})</span>
        </div>
      }
    />
  );
};

export default HeaderView;
```

the way to test this simple component is to to simply render it then verify what
was rendered. this component can be tested in 2 ways:

- using snapshot testing:

```typescript
import React from "react";
import { render } from "@testing-library/react";
import HeaderView from "../../components/views/HeaderView";

it("snapshot test", () => {
  const { baseElement } = render(<HeaderView count={countProp} />);

  expect(baseElement).toMatchSnapshot();
});
```

this creates a snapshot file for this test suite that
contains the snapshots created by each of the tests.

- using react testing library

if you already know what to expect from your component
you can simply assert that the expected content
is being rendered correctly, which takes us to the second
way of testing react components:

```typescript
import React from "react";
import { render } from "@testing-library/react";
import HeaderView from "../../components/views/HeaderView";

describe("Name of the group", () => {
  const countProp = 200;
  it("it displays the correct text", () => {
    const { getByText } = render(<HeaderView count={countProp} />);

    const pending = getByText(/^pending/i);

    expect(pending).toHaveTextContent(`pending(${countProp})`);
  });
});
```

here we render the component then we assert that it
displays the text correctly given certain props.
this wa we can assert that what we want is being rendered
without the need to go through a snapshot file that can
be long and complex depending on the components used
with the one we are currently testing.

### a - testing a composite component

composite components are components that are built using
other components (both simple and composite) to build
the structure needed.

an example of a composite component is `TodoFormView`:

```typescript
import React from "react";
import { Form, Input, Icon, Checkbox, Button, Divider } from "antd";
import FormInput from "../generic/FormInput";
import Styles from "../../Styles";
import { TodoFormViewProps } from "../../models/component-props/TodoFormProps";

const TodoFormView: React.FC<TodoFormViewProps> = ({
  warning,
  title,
  status,
  submitForm,
  onBlurInput,
  onInput,
  onCheckBox
}) => {
  // form layout config
  const formLayout = {
    wrapperCol: {
      xs: { span: 25, offset: 0 },
      sm: { span: 24, offset: 0 }
    }
  };

  return (
    <Form
      layout="inline"
      {...formLayout}
      onSubmit={submitForm}
      style={Styles.Form}
      data-testid="new-todo-form"
    >
      <div style={Styles.FormBody}>
        <FormInput warning={warning} style={Styles.InputContainer}>
          <Input
            suffix={<Icon type="alert" />}
            placeholder="Add a Todo"
            style={Styles.Input}
            value={title}
            onChange={onInput}
            onBlur={onBlurInput}
          />
        </FormInput>
        <Form.Item>
          <Checkbox
            checked={status}
            style={Styles.CheckBox}
            onChange={onCheckBox}
            data-testid="new-todo-status"
            value={status}
          />
        </Form.Item>
        <Form.Item>
          <Button
            style={Styles.Button}
            icon="save"
            htmlType="submit"
            data-testid="save-todo"
          >
            Save Todo
          </Button>
        </Form.Item>
      </div>
      <Divider style={Styles.Divider} data-testid="divider" />
    </Form>
  );
};

export default TodoFormView;
```

this component uses the `FormInput` component as a building block for its
structure.

testing this component can be done using one of 2 methods:
snapshot testing or using `react testing library`.

to test this component, no matter which method we are
going to use, we will need to mock the `Forminput` component that is being imported.
react testing library

- using snapshot test

```typescript
import React from "react";
import { render } from "@testing-library/react";
import TodoFormView from "../../components/views/TodoFormView";

jest.mock("../../components/generic/FormInput", () => "MockFormInput");

it("snapshot", () => {
  const { baseElement } = render(
    <TodoFormView
      status={false}
      submitForm={jest.fn()}
      warning={false}
      onBlurInput={jest.fn()}
      onCheckBox={jest.fn()}
      onInput={jest.fn()}
      title="title"
    />
  );

  expect(baseElement).toMatchSnapshot();
});
```

in this test we just assign a new name to the mock

- using react testing library
