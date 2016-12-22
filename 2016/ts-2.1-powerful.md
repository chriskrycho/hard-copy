---
Title: keyof and Mapped Types In TypeScript 2.1
Subtitle: Making JavaScript dance to an ML-ic tune.
Date: 2016-12-17 23:25
Modified: 2016-12-18 09:55
Tags: javascript, typescript, software development, programming languages
---

In the last few months, I've been playing with both [Flow] and [TypeScript] as tools for increasing the quality and reliability of the JavaScript I write at Olo. Both of these are syntax that sits on top of normal JavaScript to add type analysis—basically, a form of [gradual typing] for JS.

[Flow]: https://flowtype.org
[TypeScript]: http://www.typescriptlang.org
[gradual typing]: https://en.wikipedia.org/wiki/Gradual_typing

Although TypeScript's tooling has been better all along[^tooling] I initially preferred Flow's type system quite a bit: it has historically been much more focused on [soundness], especially around the *many* problems caused by `null` and `undefined`, than TypeScript. And it had earlier support for [tagged unions], a tool I've come to find invaluable since picking them up from my time with Rust.[^ml-descended] But the 2.0 and 2.1 releases of TypeScript have changed the game substantially, and it's now a *very* compelling language in its own right—not to mention a great tool for writing better JavaScript. So I thought I'd highlight how you can get a lot of the benefits you would get from the type systems of languages like Elm with some of those new TypeScript features: the *`keyof` operator* and *mapped types*.

[soundness]: http://stackoverflow.com/questions/21437015/soundness-and-completeness-of-systems
[tagged unions]: https://flowtype.org/docs/disjoint-unions.html

[^tooling]: it's no surprise that Microsoft's developer tooling is stronger than Facebook's

[^ml-descended]: along with all the other ML-descended languages I've played with, including Haskell, F^♯^, PureScript, and Elm.

---

<i>Some readers may note that what I'm doing here is a *lot* of wrangling to cajole TypeScript into giving me the kinds of things you get for free in an ML-descended language. Yep. The point is that you *can* wrangle it into doing this.</i>

---

### Plain old JavaScript

Let's say we want to write a little state machine in terms of a function to go from one state to the next, like this:

```javascript
function nextState(state) {
  switch(state) {
    case 'Pending': return 'Started';
    case 'Started': return 'Completed';
    case 'Completed': return 'Completed';
    default: throw new Error(`Bad state: ${state}`);
  }
}
```

This will work, and it'll even throw an error if you hand it the wrong thing. But you'll find out at runtime if you accidentally typed `nextState('Pednign')` instead of `nextState('Pending')`---something I've done more than once in the past. You'd have a similar problem if you'd accidentally written `case 'Strated'` instead of `case 'Started'`.

