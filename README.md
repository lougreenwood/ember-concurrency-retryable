# ember-concurrency-retryable

An Ember addon that adds retry strategies and a task modifier for automatically
retrying ember-concurrency tasks.

## Installation

```
ember install ember-concurrency-retryable
```

## Usage

### via wrapping of `task`
`ember-concurrency-retryable` adds a `retryable` function which can wrap an
`ember-concurrency` `TaskProperty`. You can use the addon via this wrapping
strategy like below:

```javascript
// components/flaky-component.js
import Component from '@ember/component';
import { task } from 'ember-concurrency';
import { retryable, DelayPolicy } from 'ember-concurrency-retryable';
import getWidgetsFromFlakyApi from '../utils/get-widgets';
import UnavailableError from '../utils/errors/unavailable';

const myDelayPolicy = new DelayPolicy({
  delay: [300, 600, 1200, 2400],
  reasons: [UnavailableError]
});

export default Component.extend({
  myTask: retryable(task(function*() {
    return yield getWidgetsFromFlakyApi();
  }), myDelayPolicy)
});
```

### via `task` property modifier
Personally, I think the wrapping method above looks kinda wack. Instead, you
can also add a `.retryable` modifier to ember-concurrency tasks through the
magic of reaching into private APIs! `ember-concurrency-retryable` provides a
`defineModifier` function that can be used somewhere early in the boot process,
such as `app.js` or in an initializer, that will add a `retryable` modifier
method to the `TaskProperty` prototype.

```javascript
// app.js
import defineModifier from 'ember-concurrency-retryable/define-modifier';

defineModifier();

// remainder of app.js...
```

```javascript
// components/flaky-component.js
import Component from '@ember/component';
import { task } from 'ember-concurrency';
import DelayPolicy from 'ember-concurrency-retryable/policies/delay';
import getWidgetsFromFlakyApi from '../utils/get-widgets';
import UnavailableError from '../utils/errors/unavailable';

const myDelayPolicy = new DelayPolicy({
  delay: [300, 600, 1200, 2400],
  reasons: [UnavailableError]
});

export default Component.extend({
  myTask: task(function*() {
    return yield getWidgetsFromFlakyApi();
  }).retryable(myDelayPolicy)
});
```

Since this relies on accessing `ember-concurrency`'s `TaskProperty` private
export and modifying the prototype for that object, I probably shouldn't
really *recommend* it.

## How does it work?

`retryable` works by wrapping a `task`'s generator function in a
try/catch and transparently retrying the generator function in accordance with
the passed in retry policy. `ember-concurrency-retryable` uses generator
functions and `ember-concurrency` primatives like `timeout` so that it plays
along nicely with `ember-concurrency` like any other task, and remains fully
cancelable.

## Adding retry policies

Retry policies are just plain old JavaScript classes/objects that correspond
to a particular interface. You can use those packaged with the addon, extend
from those, or build your own that fit your use case. A retry policy need only
correspond to the following interface:

```typescript
interface RetryPolicy {
  shouldRetry(retryState: TaskRetryState, reason: any): boolean;
  *retry(retryState: TaskRetryState): IterableIterator<any>;
}
```

## Motivation

As long as web apps need a network, network errors will exist. Sometimes
they're blips. In the case of mobile internet, intermittent network errors are
just part of the reality of distributed connectivity. There are many use cases
where user intervention should not be required for recovery from boring,
routine network blips, API failures, etc.

Some use-cases for automatic retrying of tasks:

* Long-polling tasks
    * Background auto-saves
* Telemetry/analytics
* Integrations w/ 3rd party APIs that don't quite have five 9s (or three)
* Applications that deal with periodic resource contention
    * e.g. app talks to a cloud storage API in files might be *locked* for a
      few seconds while some other system is working on it.
* Anything async that is prone to *transient failure*
    * e.g. Geolocation API failures while a mobile user is in transit

This work comes out of [an RFC](https://github.com/machty/ember-concurrency/issues/183) I opened against `ember-concurrency` a while ago.
After discussion on the RFC, it seemed like it would be best to implement this
functionality via an addon, to allow for experimentation and refinement.

## Contributing

* Working on something?
  * Open a PR with tests for what you're implementing or fixing
    * Open with `[WIP]` in the title if you want to start a conversation!

* Experiencing troubles?
  * Open an issue
  * Reach out to me on the [Ember.js Community Slack](https://embercommunity.slack.com/).
    You can find me as **@maxfierke** on there.
    (I caution that I do not scale)

### Installation

* `git clone git@github.com:maxfierke/ember-concurrency-retryable.git`
* `cd ember-concurrency-retryable`
* `yarn install`

### Linting

* `yarn lint:js`
* `yarn lint:js --fix`

### Running tests

* `ember test` – Runs the test suite on the current Ember version
* `ember test --server` – Runs the test suite in "watch mode"
* `yarn test` – Runs `ember try:each` to test your addon against multiple Ember versions

### Running the dummy application

* `ember serve`
* Visit the dummy application at [http://localhost:4200](http://localhost:4200).

For more information on using ember-cli, visit [https://ember-cli.com/](https://ember-cli.com/).

## License

This project is licensed under the [MIT License](LICENSE.md).

## Thanks

* [Ben Demboski](https://github.com/bendemboski) for [`ember-concurrency-test-waiter`](https://github.com/bendemboski/ember-concurrency-test-waiter) from which this addon approach is based.
* [Devin Weaver](https://github.com/sukima) for providing great feedback on my original ember-concurrency [RFC for retryable tasks](https://github.com/machty/ember-concurrency/issues/183).