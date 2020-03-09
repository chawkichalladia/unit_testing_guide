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

## **3 - testing a simple react component**

simple react components only using html in their
structure without relying on other components.

an example of a react simple component is this `SimpleBtn` component:

```typescript
import React from "react";
import { FilterBtnProps } from "../../models/PropsModel";
import Styles from "../../Styles";

const FilterBtn: React.FC<FilterBtnProps> = ({
  filtered,
  onOpenFilter,
  onRemoveFilter
}) => {
  const btnStyle = filtered ? Styles.DangerBtn : Styles.PrimaryBtn;

  const text = filtered ? "Remove filter" : "Filter";

  const onClick = (): void => {
    if (filtered) {
      onRemoveFilter();
    } else onOpenFilter();
  };

  return (
    <button
      type="button"
      onClick={onClick}
      style={{ ...Styles.filterBtn, ...btnStyle }}
    >
      {text}
    </button>
  );
};

export default FilterBtn;
```

this component can be test in one of two ways:

- using snapshot testing

```typescript
import React from "react";
import renderer from "react-test-renderer";
import SimpleBtn from "../../components/views/SimpleBtn";

it("snapshot", () => {
  const element = renderer.create(
    <SimpleBtn
      filtered={true}
      onOpenFilter={jest.fn()}
      onRemoveFilter={jest.fn()}
    />
  );

  expect(element).toMatchSnapshot();
});
```

this test will create a snapshot file containing the
structure that the component renders.

```typescript
exports[`snapshot 1`] = `
<button
  onClick={[Function]}
  style={
    Object {
      "color": "red",
      "marginLeft": "10px",
    }
  }
  type="button"
>
  Remove filter
</button>
`;
```

this test allows a visual inspection of the tested
component structure to verify that everything is in order.

since the component was rendered with the prop `warning`
having the value `true`, the text of the button is `Remove Filter`.

- using react testing library

```typescript
import React from "react";
import SimpleBtn from "../../components/views/SimpleBtn";
import { render } from "@testing-library/react";

it("react testing library", () => {
  const { queryByText, getByText } = render(
    <SimpleBtn
      filtered={true}
      onOpenFilter={jest.fn()}
      onRemoveFilter={jest.fn()}
    />
  );

  expect(queryByText("filter")).toBeNull();
  expect(getByText(/remove/i)).toHaveTextContent("Remove filter");
  expect(getByText(/remove/i)).toHaveStyle("color: red");
  expect(getByText(/remove/i)).not.toHaveStyle("color: blue");
});
```

in this test we assert the existence of certain elements
in the rendered structure of the component.

in this case we expect the button text to be `Remove filter` and not `filter` since
the warning prop was set to `true`.
we also expect that the button color would be `red` not
`blue`.
this allows us to verify the structure of the file
without the need to go through the structure visually.

<br />

## **4 - testing a composite react component**

composite react components are components that uses other components for their structure.

to turn the simple component `SimpleBtn` we can use the
button component provided by the antd library:

```typescript
import React from "react";
import { FilterBtnProps } from "../../models/PropsModel";
import { Button } from "antd";
import Styles from "../../Styles";

const ComplexBtn: React.FC<FilterBtnProps> = ({
  filtered,
  onOpenFilter,
  onRemoveFilter
}) => {
  const type = filtered ? "danger" : "primary";

  const text = filtered ? "Remove filter" : "Filter";

  const onClick = (): void => {
    if (filtered) {
      onRemoveFilter();
    } else onOpenFilter();
  };

  return (
    <Button type={type} onClick={onClick} style={Styles.filterBtn}>
      {text}
    </Button>
  );
};
```

this component `ComplexBtn` uses the Button component
from the antd library instead of the html button.

to test this component we will need to mock the antd
module and its Button component. then we need to apply
one of the 2 methods described before.

- using snapshot testing

