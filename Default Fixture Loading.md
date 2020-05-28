###### tags: `R4MP` `R4MP Draft`

# Default Fixture Loading

## Ramble

Our current working/testing pages all explicitly load their fixtures after the `initRAMP()` triggers.

A nice-to-have would be a way to run "out-of-the-box" RAMP without having to list out the specific fixtures. This would result in default configuration, where all the fixtures are the stock fixtures (our legend, our grid, our appbar, our details panel, etc).

Further, we have a concept of some core fixtures having "reserved" names. The fine details can be found in the [event api design](https://github.com/ramp4-pcar4/r4design/issues/14), the TLDR is it can be useful to replace an out-of-box fixture with a new fixture having the same name and function interfaces, so all calls to that fixture still line up.

There is also the idea of removing a fixture, which sounds simple but there may be some wrinkles.

## Default Fixture Setup

This covers the standard situation of just wanting the standard fixtures. Adding the fixtures themselves is simple; the benefits of having a default setup wrapper are more apparent after reading the event API sections further down.

### Request Defaults

We have an instance method that adds our "out-of-the-box" fixtures. This still puts the onus on the page author, but reduces the code to one simple method call. 

```js
function initRAMP() {
    var rInstance = new RAMP.Instance(...);
    rInstance.fixtures.addDefaults();
}
```

Alternately, this could be part of an options param for the RAMP instance.

```js
function initRAMP() {
    var rInstance = new RAMP.Instance(..., { defaultFixtures: true });
}
```

### Forced Exclusion

We always auto-add the default fixtures unless the page author excludes them.  Given fixtures execute code when they are added, this exclusion would need to be on the instance constructor to prevent unwanted default fixtures from running `added()` logic all the time.

```js
function initRAMP() {
    var rDefaultInstance = new RAMP.Instance(...);
    var rCustomInstance = new RAMP.Instance(..., { defaultFixtures: false });
    rCustomInstance.fixture.add(customFixture);
}
```

### Moar?

Other ideas can lurk here.

## Event API Implications

Revist the design fun [here](https://github.com/ramp4-pcar4/r4design/issues/14). Key takeaways from that design are:

- Want none or minimal event API code to be required on host page if deploying RAMP out-of-the-box.
- Need a clear and easy way to suppress or remove default event handlers when customizing.

### Bundle Party

This idea bundles the default event bindings with the setup of default fixtures. Basically, you get default fixtures and events, or you get neither.

For the `Request Defaults` proposal above, our default events would be created in the `fixtures.addDefault()` call. The `Forced Exclusion` proposal would have default events on instance constructor, and would skip them in the suppression case.

**Pros:**

- Satisfies our desire for no page code with simplicity.

**Cons:**

- Doing very minor modifications to the out-of-box config (e.g. use out of box but replace one fixture) might require more work than other approaches (i.e. needing to un-register fixtures and/or events after the defaults run)

### Two Step Shuffle

Similar to the above, but the adding of default events is an explicit call.

```js
function initRAMP() {
    var rInstance = new RAMP.Instance(...);
    rInstance.fixtures.addDefault();
    rInstance.fixtures.addDefaultEvents();
}
```

**Pros:**

- Adds a bit more granularity and control for the page code.

**Cons:**

- Another line required on the page.
- Uncertain if the benefits of a separate step amounts to much.

### Moaar?

Other ideas for event setup here

## Minor Modding

Consider the following reasonable scenarios

- Site author wants default configuration but wants one default fixture not be included. Prefer it never loads instead of a load-then-remove pattern.
- Site author wants default configuration but wants to replace a default fixture with a new fixture, using the same name to preserve event alignment.

Adding a subset of default fixtures is not too difficult, as having a batch of `fixture.add()` commands in the startup gets it done (we currently do this in our sample pages). The trickery comes in when considering the default events.

Also remember that events work on name strings, so it is ok to have an event defined before a fixture exists.

### Fixture Part - Old School Cool

For this idea we pretty much expect the page to manually add the fixtures and events they need. Not default configuration, so write your custom code, sir.

**Pros:**

- Maximum control for page author
- Minimal fuss for RAMP core

**Cons:**

- Minor mod can lead to bloated page code.

### Fixture Part - Customized Exclusions

This can be more helpful if someone just wants to replace or omit one or two default fixtures. Our defaulting process (whichever we choose from above) has an optional list of exclusions.

E.g. removing geosearch and replacing north arrow with a snowman.

```js
function initRAMP() {
    var rInstance = new RAMP.Instance(...);
    rInstance.fixtures.addDefaults({ defaultOmit: ['geosearch', 'northarrow'] });
    rInstance.fixtures.add('northarrow'); // custom code would have been loaded
}
```

It's possible I still don't fully understand the loading process and the north arrow replacement would have happened automatically due to the custom fixture script being run prior on the page.

**Pros:**

- Page author can do minor mods with just a bit of code.

**Cons:**

- Just looking at the sample code the parameter seems awkward.
- Minor fuss for RAMP core.

### Fixture Part - Nuns! Reverse!

This case we run the normal defaulting, then expect the page code to remove anything it no longer wants.

**Pros:**

- Minimal fuss for RAMP core

**Cons:**

- Inefficient. Fixtures will be starting up then get removed.
- Knowing what to remove can be complex for someone not at ninja-level RAMP skills. Exacerbated when considering the event side of things.

The amount of page code could be less (better!) or more (worse!) depending on what type of mod is happening.

### Event Part - Named Default Events

This is the most basic approach, but requires more (albiet simple) code on the page, and the page auther needs to read the docs to understand what they need to do (not as simple).

All events on the event API have names. Internally to RAMP there will be methods to create our out-of-box events, and they will have constant names. These names will be documented. A page author would then include the default fixtures they want, and then add the default events they require (just by name, not coding the default events).

We would need some type of method to facilitate the manual request of a default event creation, like:

```js
instance.addDefaultEvent('<default event name>');
// or 
fixture.addDefaultEvent('<default event name>');
```

The fixture based method is probably better, as it aligns with the `fixture.on()` and `fixture.off()` methods. My one hesitancy is if there is a default event that is sort of nebulous, but we can always just pick a fixture as "owner" and document it.

The code inside `addDefaultEvent()` would leverage the same event creation methods that would get invoked when just using out-of-the-box RAMP.

Sample code: a ramp that only has out-of-box legends and grids

```js
function initRAMP() {
    var rInstance = new RAMP.Instance(...);
    var legFix = rInstance.fixtures.add('legend');
    rInstance.fixtures.add('grid');
    legFix.addDefaultEvent('ramp-legend-opens-grid');
    legFix.addDefaultEvent('ramp-legend-remove-layer-closes-grid');
}
```

**Pros:**

- Keeps things fairly in line with our recommended event API defaulting.
- Additions to the RAMP core are minimal.
- Allows for author to set up things exactly as they please.
- Can allow default events to be leveraged for overwritten same-named fixtures.

**Cons:**

- Authors really need to know what they're doing. The severity of this will depend how many default events there end up being and how convoluted they are.
- More page code, though far better than having to define the event routines in the page.

### Event Part - Robot Overlord Enhanced Service

In this variation, we have one "add default events" function that takes a list of default fixtures, and behind the scenes figures out what default events are required for that permutation.  Under the hood we'll probably need a functionality matrix or a requirement mapping to know what default fixtures need to exists to trigger the adding of a default event.

Sample code: a ramp that only has out-of-box legends and grids

```js
function initRAMP() {
    var rInstance = new RAMP.Instance(...);
    rInstance.fixtures.add('legend');
    rInstance.fixtures.add('grid');
    rInstance.fixtures.addDefaultEvents(['legend', 'grid']);
}
```

**Pros:**

- Keeps things fairly in line with our recommended event API defaulting.
- Easy for an author to get a working subset of default fixtures.

**Cons:**

- Internal logic, while not complex, requires a lot of analysis and typing out the requirement matrix. Matrix also needs to be updates if default events change.
- Author can only get the perscribed events. They can't chose to omit one (though they can always add more code to remove a named event).
- Things could get squirrely if a default fixture is replaced with a same-named fixture, and author uses the default events for that fixture. Again this is authors fault, but might not be obvious since all the setup is hidden.

### Moooaaar?

Maybe other approaches?

## Old James' Choices

**Default Fixture Setup**: Fairly indifferent; It's all fairly simple with respect to page code. I like Request Defaults because you see that you have requested the default. But if default mode is very common it may feel like unnecessary page code.

**Default Event API**: Bundle Party. We can always shift to two-step later (i.e. sometime before v1 release) if we find we need it.

**Minor Modding - Fixture Part**: Leaning towards the Old School Cool. It works, it's not that hard. If we find page code is getting crazy we can entertain the other ideas, Customized Exclusions being my 2nd choice.

**Minor Modding - Event Part**: Named Default Events. Robot Overlord is cute but seems like it's extra effort and really if anyone is modding stuff they should know what they are doing. Less hand-holding, more quality documentaion, donethanks. I'd be most happy to hear a better idea on this one over the others.

But am open to grand discussions...

Overall I think with the named events and a few convenience functions / convenience constructor options we can get a lot of milage without needing to write too much "predict what the user dreams of" code inside RAMP.

