## React vs Ember: State Management

_This is the second post in a segmented series on react (and react's ecosystem) vs ember. To read the previous, [go here to read about Event Handling](https://github.com/NullVoxPopuli/react-vs-ember/tree/master/event-handling).  The *Event Handling* post is not required reading for this post, but makes the same general assumptions about prior knowledge ([SPAs](https://en.wikipedia.org/wiki/Single-page_application), [ECMAScript](https://en.wikipedia.org/wiki/ECMAScript), etc). [See the series intro here](https://github.com/NullVoxPopuli/react-vs-ember#react-vs-ember-introduction)._

Managing application state can easily become non-trivial over time, so it’s important to have a good foundation to build upon. Without knowing the options and how to manage them, trying to establish a foundation will result in a lack of testability and greatly reduced maintainability. (Testing will be covered in the next segment of this series)

This post will demonstrate two different ways of managing application state: With and without redux and what using or not using redux means in both the react and ember ecosystems. For a refresher on why anyone would want to use redux, read the [motivation page from redux's website](https://redux.js.org/introduction/motivation). The purpose of the comparison is to demonstrate the similarities and differences between react and ember with both redux and the ecosystem-native state management approach.

All projects used in this blog post implement [TodoMVC](http://todomvc.com/), which is a simple todo app with some styling and predefined behavior. The code can be [viewed here](https://github.com/NullVoxPopuli/react-vs-ember/tree/master/state-management).


Because redux is just javascript, there are negligible differences between the react and ember worlds. As shown [in the repo folder for this blog post](https://github.com/NullVoxPopuli/react-vs-ember/tree/master/state-management), the react and ember redux implementations use the _exact_ same `redux-store` folder.


Something something about application state, and why state management something something



### The custom redux layout

Officially, [redux](https://redux.js.org) has [no opinion](https://redux.js.org/faq/code-structure) on code structure. The most commonly seen structure is the 'rails style', where there is an `actions` folder, and a `reducers` folder -- resulting in segregation of your feature-code. While this pattern may be easy to implement (i.e.: low effort), it is not suitable for maintainability and discovery.

The following structure is the 'ducks' structure mentioned in the redux docs -- it's really a feature-based domain-concept layout that focuses on grouping related behavior rather than grouping by 'the type of the thing' (all actions in an 'actions' folder, for example).

To see implementation details, specifically for managing imports and (re)exports, the code is available for browsing [here](https://github.com/NullVoxPopuli/react-vs-ember/tree/master/state-management/ember-redux/src/redux-store).

```bash
redux-store/
│
│   # re-exports the main things for public-api and redux-store creation
├── index.ts
│   # devtools setup
├── enhancers.ts
│   # setup sagas, thunks, logging, etc
├── middleware.ts
│   # all reducers combined here
│   # this is the only place where combineReducers is called,
│   # and this defines the top level 'State' type
├── reducers.ts
│
│   # a domain concept
└── todos/
    │
    │   # re-exports the public api things (action creators, the reducer).
    │   # also defines the 'ActionTypes' type which constrains the
    │   # reducer for todos to a fixed set of types for each possible action
    │   # available on the todos 'namespace'
    ├── index.ts
    │   # selectors are helper functions for retrieving data out of the global
    │   # redux state. these should be pure functions, just like the reducers.
    ├── selectors.ts
    │   # defines the domain concept namespace, types and initial state.
    ├── shared.ts
    │
    └── actions/
        │
        │   # Each action contains everything that is needed for
        │   # a particular behavior.
        │   # - action type constant
        │   # - action creator
        │   # - reducer
        │   # - sagas (if being used)
        ├── add.ts
        ├── clear-completed.ts
        ├── complete.ts
        ├── destroy.ts
        ├── edit.ts
            ... etc
```
## Redux (Both React and Ember)

Redux is vanilla javascript, so it can be used with any library -- it doesn't even have to be used in a frontend context. 

The parts that do differ and tie redux into the frontend come from the corresponding packages: [react-redux](https://github.com/reduxjs/react-redux) and [ember-redux](https://github.com/ember-redux/ember-redux). These packages provide convenience and ease of setup by making assumptions about the development environment and providing some general abstractions and configurations.

Next is a comparison of the similarities and differences of using redux in both react and ember.

### Without a container

The following example shows usage without a container, or rather, since naming in programming is very subjective, it defines a container that it used to render a list of things.  The reason these aren't traditionally containers is because they each contain a template and that template is somewhat coupled to the rendered content (`ul` has `li`s). The focus here should be that the `@connect` usage is the exact same in both environments.

#### Usage in React
[react/src/ui/components/todo-list.tsx](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/react/src/ui/components/todo-list.tsx)
```tsx
// imports omitted

const mapStateToProps = (state: State) => ({
  todos: list(state)
});

@connect(mapStateToProps)
export default class TodoList extends React.Component<Props> {
  render() {
    const { todos } = this.props;

    return (
      <ul className='todo-list'>
        {todos.map((t, i) => <TodoItem key={i} todo={t} />)}
      </ul>
    );
  }
}
```


#### Usage in Ember
ember/src/ui/components/todo-list/{ [component.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember-redux/src/ui/components/todo-list/component.ts) | [template.hbs](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember-redux/src/ui/components/todo-list/template.hbs) }
```ts
// imports omitted
@connect(stateToComputed)
export default class TodoListComponent extends Component {
  tagName = 'ul';
  classNames = ['todo-list'];
}

```
```hbs
{{#each todos as |todo|}}
  <TodoItem @todo={{todo}} />
{{/each}}
```

### With a container
This next example demonstrates how one might abstract away all store/state logic into a container. This provides the benefit that the component can be unit tested without having to create the entire store. Enhanced composability can be achieved If a display or container component could be generic enough that they could be mixed and matched with other display and container components — higher-order-components fit in to this category of containers, but that’s a topic for another time.


#### Usage in React

[src/ui/components/todo/index.tsx](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/react/src/ui/components/todo/index.tsx)
```tsx

import * as React from 'react';
import { connect } from 'react-redux';

import { edit, destroy, toggle } from '@store/todos';

import TodoDisplay from './display';

const mapDispatchToProps = (dispatch) => ({
  destroyTodo: (id: number) => dispatch(destroy(id)),
  toggleCompletion: (id: number) => dispatch(toggle(id)),
  editTodo: (id: number, text: string) => dispatch(edit(id, text))
});

export default connect(null, mapDispatchToProps)(TodoDisplay);
```
[src/ui/components/todo/display.tsx](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/react/src/ui/components/todo/display.tsx)
```tsx
// file heavily abbreviated

export default class TodoDisplay extends React.Component<Props, State> {
  state = { editing: false };

  didFinishEditing = (e:  React.FocusEvent<HTMLInputElement>) => {
    const { editTodo, todo: { id } } = this.props;

    const text = e.target.value;

    editTodo(id, text);
    this.setState({ editing: false });
  }

  didDoubleClickLabel = () => {
    this.setState({ editing: true });
  }

  // ... other action handlers ...

  render() {
    // actions retrieved from props from the container
    const { todo, destroyTodo, toggleCompletion } = this.props;

    // ... template omitted
  }
}

```


#### Usage in Ember

src/ui/components/todo-item/{ [component.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember-redux/src/ui/components/todo-item/component.ts) | [template.hbs](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember-redux/src/ui/components/todo-item/template.hbs) }
```ts
import Component from '@ember/component';
import { action } from '@ember-decorators/object';

import { connect } from 'ember-redux';
import { edit, destroy, toggle } from 'example-app/src/redux-store/todos';

const dispatchToActions = {
  deleteTodo: destroy,
  completeTodo: toggle,
  editTodo: edit
}

@connect(null, dispatchToActions)
export default class TodoItemContainer extends Component {
  tagName = 'li';
  editing = false;
  classNameBindings = ['todo.completed', 'editing'];

  @action
  startEditing(this: TodoItemContainer) {
    this.set('editing', true);
  }

  @action
  doneEditing(this: TodoItemContainer) {
    this.set('editing', false);
  }
}
```
```hbs
<TodoDisplay
  @todo={{todo}}
  @props={{hash
    deleteTodo=(action "deleteTodo" todo.id)
    completeTodo=(action "completeTodo" todo.id)
    editTodo=(action "editTodo" todo.id)  
  }}
/>
```

src/ui/components/todo-item/display/{ [component.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember-redux/src/ui/components/todo-item/display/component.ts) | [template.hbs](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember-redux/src/ui/components/todo-item/display/template.hbs) }
```ts
export default class TodoItemDisplay extends Component {

  @action
  didClickLabel() {
    this.props.startEditing();
    this.send('focusInput');
  }

  @action
  didFinishEditing(e: KeyboardEvent) {
    const target = (e.target as HTMLInputElement);
    const text = target.value;

    this.props.editTodo(text);
    this.props.doneEditing();
  }
}
```



## Without Redux

### React

Redux is the go-to for application state management in react, but a fairly new API within react allows for sensible state-management without additional dependencies. [The Context API](https://reactjs.org/docs/context.html) allows state to be shared across multiple components.  Context's are loosely similar to Ember's Services (services will be covered below), in that they can define silo'd behavior and be used by components. There are some restrictions that context's have, however. One is that there must be a context Provider, which provides the configured context to all children in the tree below the Provider, and then there is the Consumer, which allow children of the Consumer in the immediate rendering context to access the Context's data.

Below is an example of how the `TodoMVC` app would be managed and interact with the `todos` state with a context instead of redux.

Instead of actions/reducers/etc, a context is configured, and given a value during invocation-time.
The `Application` component becomes the state manager for this particular app.

```tsx
// import omitted
export default class Application extends React.Component<{}, State> {
  constructor(props) {
    super(props);

    this.state = {
      todos: [],
      clearCompleted: this.clearCompleted,
      // ...
    }
  }

  clearCompleted = () => {
    const todos = this.state.todos.filter(t => !t.completed);

    this.setState({ todos });
  }

  // ... other actions omitted

  render() {
    return (
      <TodosContext.Provider value={this.state}>
        <TodoMVC />
      </TodosContext.Provider>
    );
  }
}

```

Using the context from any descendant in the tree looks like the following snippet.
The consumer expects a function which can use the short-hand argument deconstruction syntax to pull out only what is needed for that render.

```tsx
import * as React from 'react';

import HeaderDisplay from './display';
import { TodosContext } from '@contexts/todos';

export default class HeaderContainer extends React.Component {
  render() {
    return (
      <TodosContext.Consumer>
        {({ add }) => <HeaderDisplay addTodo={add} />}
      </TodosContext.Consumer>
    );
  }
}
```



### Ember

Ember comes with a few tools for managing application level state out of the box. The most common solution for application state is the [Service](https://guides.emberjs.com/release/applications/services/). Services are similar to the Context API that react provides (shown above), but does not require to be _provided_ to child components, as ember supports [dependency injection](https://guides.emberjs.com/release/applications/dependency-injection/), allowing for easier testing, and less boilerplate when hooking pieces of the app together.

The other side to the provided state management is the supplemental package, [ember-data](https://github.com/emberjs/data). It provides a frontend ORM for managing data models that can be backed by varying data sources via [adapters and serializers](https://emberigniter.com/fit-any-backend-into-ember-custom-adapters-serializers/). In the following sample code, ember-data is only used to manage the list of todos locally in the browser.

[/src/data/models/todo.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember/src/data/models/todo.ts)
```ts
import Model from 'ember-data/model';
import { attr } from '@ember-decorators/data';

export default class Todo extends Model {
  @attr text?: string;
  @attr completed?: boolean;
}
```

Below is a service that abstracts the ember-data store and provides convenience functions for the `TodoMVC` problem space.

[/src/services/todos.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember/src/services/todos.ts)
```ts
export default class TodosService extends Service {
  @service store!: DS.Store;

  find(id: ID): Todo | null {
    return this.store.peekRecord('todo', id);
  }

  destroyTodo(id: ID) {
    const record = this.find(id);

    if (record) {
      record.deleteRecord();
    }
  }

  // other functions omitted
}
```

The usage of the service is provided using the `@service` decorator which dependency injects the service into the Component.
[/src/ui/components/header/component.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember/src/ui/components/header/component.ts)
```ts
import Component from '@ember/component';
import { service } from '@ember-decorators/service';
import { action } from '@ember-decorators/object';

import TodosService from 'example-app/services/todos';

export default class Header extends Component {
  @service todos!: TodosService;

  text = '';

  @action
  didSubmit(this: Header) {
    this.todos.add(this.text);
    this.set('text', '');
  }
}
```

The last piece of the built-in state management that ember provides is [routing](https://guides.emberjs.com/release/routing/defining-your-routes/). The routing in this version of the app is responsible for which set of todos are being viewed, All, Active, or Completed. Routes did not _have_ to be used for the filtering of these lists, as one could set up a series of [computed properties](https://guides.emberjs.com/release/object-model/computed-properties/) with the filter value as the dependent key.

/src/ui/routes/completed/{ [route.ts](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember/src/ui/routes/completed/route.ts) | [template.hbs](https://github.com/NullVoxPopuli/react-vs-ember/blob/master/state-management/ember/src/ui/routes/completed/template.hbs) }
```ts
import DS from 'ember-data';
import Route from '@ember/routing/route';

import Todo from 'example-app/ui/data/models/todo';

export default class CompletedRoute extends Route {
  store!: DS.Store;

  model() {
    return this.store
      .peekAll('todo')
      .filter((todo: Todo) => todo.completed);
  }
}
```
```hbs
<section class='main'>
  <TodoList @todos={{model}} />
</section>

<Footer @todos={{model}} />
```



## State Management: Final Thoughts

Both React and Ember provide state management out of the box, but Redux provides some shnazzy debugging abilities ([time-travel](https://medium.com/the-web-tub/time-travel-in-react-redux-apps-using-the-redux-devtools-5e94eba5e7c0)) due to its immutable nature of handling changes. The predicable and traversable state changes bring sanity to debugging when multiple areas of an app may be trying to change something at relatively the same time.  On the flip side, Redux has had many questions on its [perfromance](https://github.com/redux-saga/redux-saga/issues/241#issuecomment-207202589) [with](https://github.com/reduxjs/redux/issues/768
)  [large](https://github.com/markerikson/react-redux-links/blob/master/react-performance.md
) [sets](https://somebody32.github.io/high-performance-redux/
) of data.

As with all things, it's important to keep in mind that the right tool should be used for the job. [Not everything needs to be in redux](https://redux.js.org/faq/organizing-state#organizing-state)
Using redux doesn't mean that other state-management should be avoided.
For example, when working with a couple form fields, whose state you don't care about in other components, using redux to manage the values and `onChange` events would cause unnecessary indirection / complication without any benefit.

There may be situations where you need all different kinds of state management. Maybe services or contexts could be used managing computed values, redux could control things like whether or not the sidebars are popped open in your app, or redux could supplement some other state management technique for interacting with websockets.  There is no wrong answer... unless the answer makes your life difficult.

It's all about the Developer Experience (DX).

So do what feels right for each situation.

## The End (of part 2)

In order to try to stay focused, I made a lot of assumptions about the readers' knowledge.
If you want to know more about anything mentioned in this post, feel free to tweet at me @NullVoxPopuli.
Maybe:
- why use TypeScript?
- why are the TypeScript definitions different?
- why does ember user separate template files?
- why does ember use handlebars instead of JSX?
- why does your container example for ember have local state management?