```typescript
import React from "react";
import renderer from "react-test-renderer";
import ComplexBtn from "../../components/views/ComplexBtn";

jest.mock("antd", () => ({
  Button: "MockBtn"
}));

it("ComplexBtn snapshot", () => {
  const element = renderer.create(
    <ComplexBtn
      filtered={true}
      onOpenFilter={jest.fn()}
      onRemoveFilter={jest.fn()}
    />
  );

  expect(element).toMatchSnapshot();
});
```

this test will render the `ComplexBtn` component
substituting the `Button` component imported from `antd`
with the mocked version `MockBtn` as provided in the mock
function.

```typescript
exports[`ComplexBtn snapshot 1`] = `
<MockBtn
  onClick={[Function]}
  style={
    Object {
      "marginLeft": "10px",
    }
  }
  type="danger"
>
  Remove filter
</MockBtn>
`;
```

in the snapshot we can see the mocked component with its
props and children.

- using react testing library

using testing library we will need to verify the existence
of the needed components, as well as verify that the
mocked component have been called correctly.

```typescript
import React from "react";
import ComplexBtn from "../../components/views/FilterBtn";
import { render } from "@testing-library/react";
import { Button } from "antd";
import Styles from "../../Styles";

jest.mock("antd");
it("ComplexBtn react testing library", () => {
  const mockButton = Button as jest.Mocked<typeof Button>;

  render(
    <ComplexBtn
      filtered={true}
      onOpenFilter={jest.fn()}
      onRemoveFilter={jest.fn()}
    />
  );

  expect(mockButton).toHaveBeenCalledTimes(1);
  expect(mockButton).toHaveBeenLastCalledWith(
    {
        ...mockButton.defaultProps
      type: "danger",
      onClick: expect.any(Function),
      style: Styles.filterBtn,
      children: "Remove filter",
      "data-testid": "filter-btn",
    },
    {}
  );
});
```

in this test we verify that the `Button` component from
antd have only been called once on the render of
`ComplexBtn` and have been called with the default props
as well as the props that we defined for it.

## **5 - testing event handlers**

for demonstration purposes we will be testing the following component:

```typescript
import React, { useState } from "react";
import useTodoStore from "../stores/TodoStore";
import useInputValidation from "../hooks/useInputValidation";
import TodoFormView from "./views/TodoFormView";

const TodoForm: React.FC = () => {
  //initialize state
  const [title, setTitle] = useState("");
  const [status, setStatus] = useState(false);

  //state update function
  const saveTodo = useTodoStore(state => state.saveTodo);

  // form validation
  const [warning, validateInputLength, , onBlurInput] = useInputValidation();

  // form event handlers
  const onInput = (e: React.ChangeEvent<HTMLInputElement>): void => {
    setTitle(e.target.value);
    if (e.target.value.length > 3) validateInputLength(e.target.value);
  };

  const onCheckBox = (): void => {
    setStatus(!status);
  };

  const submitForm = (e: React.FormEvent): void => {
    try {
      e.preventDefault();
      if (title.length > 3) {
        saveTodo({
          title,
          status
        });
        setTitle("");
        setStatus(false);
      }
      validateInputLength(title);
    } catch (error) {
      console.trace(error);
    }
  };

  return (
    <TodoFormView
      status={status}
      warning={warning}
      title={title}
      onBlurInput={onBlurInput}
      onCheckBox={onCheckBox}
      onInput={onInput}
      submitForm={submitForm}
    />
  );
};

export default TodoForm;
```

since this component relies on different external modules,
we will need to mock those module before starting our test.

our testing file will look like this:

```typescript
import React from "react";
import { render, act, RenderResult } from "@testing-library/react";
import TodoForm from "../../components/TodoForm";
import * as useTodoStore from "../../stores/TodoStore";
import * as useInputValidation from "../../hooks/useInputValidation";
import * as TodoFormView from "../../components/views/TodoFormView";

jest.mock("../../stores/TodoStore");
jest.mock("../../hooks/useInputValidation");
jest.mock("../../components/views/TodoFormView");

describe.only("<TodoForm />", () => {});
```

for this test we are importing the `act` hook from
`@testing-library/react` which will allow for handling
state changes.

