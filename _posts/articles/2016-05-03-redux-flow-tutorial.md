---
layout: post
title: "Redux Flow Tutorial"
modified:
categories: articles
excerpt:
tags: [facebook, flow, react, redux, jsx]
image:
  feature:
date: 2016-05-03T20:49:00-00:00
---

# Redux Flow Tutorial

In this tutorial I'm going to show you how you can add Facebook's [Flow](http://flowtype.org/) to your [Redux](https://github.com/reactjs/redux) work-flow, and stop using the older `propTypes` feature for JSX template validation. Although [it's possible](https://github.com/facebook/immutable-js/issues/203#issuecomment-187175318) to use [Immutable.js](https://facebook.github.io/immutable-js/) with Flow, using something like Immutable.js reduces the insight Flow has into your code, and so reduces the number of bugs that it will detect. Instead, I'm going to use ES6/7 + [Ramda](http://ramdajs.com/), which has the additional benefit that you can increase the amount of functional re-use you can achieve due to Ramda's excellent [auto-currying support](http://fr.umio.us/why-ramda/), and that you'll be using native JavaScript types instead of Immutable's proprietary types.

It won't all be plain sailing though, and you'll see along the way that Flow and the Flow infrastructure still have a few rough edges, but that Flow often has an insight into your code that you won't have experienced with traditional compilers.

If you don't have time to follow the tutorial you can just read along. The [redux-flow-tutorial](https://github.com/dchambers/redux-flow-tutorial) contains all of the resultant source code for you to look at, with commits at the various milestones throughout the article so you can track how we arrive at the final conclusion.


## What about TypeScript?

At this point it's worth talking about TypeScript. TypeScript also provides static typing for JavaScript, but uses a type system that started life being much more similar to what you get with Java and C#, and so didn't work with lots of idiomatic JavaScript. More recently, TypeScript has begun adding the same type of features you find in Flow (like _union-types_, _intersection-types_ and _action-guards_), but it still hasn't achieved parity, and it still can't be used to type an idiomatic Redux reducer, or lots of other idiosyncratic JavaScript patterns we find in the wild.

There's nothing particularly wrong with that, but it does mean that TypeScript is better suited for Angular 2 development then it is for Redux development right now, though the two do seem to be slowly converging. However, the thing that draws me to Flow is that it fits better into the NPM eco-system, and can be used alongside stellar tools like Babel and ESLint. This is a shame because at this point TypeScript has a more mature eco-system than Flow; partly due to it being an older project, but also because it's a less technically challenging endeavour.


## Before We Start

Before we start, let's upgrade to a recent version of Node.js so we can (almost) have ES6 support without using a transpiler:

~~~bash
nvm install 5.11.0
~~~

If you don't use NVM then you may want to upgrade some other way, or just not bother since it's not too critical to the rest of this tutorial.

## Let's Go...

We'll start by setting up a new project:

~~~bash
mkdir flow-redux-tutorial
cd flow-redux-tutorial
npm init -f
~~~

and installing Redux and Ramda:

~~~bash
npm install --save redux ramda

~~~
Now you can create a `src/reducers/todo-items-reducer.js` file with the following contents:

~~~js
import {compose as lens, lensIndex as i, lensProp as p, remove, set} from 'ramda';

const ADD_TODO = 'ADD_TODO';
const DELETE_TODO = 'DELETE_TODO';
const EDIT_TODO = 'EDIT_TODO';
const TOGGLE_TODO = 'TOGGLE_TODO';
const COMPLETE_ALL = 'COMPLETE_ALL';
const CLEAR_COMPLETED = 'CLEAR_COMPLETED';

export const todoItemsReducer = (todoItems = [], action) => {
  switch(action.type) {
    case ADD_TODO: {
      const newTodoItem = {text: action.text, completed: false};
      return [...todoItems, newTodoItem];
    }

    case DELETE_TODO: {
      return remove(action.index, 1, todoItems);
    }

    case EDIT_TODO: {
      return set(lens(i(action.index), p('text')), action.text, todoItems);
    }

    case TOGGLE_TODO: {
      const completed = todoItems[action.index].completed;
      return set(lens(i(action.index), p('completed')), !completed, todoItems);
    }

    case COMPLETE_ALL: {
      return todoItems.map((todoItem) => ({...todoItem, completed: true}));
    }

    case CLEAR_COMPLETED: {
      return todoItems.filter((todoItem) => !todoItem.completed);
    }

    default: {
      return todoItems;
    }
  }
};
~~~

This reducer is using ES6/7 features like the array/object spread operators to ensure that the intent of the program will be understood by Flow when we introduce it.


## Testing our reducer

Let's install Mocha, Chai & Babel so we can test our reducer:

~~~bash
npm install --save-dev mocha chai babel babel-preset-modern babel-plugin-transform-object-rest-spread
~~~

Here we've installed [babel-preset-modern](https://www.npmjs.com/package/babel-preset-modern) instead of [babel-preset-es2015](https://www.npmjs.com/package/babel-preset-es2015), which will reduce what's transpiled to pretty much just the `import` and `export` statements &mdash; use babel-preset-es2015 instead if you didn't upgrade to a recent version of Node.js at the beginning of the tutorial.

You'll need to create a `.babelrc` file with the following contents before Babel can have any effect:

~~~json
{
  "presets": ["modern"],
  "plugins": [
    "transform-object-rest-spread"
  ]
}
~~~

Now create a `src/reducers/todo-items-reducer.spec.js` file with the following contents:

~~~js
import {todoItemsReducer} from './todo-items-reducer';
import {describe, it} from 'mocha';
import {expect} from 'chai';

describe('todo-items-reducer', () => {
  const initialTodoItems = [
    {text: 'Do stuff.', completed: true},
    {text: 'Do more stuff.', completed: false}
  ];

  it('allows items to be added', () => {
    const todoItems = todoItemsReducer(undefined, {type: 'ADD_TODO', text: 'Do stuff.'});
    expect(todoItems).to.deep.equal([{text: 'Do stuff.', completed: false}]);
  });

  it('allows items to be removed', () => {
    const todoItems = initialTodoItems;
    const updatedTodoItems = todoItemsReducer(todoItems, {type: 'DELETE_TODO', index: 0});
    expect(updatedTodoItems).to.deep.equal([
      {text: 'Do more stuff.', completed: false}
    ]);
  });

  it('allows items to be edited', () => {
    const todoItems = initialTodoItems;
    const editedTodoItems = todoItemsReducer(todoItems, {type: 'EDIT_TODO', index: 0, text: 'Do some stuff.'});
    expect(editedTodoItems).to.deep.equal([
      {text: 'Do some stuff.', completed: true},
      {text: 'Do more stuff.', completed: false}
    ]);
  });

  it('allows items to be marked as completed', () => {
    const todoItems = initialTodoItems;
    const modifiedTodoItems = todoItemsReducer(todoItems, {type: 'TOGGLE_TODO', index: 1});
    expect(modifiedTodoItems).to.deep.equal([
      {text: 'Do stuff.', completed: true},
      {text: 'Do more stuff.', completed: true}
    ]);
  });

  it('allows completed items to be marked as uncompleted', () => {
    const todoItems = initialTodoItems;
    const modifiedTodoItems = todoItemsReducer(todoItems, {type: 'TOGGLE_TODO', index: 0});
    expect(modifiedTodoItems).to.deep.equal([
      {text: 'Do stuff.', completed: false},
      {text: 'Do more stuff.', completed: false}
    ]);
  });

  it('allow all items to be completed at once', () => {
    const todoItems = initialTodoItems;
    const modifiedTodoItems = todoItemsReducer(todoItems, {type: 'COMPLETE_ALL'});
    expect(modifiedTodoItems).to.deep.equal([
      {text: 'Do stuff.', completed: true},
      {text: 'Do more stuff.', completed: true}
    ]);
  });

  it('allows completed items to be removed', () => {
    const todoItems = initialTodoItems;
    const modifiedTodoItems = todoItemsReducer(todoItems, {type: 'CLEAR_COMPLETED'});
    expect(modifiedTodoItems).to.deep.equal([
      {text: 'Do more stuff.', completed: false}
    ]);
  });
});
~~~

and replace the `test` script in `package.json` with this:

~~~
"test": "mocha --compilers js:babel-core/register src/**/*.spec.js"
~~~

If you now run `npm test` you will see that all of the tests successfully pass. The eagle eyed among you may have noticed that we didn't use _action-creator_ functions within the test. Action creator functions have always felt to me like necessary boiler-plate, the need for for which would disappear if our JSON structures could be automatically type checked; well now they will be!


## Adding a linting pre-test step

We'll need to install ESLint so we can lint our code:

~~~
npm install --save-dev eslint
~~~

and add an `.eslintrc` config file to enable the default set of linting rules and ES6/7 support, which will look like this:

~~~
{
  "extends": "eslint:recommended",
  "env": {
    "es6": true
  },
  "parserOptions": {
    "sourceType": "module",
    "ecmaFeatures": {
      "experimentalObjectRestSpread": true
    }
  }
}
~~~

We can now add a `pretest` script above the `test` script in `package.json`:

~~~
"pretest": "eslint src",
~~~

which will also cause linting tests to run when you run `npm test` again. So far so good...

Ideally, you should configure your editor so that it can display ESLint errors in-line since this will make development much easier &mdash; for example, I use [linter-eslint](https://atom.io/packages/linter-eslint) for [Atom](https://atom.io/).


## Time for some Flow annotations!

Next, I'm going to walk you through the process of annotating the code we've written for Flow, but you may want to wait until we reach the **Putting it all together...** section before updating any files.

Within `todo-items-reducer.js`, we previously described a reducer that took some state and an action, and returned some modified state. If we say that `state` is of type `TodoItems`, and that `action` is of type `TodoAction`, then we can create an annotated version of the `todoItemsReducer` function where the function signature changes to look like this:

~~~js
export const todoItemsReducer = (todoItems: TodoItems = [], action: TodoAction): TodoItems => {
~~~

leaving us only the task of defining `TodoItems` and `TodoAction`.


### The `TodoItems` type

`TodoItems` can be defined as an array of `TodoItem`, like so:

~~~js
export type TodoItems = Array<TodoItem>;
~~~

where `TodoItem` can be defined as an object having an `item` property of type `string` and a `completed` property of type `boolean`, for example:

~~~js
type TodoItem = {
  text: string,
  completed: boolean
};
~~~

The neat thing here is that the empty array literal `[]` qualifies as being of type `TodoItems`, and any correctly typed object (e.g.`{text: 'Do stuff.', completed: false}`) qualifies as being of type `TodoItem`. It's got nothing to do with which constructor was used to create an object, and even if you don't use _action-creator_ functions to create your actions they will still be of the right type.

Even code that builds a type up in stages will type check fine, like this for example:

~~~js
const obj = {text: 'Do stuff.'};
const todoItem: TodoItem = {...obj, completed: false};
~~~

Again, Flow comprehends the flow of our code ... nice!


### The `TodoAction` type

We saw previously that `action.type` had one of six possible values (e.g. `ADD_TODO` and `DELETE_TODO`), and that the reducer's switch statement took a different path depending on which value it had. This really means that there are six different types here (e.g. `AddTodoAction` and `DeleteTodoAction`), and so `TodoAction` can be defined as a union type, like so:

~~~js
type TodoAction = AddTodoAction | DeleteTodoAction | EditTodoAction |
  ToggleTodoAction | CompleteAllAction | ClearCompletedAction;
~~~

Subsequently, the types themselves can be defined like this:

~~~js
type AddTodoAction = {
  type: 'ADD_TODO',
  text: string
};
type DeleteTodoAction = {
  type: 'DELETE_TODO',
  index: number
};
type EditTodoAction = {
  type: 'EDIT_TODO',
  index: number,
  text: string
};
type ToggleTodoAction = {
  type: 'TOGGLE_TODO',
  index: number
};
type CompleteAllAction = {
  type: 'COMPLETE_ALL'
};
type ClearCompletedAction = {
  type: 'CLEAR_COMPLETED'
};
~~~

In all cases, instead of defining `type` as a `string`, it's defined as a `string` with a particular value.

This is important since the values of `type` are distinct within the union type `TodoAction`, so that for any `action` of type `TodoAction` where `action.type` is equal to `'ADD_TODO'`, we can unambiguously say that the type of that action is `AddTodoAction`, rather than `DeleteTodoAction`, or some other action.

This means that while this code will error:

~~~js
const f = (action: TodoAction) => {
  action.type; // okay
  action.text; // will error!!!
};
~~~

this code will type check fine because of the qualifying `if` guard:

~~~js
const f = (action: TodoAction) => {
  if(action.type === 'ADD_TODO') {
    action.text;
  }
};
~~~

That's pretty sweet!


### Putting it all together...

When you put it all together you should end up with a `todo-items-reducer.js` that looks like this:

~~~
/* @flow */
import {compose as lens, lensIndex as i, lensProp as p, remove, set} from 'ramda';

export export type TodoItems = Array<TodoItem>;
type TodoItem = {
  text: string,
  completed: boolean
};

type TodoAction = AddTodoAction | DeleteTodoAction | EditTodoAction |
  ToggleTodoAction | CompleteAllAction | ClearCompletedAction;
type AddTodoAction = {
  type: 'ADD_TODO',
  text: string
};
type DeleteTodoAction = {
  type: 'DELETE_TODO',
  index: number
};
type EditTodoAction = {
  type: 'EDIT_TODO',
  index: number,
  text: string
};
type ToggleTodoAction = {
  type: 'TOGGLE_TODO',
  index: number
};
type CompleteAllAction = {
  type: 'COMPLETE_ALL'
};
type ClearCompletedAction = {
  type: 'CLEAR_COMPLETED'
};

const ADD_TODO = 'ADD_TODO';
const DELETE_TODO = 'DELETE_TODO';
const EDIT_TODO = 'EDIT_TODO';
const TOGGLE_TODO = 'TOGGLE_TODO';
const COMPLETE_ALL = 'COMPLETE_ALL';
const CLEAR_COMPLETED = 'CLEAR_COMPLETED';

export const todoItemsReducer = (todoItems: TodoItems = [], action: TodoAction): TodoItems => {
  switch(action.type) {
    case ADD_TODO: {
      const newTodoItem = {text: action.text, completed: false};
      return [...todoItems, newTodoItem];
    }

    case DELETE_TODO: {
      return remove(action.index, 1, todoItems);
    }

    case EDIT_TODO: {
      return set(lens(i(action.index), p('text')), action.text, todoItems);
    }

    case TOGGLE_TODO: {
      const completed = todoItems[action.index].completed;
      return set(lens(i(action.index), p('completed')), !completed, todoItems);
    }

    case COMPLETE_ALL: {
      return todoItems.map((todoItem) => ({...todoItem, completed: true}));
    }

    case CLEAR_COMPLETED: {
      return todoItems.filter((todoItem) => !todoItem.completed);
    }

    default: {
      return todoItems;
    }
  }
};
~~~

Unfortunately, if you run `npm test` at this point you'll notice that it now fails on the linting step, and so we'll need to fix that next...


## Making ESLint & Babel support Flow

We can fix the linting errors we saw previously by installing the following packages:

~~~
npm install --save-dev babel-eslint eslint-plugin-babel eslint-plugin-flow-vars
~~~

and updating `.eslintrc` to have a `plugins` and `parser` section, so that it becomes:

~~~json
{
  "extends": "eslint:recommended",
  "env": {
    "es6": true
  },
  "plugins": [
    "flow-vars"
  ],
  "parser": "babel-eslint",
  "parserOptions": {
    "sourceType": "module",
  }
}
~~~

If you now run `npm run pretest` you'll see that the linting is working again, but if you run `npm test` you'll see that the tests themselves still fail.

We can fix this by installing the following Babel plug-in:

~~~
npm install --save-dev babel-plugin-transform-flow-strip-types
~~~

and adding `transform-flow-strip-types` to the `plugins` section of `.babelrc`, so you end up with:

~~~
"plugins": [
  "transform-object-rest-spread",
  "transform-flow-strip-types"
]
~~~

at which point `npm test` will now work again!


## Adding a typing pre-test step

Before we forget, let's start by adding a `/* @flow */` comment to the first line of `todo-items-reducer.spec.js`.

Now, it's finally time to install Flow:

~~~bash
npm install --save-dev flow-bin
~~~

**Warning**: The [flow-bin](https://www.npmjs.com/package/flow-bin) package doesn't currently work on Windows or 32bit Linux. Windows users can probably install this themselves, ensuring that it's on the path, given there are [now non-official Windows binaries](http://www.ocamlpro.com/pub/ocpwin/flow-builds/) being made available.

The Flow library should be initialized as follows, which will cause it to create a `.flowconfig` file for your project:

~~~bash
flow init
~~~

**Warning:** If you don't have `./node_modules/.bin` permanently added to your path then you'll need to run `./node_modules/.bin/flow init` instead.

Let's now replace the `pretest` script in `package.json` with this, so that we have separate `pretest:lint` and `pretest:typecheck` scripts:

~~~
"pretest": "npm run pretest:lint && npm run pretest:typecheck",
"pretest:lint": "eslint src",
"pretest:typecheck": "flow",
~~~

If you run `npm test` again you should see that type-checking happens now too, and that everything passes!

If you run `npm test` a second time you'll notice that it's quicker the second time around due to the fact that the `flow` command spawned a background daemon the first time around. You can confirm this for yourself by running:

~~~bash
ps -ef | grep flow-bin
~~~


## Getting type feedback in your editor

Just like it is with linting, not having typing feedback in your editor makes for a bad developer experience. I'll assume you're using Atom here since that's what I use, and that you've already installed [linter-eslint](https://atom.io/packages/linter-eslint) so you already have linting feedback within your editor.

Given this is the case, you'll next want to install the [linter-flow](https://atom.io/packages/linter-flow) plug-in. After installing you should get Flow errors displayed within the Atom &mdash; just try breaking stuff!

**Warning:** Depending on which OS you use and how you installed Node.js, you may now need to start Atom from the command-line so that Atom has access to your normal environment variables.


## Kicking the tyres

At this point you may want to have a play around, and see exactly what Flow is and isn't capable of.

**Warning:** Supporting union and intersection types in a way where the error messages you receive still make sense in all cases is non-trivial, and there are presently still a few edge cases you may run into. These are slowly being fixed as far as I can tell, and things will hopefully become stable before too much longer.


## Adding a simplistic view

Before we can show how Flow's typing can be used instead of React's `propType` feature, we'll need to install React so we can create a rudimentary view.

Begin by installing the library dependencies we'll need:

~~~bash
npm install --save react react-dom react-redux
~~~

followed by installing the development dependencies we'll need:

~~~bash
npm install --save-dev babel-preset-react eslint-plugin-react
~~~

Next, add `react` to the list of  presets in `.babelrc`, so it looks like this:

~~~
"presets": ["modern", "react"],
~~~

and similarly add `react` to the list of plug-ins in `.eslintrc` so it looks like this:

~~~
"plugins": [
  "flow-vars",
  "react"
],
~~~

Then, add the following block to the `parserOptions` section of `.eslintrc`:

~~~
"ecmaFeatures": {
  "jsx": true
}
~~~

and finally update the `extends` definition in `.eslintrc` to be an array so we can add a `plugin:react/recommended` entry, as follows:

~~~
"extends": ["eslint:recommended", "plugin:react/recommended"],
~~~

With the config in place, you can now add the following new files:

`src/Todo.jsx`:

~~~js
{% raw %}
import React, {PropTypes} from 'react';

export const Todo = ({text, completed, onClick}) => (
  <li
    onClick={onClick}
    style={{
      textDecoration: completed ? 'line-through' : 'none'
    }}
  >
    {text}
  </li>
);

Todo.propTypes = {
  text: PropTypes.string.isRequired,
  completed: PropTypes.bool.isRequired,
  onClick: PropTypes.func.isRequired
};

export default Todo;
{% endraw %}
~~~

`src/TodoList.jsx`:

~~~js
import React, {PropTypes} from 'react';
import {connect} from 'react-redux';
import Todo from './Todo';

export const TodoList = ({todos, onTodoClick}) => (
  <ul>
    {
      todos.map((todo, index) =>
      <Todo
        key={index}
        {...todo}
        onClick={() => onTodoClick(index)}
      />
    )}
  </ul>
);

TodoList.propTypes = {
  todos: PropTypes.arrayOf(PropTypes.shape({
    text: PropTypes.string.isRequired,
    completed: PropTypes.bool.isRequired
  }).isRequired).isRequired,
  onTodoClick: PropTypes.func.isRequired
};

export const TodoListHOC = connect(
  (state) => ({
    todos: state
  }),
  (dispatch) => ({
    onTodoClick: (index) => {
      dispatch({type: 'TOGGLE_TODO', index});
    }
  })
)(TodoList);

export default TodoListHOC;
~~~

and `src/app.js`:

~~~js
/* global document */
import React from 'react';
import {render} from 'react-dom';
import {Provider} from 'react-redux';
import {createStore} from 'redux';
import {todoItemsReducer} from './reducers/todo-items-reducer';
import TodoList from './TodoList';

const store = createStore(todoItemsReducer);
const appElem = document.createElement('div');
appElem.id = 'app';

store.dispatch({type: 'ADD_TODO', text: 'Do stuff.'});
store.dispatch({type: 'ADD_TODO', text: 'Do more stuff.'});
store.dispatch({type: 'TOGGLE_TODO', index: 0});

document.body.appendChild(appElem);

render(
  <Provider store={store}>
    <TodoList />
  </Provider>,
  appElem
);
~~~

If you try running `npm test` you'll notice you get a number of Flow errors due to the 'fbjs' utility library. Until these errors are fixed upstream, you can just have Flow ignore this library by adding the following line to the `ignore` section of `.flowconfig`:

~~~
.*node_modules/fbjs.*
~~~

## Viewing the running app

To view our running app we'll install Budo:

~~~bash
npm install --save-dev budo babelify
~~~

and then add the following `start` script to `package.json`:

~~~
"start": "budo src/app.js -- -t babelify --extension=jsx"
~~~

so that we can start the server like this:

~~~bash
npm start
~~~

## JSX Validation Without `propType`

Adding type annotations to `Todo.jsx` and `TodoList.jsx` first requires us to add a `/* @flow */` comment to both files, then to update the renderer in `Todo.jsx` to have this type signature:

~~~js
type TodoArgs = {text: string, completed: boolean, onClick: Function};
export const Todo = ({text, completed, onClick}: TodoArgs): Object => (
~~~

and the function signature in `TodoList.jsx` to have this type signature:

~~~js
import type {TodoItems} from './reducers/todo-items-reducer';

type TodoListArgs = {todos: TodoItems, onTodoClick: Function};
export const TodoList = ({todos, onTodoClick}: TodoListArgs): Object => (
~~~

At which point if you intentionally create an invalid JSX component like this in `Todo.jsx`:

~~~js
<Todo invalid-prop/>
~~~

then ... nothing!


## JSX Validation Without `propType` (Attempt Two)

Unfortunately, at present, Flow does not support stateless functional React components, and only supports `React.createClass` and `extends React.Component`. Let's update both `Todo.jsx` and `TodoList.jsx` to use `extends React.component` syntax.

Before we do this we'll need to add support for ES7 class properties by running this:

~~~bash
npm install babel-plugin-transform-class-properties
~~~

and updating the `plugins` section of `.babelrc` to this:

~~~
"plugins": [
  "transform-class-properties",
  "transform-object-rest-spread",
  "transform-flow-strip-types"
]
~~~

Once this is done you can update `Todo.jsx` to look like this:

~~~js
{% raw %}
/* @flow */
import React from 'react';

type TodoArgs = {text: string, completed: boolean, onClick: Function};
export default class Todo extends React.Component {
  props: TodoArgs;

  constructor(props: TodoArgs) {
    super(props);
  }

  render() {
    return (
      <li
        onClick={this.props.onClick}
        style={{
          textDecoration: this.props.completed ? 'line-through' : 'none'
        }}
      >
        {this.props.text}
      </li>
    );
  }
}
{% endraw %}
~~~

and `TodoList.jsx` to look like this:

~~~js
/* @flow */
import React from 'react';
import {connect} from 'react-redux';
import Todo from './Todo';
import type {TodoItems} from './reducers/todo-items-reducer';

type TodoListArgs = {todos: TodoItems, onTodoClick: Function};
export class TodoList extends React.Component {
  props: TodoListArgs;

  constructor(props: TodoListArgs) {
    super(props);
  }

  render() {
    return (
      <ul>
        {
          this.props.todos.map((todo, index) =>
          <Todo
            key={index}
            {...todo}
            onClick={() => this.props.onTodoClick(index)}
          />
        )}
      </ul>
    );
  }
}

export const TodoListHOC = connect(
  (state) => ({
    todos: state
  }),
  (dispatch) => ({
    onTodoClick: (index) => {
      dispatch({type: 'TOGGLE_TODO', index});
    }
  })
)(TodoList);

export default TodoListHOC;
~~~

Now, the same invalid JSX component within `Todo.jsx` will yield type errors as expected.


## Type Verification on Higher Order Components

I held back from publishing this article for over a week because of this [Redux issue comment](https://github.com/reactjs/redux/issues/290#issuecomment-212866184) which seems to indicate that it should be possible to do type verification on HOCs created with `connect()`, but ten days later and the Redux version is yet to magically appear! So, I'm publishing anyway, but with the hope that further progress in this area is made, and JSX validation starts to work for HOCs too.
