+++
title = "Web Components Part 3"
date = "2023-11-11T12:30:00Z"
draft = false
description = "Part 3 of exploring the current state of Web Components: learnings and common gotchas"

tags = []
+++

> You can find the full examples used in this blog series [on Github](https://github.com/lpedrosa/webcomponents-blog-examples)

This is part 3 of my Web Components series:

- [Part 1: Defining a custom element - configuring and styling the element, exposing an API]({{<ref "/blog/web-components-part-1">}})
- [Part 2: Composing elements and state - building complex components, communicating between using events]({{<ref "/blog/web-components-part-2">}})
- Part 3: Learnings and common Gotchas - things I have learned while just using the vanilla APIs

For the final article in this series, we will go through some things I have learned while exploring
the current state of Web Components.

## Custom Elements so far

In the last two articles we learned that you can create custom HTML elements using the browser's
native [Web Component APIs](https://developer.mozilla.org/en-US/docs/Web/API/Web_components).

Custom Elements behave just like any other built-in element, which means that:

- you can obtain a reference to them, using the [DOM Selector API](https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Locating_DOM_elements_using_selectors)
- you can configure them declaratively, using HTML attributes
- you can also configure them programmatically, using JavaScript properties

Custom Elements also offer some templating abilities using the Shadow DOM and the `<slot>` element.
This allows users to customize certain parts of a custom element without imposing changes to the
component's internal logic.

The Web Component APIs also allow your Custom Elements to react to [attribute changes](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements#responding_to_attribute_changes), allowing
you to re-configure the component and adapt its internal logic.

## Rendering

You might have noticed that the Web Components APIs do not impose any particular rendering strategy
on your Custom Element.

The examples we have seen so far, use declarative rendering, which looks something like this:

```js
customElements.define(
  "hello-world",
  class extends HTMLElement {
    constructor() {
      super();

      // here we "declare" how we want our element to look like
      this.innerHtml = `
        <p>Hello World</p>
      `;
    }
  }
);
```

But you can also use `document.createElement()` to render a custom element:

```js
customElements.define(
  "hello-world",
  class extends HTMLElement {
    constructor() {
      super();

      // here we "imperatively" build our custom element
      const p = document.createElement("p");
      p.textContent = "Hello World";

      this.appendChild(p);
    }
  }
);
```

With Web Components, you have **complete control over the rendering process of a component**. This
means you also decide what happens to the element, when its attributes or children change.

However, you will be left with some questions such as:

- Should a change in one of my custom element's attributes trigger a full render of its elements?
- Then, how do I render only the parts that have changed?
- How do I keep the rendering logic all in one place?

### Rendering, Rule of thumb:

This is my attempt to answer some of the above questions:

1. Use declarative rendering when possible, as it is easier to understand what HTML elements will
   the component render into, and what internal state the rendered elements depends on.
2. If you are using template literals for string interpolation, sanitize your inputs (none of the
   examples shown so far do that for simplification purposes)[^1]
3. When performance becomes a concern, you can always fallback to imperative updates e.g.
   `document.querySelector` the parts you want to change

## State

Custom Elements can have internal state (e.g. through a private instance variable). However, if you
notice carefully, most of the built in components expose their different states through attributes,
for example:

- [`<input type="checkbox">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/checkbox)
  and its `checked` attribute
- [`<input type="range"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/range)
  and its controls i.e. `min`, `max`, `step`

React gets it right by saying that you should think of your UI components as function of state. I
strongly recommend reading through their [Managing State](https://react.dev/learn/reacting-to-input-with-state)
series.

In that sense, built in components follow the same idea. When you change the component's attributes,
the UI _reacts_ and its state changes.[^2]

For this last part, I wanted to implement a stateful application, in order to figure out what is the
most intuitive way to manage state using Custom Elements.

I created a simplified [TODO application](https://github.com/lpedrosa/webcomponents-blog-examples/tree/master/part-3),
where:

- The TODOs _source-of-truth_ is an external API (simulated in the example)
- The `<todo-app>` element is the state manager and the entry point for this application
- The UI state of the remaining custom elements is controlled through attributes e.g. `loading`
- If the custom element is a list, the state also uses a combination of slotted children and
  handling the `slot-changed` event

### Struggling with lists

Regarding the last point, I initially went with a custom list that keeps an internal reference
to the TODO's it manages ([peek inside `<todo-list>`](https://github.com/lpedrosa/webcomponents-blog-examples/blob/master/part-3/src/components/todo-list.js)).

The main issue was that you couldn't manage the list items in a declarative way, for example:

```html
<!-- ❌ this wasn't possible -->
<todo-list>
  <todo-holder todo-id="1" content="This is a TODO"></todo-holder>
  <todo-holder todo-id="2" content="Another TODO"></todo-holder>
</todo-list>
```

Instead, you had to use the `<todo-list>` programmatic API to manage its internal items:

```html
<todo-list></todo-list>

<script>
  const todos = [
    {
      todoId: 1,
      content: "This is a TODO",
    },
    {
      todoId: 2,
      content: "Another TODO",
    },
  ];

  const todoList = document.querySelector("todo-list");
  todoList.replaceTodos(todos);
</script>
```

After thinking a bit about this, I created [second version](https://github.com/lpedrosa/webcomponents-blog-examples/blob/master/part-3/src/components/todo-list-v2.js)
using a `<slot>` based implementation.

You can still manage the individual items through the programmatic API, but also by declaring them
directly in the HTML:

```html
<!-- ✅ declaring the items is now possible! -->
<todo-list>
  <todo-holder slot="item" todo-id="1" content="This is a TODO"></todo-holder>
  <todo-holder slot="item" todo-id="2" content="Another TODO"></todo-holder>
</todo-list>

<script>
  // you can still use the element's API
  const todoList = document.querySelector("todo-list");
  todoList.addTodo({ todoId: 3, content: "Nice stuff!" });

  const todoIdToRemove = 1;
  todoList.removeTodo(todoIdToRemove);

  // as well as the native DOM APIs
  // adding
  const newTodo = document.createElement("todo-holder");
  newTodo.todoId = todoId;
  newTodo.content = content;
  newTodo.slot = "item";

  todoList.appendChild(newTodo);

  // removing
  const todoToDelete = todoList.querySelector("todo-holder[todo-id='2']");
  todoToDelete.remove();
</script>
```

### State, Rule of thumb:

1. Keep your custom elements as pure as possible
2. Move most of the UI rendering state to attributes, especially if they are simple properties such
   as a `number` or `string`
3. If the properties are complex objects, manage them through a programmatic API
4. If you want to support declarative rendering, you can use the `<slot>` technique described above
5. If you have state that influences several elements, _lift it_ to a parent element e.g.
   `<todo-app>`

## Wrapping Up

We conclude this series by exploring some questions that might have come up, while reading the
previous parts such as _what should be my rendering strategy?_ or _where should I store state?_

Web Components are very flexible and because of that, they do not enforce a particular style or
strategy.

By investigating how built in browser elements operate, and how to use them effectively, you can
stick to the provided DOM APIs to build your own custom elements.

With this final part, we should have hopefully gave you some guidelines to effective leverage Web
Components to build you own custom elements and rich UI applications.

All of this backed by your pre-existing DOM API knowledge.

[^1]: Alternatively, you can assume the inputs come from a trusted source. Ha!
[^2]:
    This might be why Web Components only provides the `observedAttributes` and
    `attributeChangedCallback` features.