we also imported the `RenderResult` interface for type
casting.

we also imported:

- `useTodoStore` hook for state management

- `useInputValidation` custom hook for managing input validation

- `TodoFormView` the child component of the `TodoForm`

after the imports are done we start our test by mocking
all the external modules, being `useTodoStore`,
`useInputValidation` and `TodoFormView`.
mocking those modules will guarantee that our component
is being tested separately from the other components and
modules.

the first step in our test is to assign the `Mocked` type
to our mocked components:

```typescript
const mockUseTodoStore = useTodoStore as jest.Mocked<typeof useTodoStore>;
const mockUseInputValidation = useInputValidation as jest.Mocked<
  typeof useInputValidation
>;
const mockTodoFormView = TodoFormView as jest.Mocked<typeof TodoFormView>;
```

next we will need to create jest mock functions to be used
as the return values for the custom hook
`useInputValidation` and the zustand `useTodoStore` hook,
as well as a variable for storing the rendered component:

```typescript
const validate = jest.fn();
const blur = jest.fn(),
  save = jest.fn();

let rendered: RenderResult = {} as RenderResult;
```

after defining the variables that we need for our test,
we need to write the necessary actions to be taken in our
test hooks `beforeEach` and `afterEach` :

```typescript
afterEach(() => {
  jest.clearAllMocks();
  jest.restoreAllMocks();
});

beforeEach(() => {
  mockTodoFormView.default.mockReturnValue(null);
  mockUseInputValidation.default.mockReturnValue([false, validate, null, blur]);
  mockUseTodoStore.default.mockImplementation(arg => {
    if (arg.toString().includes("state.saveTodo")) return save;
  });

  jest.spyOn(console, "trace").mockImplementation(() => null);

  rendered = render(<TodoForm />);
});
```

in the `afterEach` hook, we are clearing and restoring our
mocks to their initial state so the test won't interfere
with each other.

as for the `beforeEach` hook, we are:

- assigning a null as return value for `TodoFormView`
  component.

- assigning an array as the return value for the
  `useInputValidation` hook

- mocking the implementation of the `useTodoStore` hook

- mocking the `trace` method on the `console` object to
  test for error handling

- and finally storing the rendered component in the
  `rendered` variable

after the setup is complete, we can start writing our test:

```typescript
it("onInput is fires the proper functions with target.value length =< 3", () => {
  const INPUT_VALUE = "abc";

  const inputEvent: React.ChangeEvent<HTMLInputElement> = {} as React.ChangeEvent<
    HTMLInputElement
  >;
  inputEvent.target = {} as EventTarget & HTMLInputElement;
  inputEvent.target.value = INPUT_VALUE;
  act(() => mockTodoFormView.default.mock.calls[0][0].onInput(inputEvent));
  expect(validate).not.toBeCalled();

  expect(mockTodoFormView.default).toBeCalledTimes(2);
  expect(mockTodoFormView.default).toHaveBeenLastCalledWith(
    expect.objectContaining({ title: INPUT_VALUE }),
    {}
  );

  // no stack trace error message have been called
  expect(console.trace).not.toHaveBeenCalled();
});
```

in this test, we are testing the `onInput` eventHandler
that was passed to the `TodoFormView` as prop.

for this test we will study the case where the input text
length is less than or equal 3.

to test this component we first need to create the event
object that will be passed as argument to the `onInput`
event handler.

after defining the event object, we will need to call the
the event handler through the props passed to
`TodoFormView` mock component. this method call will need
to wrapper in the `act` hook to handle the state changes
resulted by this action.

after our call we can start writing the assertions we
need to make about our code.

first we need to verify that the `validate` mock function
have not been called since the length of the input is 3.

then we will need to verify that `TodoFormView` have been
called twice, once for the initial render and once after
the state have been updated.

we will also need to verify that the `TodoFormView`
component have been called with the new input text as the
value for the `title` prop.

finally we assert that the `console.trace` method have
not been called as a precaution for any unwanted side
effects.
