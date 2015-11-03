---
layout: post
title: "Testing React Components with Teaspoon & Unexpected"
modified:
categories: articles
excerpt:
tags: [react, testing, bdd, acceptance-testing]
image:
  feature:
date: 2015-10-25T13:18:00-00:00
---

This article shows you how you can use [teaspoon](https://www.npmjs.com/package/teaspoon) and [unexpected](http://unexpected.js.org/) to test your React components in a way that combines the advantages of unit testing with the advantages of acceptance testing. So we're clear on what that means, let's list some of the advantages of unit testing:

  * Tests are fast
  * Tests are reliable
  * Tests are easy to write

and some of the advantages of acceptance testing:

  * Tests verify the behaviour you're actually interested in.
  * Tests are easy to read and comprehend.
  * Tests support aggressive re-factoring since they verify behaviour rather than private implementation detail.

In this article we'll be extending the [two axes of testing](http://reactkungfu.com/2015/07/approaches-to-testing-react-components-an-overview/#two_axes_of_testing_components) defined by Martin Grzywaczewski, in his [Approaches to testing React components](http://reactkungfu.com/2015/07/approaches-to-testing-react-components-an-overview/) article, by adding a third type of testing. This gives us three types of test in total:

  * Structure Tests (_shallow rendering_)
  * Behaviour Tests (_shallow rendering_)
  * Binding Tests (_full rendering_)

To demonstrate this approach to testing, we'll use a fork of the [React TodoMVC App](https://github.com/dchambers/react-todomvc) to provide concrete test examples.

## Structure Tests

Generically, structure tests are written like this:

> **Given** _input-form_ **then** _output-form_.

Here's a concrete example:

~~~js
it('only renders a header when there are no items in the list', function() {
  // given
  let todoApp = $(<TodoApp model={model} router={router}/>);

  // then
  expect(todoApp.shallowRender().unwrap(), 'to have rendered with all children',
    <Container componentName="TodoApp">
      <TodoHeader/>
    </Container>
  );
});
~~~

## Behaviour Tests

Behaviour tests have a similar, but slightly longer generic form:

> **Given** _input-form_ **when** _behaviour-handler-invoked_ **then** _output-form_.

Here's another example:

~~~js
it('allows an item to be added to the list', function() {
  // given
  let todoApp = $(<TodoApp model={model} router={router}/>);

  // when
  todoApp.shallowRender().find('TodoHeader').unwrap().props.onTodoAdded('Item #1');

  // then
  expect(todoApp.shallowRender().unwrap(), 'to have rendered with all children',
    <Container componentName="TodoApp">
      <TodoHeader/>
      <TodoItems activeTodoCount={1}>
        <TodoItem title="Item #1" completed={false}/>
      </TodoItems>
      <TodoFooter count={1} completedCount={0} nowShowing="all"/>
    </Container>
  );
});
~~~

## Binding Tests

Binding tests allow us to verify whether a particular handler will be invoked when some user behaviour occurs. Generically, they have the form:

> **Given** _input-form_ **when** _user-behaviour_ **then** _behaviour-handler-invoked_.

Here's a final example:

~~~js
it('allows the user to add items', function() {
  // given
  let handleTodoAdded = sinon.spy();
  let todoHeader = $(<TodoHeader onTodoAdded={handleTodoAdded}/>);

  // when
  let inputBox = todoHeader.render().find('input.new-todo');
  inputBox.dom().value = 'Item #1';
  inputBox.trigger('keyDown', {key: 'Enter', keyCode: 13, which: 13});

  // then
  sinon.assert.calledWith(handleTodoAdded, 'Item #1');
});
~~~

## Making Your App Testable

You'll need to observe the following guidelines if you want to test your app in this way:

  1. Complex components should express themselves using semantically rich child components rather than DOM elements.
  2. The call-backs on the child components should be semantic too, and divorced from DOM concerns.
  3. The props on the child components should only accept simple types so we can use [unexpected](http://unexpected.js.org/) to test the output form.
  4. We should inject into the top-level component any objects that may require different implementations in test, or that are required due to the limitations of the test environment (e.g. the router in the [TodoMVC App](https://github.com/dchambers/react-todomvc)).
  5. Parent components should not pre-bind arguments to the call-back functions they provide to child components &mdash; any arguments needed within the parent component's call-back should be provided by the child component, causing these arguments to become testable within the binding tests.
  6. If you choose to write your components in ES6 you will need to ensure that any classes or stateless functions have the same name as the component, otherwise your [teaspoon](https://www.npmjs.com/package/teaspoon) queries won't work.

I would encourage you to skim read the entire set of [React TodoMVC App tests ](https://github.com/dchambers/react-todomvc/tree/master/test) to get a better feel for what this testing style looks like in practice.

## Performance

Tests written this way seem to execute very quickly. To give you an idea, here are the performance figures when running the complete set of tests on my machine:

~~~
TodoMVC App
  UI bindings
    ✓ allows the user to add items (32ms)
    ✓ does not allow the user to add emtpy items (4ms)
    ✓ allows the user to check active items (11ms)
    ✓ allows the user to destroy items (7ms)
    ✓ allows the user to mark all items as completed (9ms)
    ✓ allows the user to unmark all items as completed (5ms)
    ✓ allows the user to clear completed items (21ms)
    ✓ allows the user to view all items (30ms)
    ✓ allows the user to view active items (13ms)
    ✓ allows the user to view completed items (11ms)
  when the Todo list start off empty
    ✓ only renders a header when there are no items in the list (4ms)
    ✓ allows an item to be added to the list (4ms)
  when the Todo list starts off with a single active item
    ✓ starts off with a completed count of zero (1ms)
    ✓ updates the summary information when an items checkbox is ticked (2ms)
    ✓ removes the items list and footer when the last item is removed (1ms)
    ✓ updates the footer information when the completed filter is clicked (3ms)
    ✓ adds new items to the bottom of the list (4ms)
  when the Todo list contains multiple items
    ✓ marks all items as done when the toggle-all arrow is clicked (10ms)
    ✓ marks and then unmarks all items when the toggle-all arrow is clicked twice (3ms)
  when the Todo list contains a mixture of completed and active items
    ✓ shows all items by default (1ms)
    ✓ does not show active items when the completed view is used (1ms)
    ✓ does not show completed items when the active view is used (1ms)
    ✓ removes only completed items when clear-completed is clicked (1ms)

Todo Footer
  - does not display the clear all completed items button if there are no completed items
  ✓ displays the clear all completed items button if there are completed items (2ms)


24 passing (212ms)
1 pending
~~~

So, apart from the binding tests which are more variable, tests are super zippy.

## Conclusion

Tests written like this combine the benefits of unit and acceptance testing, while avoiding the potential disadvantages of acceptance testing, in that:

  1. Tests are no more costly to write than unit tests.
  2. Tests aren't prohibitively expensive to maintain due to unreliability &mdash; think [Selenium](http://www.seleniumhq.org/).
  3. Tests aren't obscured by an attempt to use natural language so that they can be understood by non-technical domain experts &mdash; think [Cucumber](https://cucumber.io/) or [FitNesse](http://www.fitnesse.org/).

Happy testing!