There are many contexts like this one in JavaScript---perhaps the most obvious being [Redux actions](http://redux.js.org/docs/basics/Actions.html), but I get a lot of mileage out of the pattern in Ember, as well. In these contexts, I find it’s convenient to define types that are kind of like pseudo-enums or pseudo-simple-unions, like so:[^freeze]

```javascript
const STATE = {
  Pending: 'Pending',
  Started: 'Started',
  Completed: 'Completed',
};
```

[^freeze]: Aside: to be extra safe and prevent any confusion or mucking around, you should probably call `Object.freeze()` on the object literal, too:

    ```javascript
    const STATE = Object.freeze({
      Pending: 'Pending',
      Started: 'Started',
      Completed: 'Completed',
    })
    ```
    
    Both convention and linters make it unlikely you'll modify something like this directly---but impossible is better than unlikely.

Once you've defined an object this way, instead of using strings directly in functions that take it as an argument, like `nextState('Started')`, you can use the object property: `nextState(STATE.Started)`. You can rewrite the function body to use the object definition instead as well:

```javascript
function nextState(state) {
  switch(state) {
    case STATE.Pending: return STATE.Started;
    case STATE.Started: return STATE.Completed;
    case STATE.Completed: return STATE.Completed;
    default: throw new Error(`Bad state: ${state}`);
  }
}
```

Using the object and its keys instead gets you something like a namespaced constant. As a result, you can get more help with things like code completion from your editor, along with warnings or errors from your linter if you make a typo. You'll also get *slightly* more meaningful error messages if you type the wrong thing. For example, if you type `STATE.Strated` instead of `STATE.Started`, any good editor will give you an error---especially if you're using a linter. At Olo, we use [ESLint], and we have it [set up][ember-cli-eslint] so that this kind of typo/linter error fails our test suite (and we never merge changes that don't pass our test suite!).

This is about as good a setup as you can get in plain-old JavaScript. As long as you're disciplined and always use the object, you get some real benefits from using this pattern. But you *always* have to be disciplined. If someone who is unfamiliar with this pattern types `nextState('whifflebats')` somewhere, well, we're back to blowing up at runtime. Hopefully your test suite catches that.

[ESLint]: http://eslint.org
[ember-cli-eslint]: https://github.com/ember-cli/ember-cli-eslint/


### TypeScript to the rescue

TypeScript gives us the ability to *guarantee* that the contract is met (that we're not passing the wrong value in). As of the latest release, it also lets us guarantee the `STATES` object to be set up the way we expect. And last but not least, we get some actual productivity boosts when writing the code, not just when debugging it.

Let's say we decided to constrain our `nextState` function so that it had to both take and return some kind of `State`, representing one of the states we defined above. We'll leave a `TODO` here indicating that we need to figure out how to write the type of `State`, but the function definition would look like this:

```typescript
// TODO: figure out how to define `State`
function nextState(state: State): State {
  // the same body...
}
```

TypeScript has had [union types] since the [1.4 release] so they might seem like an obvious choice, and indeed we could write easily a type definition for the strings in `STATES` as a union:

[union types]: https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types
[1.4 release]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-1-4.html

```typescript
type State = 'Pending' | 'Started' | 'Completed';
```

Unfortunately, you can't write something like `State.Pending` somewhere; you have to write the plain string `'Pending'` instead. You still get some of the linting benefits you got with the approach outlined above via TypeScript's actual type-checking, but you don't get *any* help with autocompletion. Can we get the benefits of both?

Yes! (This would be a weird blog post if I just got this far and said, "Nope, sucks to be us; go use Elm instead.")

As of the 2.1 release, TypeScript lets you define types in terms of keys, so you can write a type like this:[^flow]

[^flow]: Flow has supported this feature for some time; you can write `$Keys<typeof STATE>`---but the feature is entirely undocumented.

```typescript
const STATE = {
  Pending: 'Pending',
  Started: 'Started',
  Completed: 'Completed',
};

type StateFromKeys = keyof typeof STATE;
```

Then you can use that type any place you need to constrain the type of a variable, or a return, or whatever:

```typescript
const goodState: StateFromKeys = STATE.Pending;

// error: type '"Blah"' is not assignable to type 'State'
const badState: StateFromKeys = 'Blah';

interface StateMachine {
  (state: StateFromKeys): StateFromKeys;
}

const nextState: StateMachine = (state) => {
  // ...
}
```

The upside to this is that now you can guarantee that anywhere you're supposed to be passing one of those strings, you *are* passing one of those strings. If you pass in `'Compelte'`, you'll get an actual error---just like if we had used the union definition above. At a minimum, that will be helpful feedback in your editor. Maximally, depending on how you have your project configured, it may not even generate any JavaScript output.[^no-emit] So that's a significant step forward beyond what we had even with the best linting rules in pure JavaScript.

[^no-emit]: Set your `"compilerOptions"` key in your `tsconfig.json` to include `"noEmitOnError": true,`.


### Going in circles

But wait, we can do more! TypeScript 2.1 *also* came with a neat ability to define “mapped types,” which map one object type to another. They have a few [interesting examples][mapped types] which are worth reading. What's interesting to us here is that you can write a type like this:

[mapped types]: http://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html#mapped-types

```typescript
type StateAsMap = {
  [K in keyof typeof STATE]: K
}
```

And of course, you can simplify that using the type we defined above, since `StateFromKeys` was just `keyof typeof STATE`:

```typescript
type StateAsMap = {
  [K in StateFromKeys]: K
}
```

We've now defined an object type whose the *key* has to be one of the items in the `State` type.

Now, by itself, this isn't all that useful. Above, we defined that as the keys on the `STATE` object, but if we tried to use that in conjunction with this new type definition, we'd just end up with a recursive type definition: `StateFromKeys` defined as the keys of `STATE`, `StateAsMap` defined in terms of the elements of `StateFromKeys`, and then `STATE` defined as a `StateAsMap`...

```typescript
const STATE: StateAsMap = {
  Pending: 'Pending',
  Active: 'Active',
  Completed: 'Completed',
}

type StateFromKeys = keyof typeof STATE;

type StateAsMap = {
  [K in StateFromKeys]: K
}
```

You end up with multiple compiler errors here, because of the circular references. This approach won't work. If we take a step back, though, we can work through this (and actually end up someplace better).


### Join forces!

First, let's start by defining the mapping generically. After all, the idea here was to be able to use this concept all over the place---e.g. for *any* Redux action, not just one specific one. We don't need this particular `State`; we just need a constrained set of strings (or numbers) to be used as the key of an object:

```typescript
type MapKeyAsValue<Key extends string> = {
  [K in Key]: K
};
```

In principle, if we didn't have to worry about the circular references, we could use that to constrain our definition of the original `STATE` itself:

```typescript
const STATE: MapKeyAsValue<State> = {
  Pending: 'Pending',
  Started: 'Started',
  Completed: 'Completed',
};
```

So how to get around the problem of circular type definitions? Well, it turns out that the `K` values in these `StateObjectKeyToValue` and `StateUnionKeyToValue` types are equivalent:

```typescript
// Approach 1, using an object
const STATE = {
  Pending: 'Pending',
  Started: 'Started',
  Completed: 'Completed',
};

type StateFromKeys = keyof typeof STATE;
type StateObjectKeyToValue = {
  [K in StateFromKeys]: K  // <- K is just the keys!
};

// Approach 2, using unions
type StateUnion = 'Pending' | 'Started' | 'Completed';
type StateUnionKeyToValue = {
  [K in StateUnion]: K  // <- K is also just the keys!
};
```

Notice that, unlike the `StateObjectKeyToValue` version, `StateUnionKeyToValue` doesn't make any reference to the `STATE` object. So we can use `StateUnionKeyToValue` to constrain `STATE`, and then just use `StateUnion` to constrain all the places we want to *use* one of those states. Once we put it all together, that would look like this:

```typescript
type StateUnion = 'Pending' | 'Started' | 'Completed';

type StateUnionKeyToValue = {
  [K in StateUnion]: K
};

const STATE: StateUnionKeyToValue = {
  Pending: 'Pending',
  Started: 'Started',
  Completed: 'Completed',
};
```

By doing this, we get two benefits. First, `STATE` now has to supply the key and value for *all* the union's variants. Second, we know that the key and value are the same, and that they map to the union's variants. These two facts mean that we can be 100% sure that wherever we define something as requiring a `State`, we can supply one of the items on `STATE` and it will be guaranteed to be correct. If we change the `State` union definition, everything else will need to be updated, too.

Now we can make this generic, so it works for types besides just this one set of states---so that it'll work for *any* union type with string keys, in fact. (That string-key constraint is important because objects in TypeScript can currently only use strings or numbers as keys; whereas union types can be all sorts of things.) Apart from that constraint on the union, though, we can basically just substitute a generic type parameter `U`, for "union," where we had `StateUnion` before.

```typescript
type UnionKeyToValue<U extends string> = {
  [K in U]: K
};
```

Then any object we say conforms to this type will take a union as its type parameter, and every key on the object must have exactly the same value as the key name:

```typescript
type State = 'Pending' | 'Started' | 'Completed';

// Use `State` as the type parameter to `UnionKeyToValue`.
const STATE: UnionKeyToValue<State> = {
  Pending: 'Pending',
  Started: 'Started',
  Completed: 'Completed',
}
```

If any of those don't have *exactly* the same value as the key name, you'll get an error. So, each of the following value assignments would fail to compile, albeit for different reasons (top to bottom: capitalization, misspelling, and missing a letter).

```typescript
const BAD_STATE: UnionKeyToValue<State> = {
  Pending: 'pending',  // look ma, no capitals
  Started: 'Strated',  // St-rated = whuh?
  Completed: 'Complete',  // so tense
};
```

You'll see a compiler error that looks something like this:

> | [ts]  
> | Type '{ Pending: "pending"; Started: "Strated"; Completed: "Complete" }' is not assignable to type 'UnionKeyToValue<State>'.  
> |   Types of property 'Pending' are incompatible.  
> |     Type '"pending"' is not assignable to type '"Pending"'.

Since the key and the name don't match, the compiler tells us we didn't keep the constraint we defined on what these types should look like. Similarly, if you forget an item from the union, you'll get an error. If you add an item that isn't in the original union, you'll get an error. Among other things, this means that you can be confident that if you add a value to the union, the rest of your code won't compile until you include cases for it. You get all the power and utility of using union types, *and* you get the utility of being able to use the object as a namespace of sorts.[^namespace]

[^namespace]: For namespacing in a more general sense, you should use... [namespaces].

[namespaces]: http://www.typescriptlang.org/docs/handbook/namespaces.html

And the TypeScript language service---which you can use from a *lot* of editors, including VS Code, Atom, Sublime Text, and the JetBrains IDEs---will actually give you the correct completion when you start definition a type. So imagine we were defining some other union type elsewhere in our program to handle events. Now we can use the same `UnionKeyToValue` type to construct this type, with immediate, *correct* feedback from the TypeScript language service:

![TypeScript live code completion of the mapped type](http://cdn.chriskrycho.com/images/completion.gif "video of constrained completion")

By inverting our original approach of using `keyof` (itself powerful and worth using in quite a few circumstances) and instead using the new mapped types, we get a *ton* of mileage in terms of productivity when using these types---errors prevented, and speed of writing the code in the first place increased as well.

Yes, it's a little verbose and it does require duplicating the strings whenever you define one of these types.[^duplication] But, and this is what I find most important: there is only one *source* for those string keys, the union type, and it is definitive. If you change that central union type, everything else that references it, including the namespace-like object, will fail to compile until you make the same change there.

[^duplication]: It would be great if we could get these benefits without the duplication---maybe someday we'll have better support in JS or TS natively.

![Updating a union](http://cdn.chriskrycho.com/images/change-union.gif "video of union update")

So it's a lot more work than it would be in, say, Elm. But it's also a lot more guarantees than I'd get in plain-old-JavaScript, or even TypeScript two months ago.

I'll call that a win.