# **_Unit testing Guide_**

<br />

## **1 - testing a simple react component**

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

## **2 - testing a composite react component**

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
