# Autodux

Automate the Redux boilerplate.

## Status: Early Release

This library is ready for production testing. Please check it out and file any issues you encounter.

## Install

```
npm install --save autodux
```

And then in your file:

```js
import autodux from 'autodux';
```

Or using CommonJS syntax:

```js
const autodux = require('autodux');
```

## Why?

Redux is great, but you have to make a lot of boilerplate:

* Action constants
* Action creators
* Reducers
* Selectors

It's **great** that Redux is such a low-level tool. It's allowed a lot of flexibility and enabled the community to experiment with best practices and patterns.

It's **terrible** that Redux is such a low-level tool. It turns out that:

* Most reducers spend most of their logic switching over action types.
* Most of the actual state updates could be replaced with generic tools like "concat payload to an array", "remove object from array by some prop", or "increment a number". Better to reuse utilities
than to implement these things from scratch every time.
* Lots of action creators don't need arguments or payloads -- just action types.
* Action types can be automatically generated by combining the name of the state slice with the name of the action, e.g., `counter/increment`.
* Lots of selectors just need to grab the state slice.

Lots of Redux beginners separate all these things into separate files, meaning you have to open and import a whole bunch of files just to get some simple state management in your app.

What if you could write some simple, declarative code that would automatically create your:

* Action type constants
* Reducer switching logic
* State slice selectors
* Action object shape - automatically inserting the correct action type so all you have to worry about is the payload
* Action creators if the input is the payload or no payload is required, i.e., `x => ({ type: 'foo', payload: x })`
* Action reducers if the state can be assigned from the action payload (i.e., `{...state, ...payload}`)

Turns out, when you add this simple logic on top of Redux, you can do a lot more with a lot less code.

```js
import autodux, { id } from 'autodux';

export const {
  reducer,
  initial,
  slice,
  actions: {
    increment,
    decrement,
    multiply
  },
  selectors: {
    getValue
  }
} = autodux({
  // the slice of state your reducer controls
  slice: 'counter',

  // The initial value of your reducer state
  initial: 0,

  // No need to implement switching logic -- it's
  // done for you.
  actions: {
    increment: state => state + 1,
    decrement: state => state - 1,
    multiply: (state, payload) => state * payload
  },

  // No need to select the state slice -- it's done for you.
  selectors: {
    getValue: id
  }
});
```

As you can see, you can destructure and export the return value directly where you call `autodux()` to reduce boilerplate to a minimum. It returns an object that looks like this:

```js
{
  initial: 0,
  actions: {
    increment: { [Function]
      type: 'counter/increment'
    },
    decrement: { [Function]
      type: 'counter/decrement'
    },
    multiply: { [Function]
      type: 'counter/multiply'
    }
  },
  selectors: {
    getValue: [Function: wrapper]
  },
  reducer: [Function: reducer],
  slice: 'counter'
}
```

Let's explore that object a bit:

```js
const actions = [
  increment(),
  increment(),
  increment(),
  decrement()
];

const state = actions.reduce(reducer, initial);

console.log(getValue({ counter: state })); // 2
console.log(increment.type); // 'counter/increment'
```


## API Differences

Action creators, reducers and selectors have simplified APIs.

### Action Creators

Action creators are optional! If you need to set a username, you might normally create an action creator like this:

```js
const setUserName = userName => ({
  type: 'userReducer/setUserName',
  payload: userName
})
```

With autoDux, if your action creator maps directly from input to payload, you can omit it. autoDux will do it for you.

By omitting the action creator, you can shorten this:

```js
actions: {
  multiply: {
    create: by => by,
    reducer: (state, payload) => state * payload
  }
}
```

To this:

```js
actions: {
  multiply: (state, payload) => state * payload
}
```

#### No need to set the type

You don't need to worry about setting the type in autoDux action creators. That's handled for you automatically. In other words, all an action creator has to do is return the payload.

With Redux alone you might write:

```js
const setUserName = userName => ({
  type: 'userReducer/setUserName',
  payload: userName
})
```

With autoDux, that becomes:

```js
userName => userName
```

Since that's the default behavior, you can omit that one entirely.

You don't need to create action creators unless you need to map the inputs to a different payload output. For example, if you need to translate between an auth provider user and your own application user objects, you could use an action creator like this:

```js
({ userId, displayName }) => ({ uid, userName })
```

Here's how you'd implement our multiply action if you want to use a named parameter for the multiplication factor:

```js
//...
actions: {
  multiply: {
    // Destructure the named parameter, and return it
    // as the action payload:
    create: ({ by }) => by,
    reducer: (state, payload) => state * payload
  }
}
//...
```

### Reducers

> Note: Reducers are optional. If your reducer would just assign the payload props to the state (`{...state, ...payload}`), you're already done.

#### No switch required

Switching over different action types is automatic, so we don't need an action object that isolates the action type and payload. Instead, we pass the action payload directly, e.g:

With Redux:

```js
const INCREMENT = 'INCREMENT';
const DECREMENT = 'DECREMENT';
const MULTIPLY = 'MULTIPLY';

const counter = (state = 0, action = {}) {
  switch (action.type){
    case INCREMENT: return state + 1;
    case DECREMENT: return state - 1;
    case MULTIPLY : return state * action.payload
    default: return state;
  }
};
```

With Autodux, action type handlers are switched over automatically. No more switching logic!

```js
const counter = autodux({
  slice: 'counter',
  initial: 0,
  actions: {
    increment: state => state + 1,
    decrement: state => state - 1,
    multiply: (state, payload) => state * payload
  }
});
```

Autodux infers action types for you automatically using the slice and the action name, and eliminates the need to write switching logic or worry about (or forget) the default case.

Because the switching is handled automatically, your reducers don't need to worry about the action type. Instead, they're passed the payload directly.


### Selectors

Selectors are designed to take the application's complete root state object, but the slice you care about is automatically selected for you, so you can write your selectors as if you're only dealing with the local reducer.

This has some implications with unit tests. The following selector will just return the local reducer state:

```
import { autodux, id } from 'autodux';

const counter = autodux({
  // stuff here
  selectors: {
    getValue: id // state => state
  },
  //other stuff
```

In your unit tests, you'll need to pass the key for the state slice to mock the global store state:

```js
test('counter.getValue', assert => {
  const msg = 'should return the current count';
  const { getValue } = counter.selectors;

  const actual = getValue({ counter: 3 });
  const expected = 3;

  assert.same(actual, expected, msg);
  assert.end();
});
```

## Extras

### assign = (key: String) => reducer: Function

Often, we want our reducers to simply set a key in the state to the payload value. `assign()` makes that easy. e.g.:

```js
const {
  actions: {
    setUserName,
    setAvatar
  },
  reducer
} = autodux({
  slice: 'user',
  initial: {
    userName: 'Anonymous',
    avatar: 'anonymous.png'
  },
  actions: {
    setUserName: assign('userName'),
    setAvatar: assign('avatar')
  }
});

const userName = 'Foo';
const avatar = 'foo.png';

const state = [
  setUserName(userName),
  setAvatar(avatar)
].reduce(reducer, undefined);
// => { userName: 'Foo', avatar: 'foo.png' }
```

### id = x => x

Useful for selectors that simply return the slice state:

```js
selectors: {
  getValue: id
}
```
