# MView - Materialized Views Library

MView is a distributed datatypes library. It consumes data-feeds and produces [materialized views](https://en.wikipedia.org/wiki/Materialized_view) as javascript objects. State is kept in memory, but may be persisted through the dump/load api.

## API Overview

```js
var mview = require('mview')
```

API Spec:

```js
mview.register([opts])
mview.text([opts])
mview.set([opts])
mview.list([opts])
```

Where:

```js
opts = {
  noTombstones: bool // default false, turns off tombstone tracking
}
```

[Tombstone tracking](https://en.wikipedia.org/wiki/Tombstone_%28data_store%29) is a technique to ensure consistency. It adds memory overhead, but allows messages to arrive out of order. Applications which implement their own causal ordering may disable tombstones.

### Using MView

MView is designed for distributed programs with reliable message broadcast. It was specifically built for [secure scuttlebutt](https://github.com/dominictarr/secure-scuttlebutt). It supports 4 different view types: registers, text buffers, sets, and ordered lists. The views are instantiated by the node's software at load, then used as follows:

 1. To generate view updates. These updates are broadcast on the network.
 2. To consume updates. As update-messages are received from the network, they are fed to the views.
 3. To produce the current state. When the node software needs to use the views, it asks them for their values.

In the examples below, there will be two functions which represent the networks's interface: `net.broadcast()` and `net.on('msg')`.

 - Assume that `broadcast()` takes a message object and sends it to all nodes (including the local node). 
 - Assume that `on('msg')` is called each time there is a new message from the local node or a remote node.

### Registers

Register views are opaque values. They are designed to converge on the most recent update value, and will not merge the values given.

```js
var reg = mview.register()
```

API Spec:

```js
reg.set(previousTags, tag, value) // updates the value
reg.tags()                        // gives the current tags
reg.toObject()                    // gives the current value
```

Registers use unique identifiers known as tags to track the partial order of updates. To generate an update, you capture the currently-set tags with `reg.tags()` - these are the values used in `previousTags`. The update also requires a new unique tag (the `tag` parameter) which the application must generate.

Example Usage:

```js
var cuid = require('cuid')
function setRegister(value) {
  net.broadcast({
    previousTags: reg.tags()
    tag: cuid(),
    value: value
  })
}
net.on('msg', function (msg) {
  reg.set(msg.previousTags, msg.tag, msg.value)
})
```

### Text

Text views are strings. They are designed to merge updates so that users can make concurrent updates to its contents.

```js
var text = mview.text()
```

API Spec:

```js
text.update(diff) // updates the string with the given diff
text.diff(str)    // generates a diff between the current string and the given string
text.toString()   // gives the current value
```

Text views use diffs structures to apply updates. These diffs are generated by passing the new value to `diff()`.

Example Usage:

```js
function setText(value) {
  net.broadcast({
    diff: text.diff(value)
  })
}
net.on('msg', function (msg) {
  text.update(msg.diff)
})
```

### Set

Set views are unordered sets of unique values.

```js
var set = mview.set()
```

API Spec:

```js
set.add(tag, value)         // adds the value to the set
set.remove(tag|tags, value) // removes the value from the set

set.tags(value)             // gets the tags for a given value

set.toObject()              // gives the current value
set.has(value)              // returns bool
set.count()                 // returns the number of values in the set
set.forEach(function(tags, value, index))
set.map(function(tags, value, index))
```

Like the register view, the set uses tags in order to track the order of updates. These tags are generated by the application.

Example Usage:

```js
var cuid = require('cuid')
function addToSet(v) {
  net.broadcast({
    type: 'add',
    tag: cuid(),
    value: v
  })
}
function removeFromSet(v) {
  net.broadcast({
    type: 'remove',
    value: v,
    tags: set.tags(v)
  })
}
net.on('msg', function (msg) {
  if (msg.type == 'add')
    set.add(msg.tag, msg.value)
  if (msg.type == 'remove')
    set.remove(msg.tags, msg.value)
})
```

### List

List views are ordered arrays of values.

```js
var list = mview.list()
```

API Spec:

```js
list.insert(tag, value)         // inserts the value into the list
list.remove(tag)                // removes the value represent by tag from the list

list.tags(index)                // gets the tags at the given position in the list
list.between(tagA, tagB, [uid]) // generates a tag which sorts greater than tagA and less than tagB

list.toObject()                 // gets the current list value
list.get(index|tag)             // gets the value at the given tag or index
list.count()                    // gets the number of values in the list
list.forEach(function(tag, value, index))
list.map(function(tag, value, index))
```

The list view uses tags like the set, but its tags are ordered (using the Logoot algorithm). This means the application uses `between()` to generate its tags.

`between()` takes two tags and generates a new tag that sorts in between them. It uses a weak random function to avoid tag-collisions between ndoes. For a strong guarantee, pass a unique id as the third param of `between()`. Doing so will increase the key length, but may be necessary in some applications.

Example Usage:

```js
var cuid = require('cuid')
function append(v) {
  net.broadcast({
    type: 'insert',
    tag: list.between(list.tags(list.count() - 1), null, cuid())
    value: v
  })
}
function prepend(v) {
  net.broadcast({
    type: 'insert',
    tag: list.between(null, list.tags(0), cuid())
    value: v
  })
}
function insert(i, v) {
  net.broadcast({
    type: 'insert',
    tag: list.between(list.tags(i-1), list.tags(i), cuid())
    value: v
  })
}
function remove(i) {
  net.broadcast({
    type: 'remove',
    tag: set.tags(i)
  })
}
net.on('msg', function (msgid, msg) {
  if (msg.type == 'insert')
    list.insert(msg.tag, msg.value)
  if (msg.type == 'remove')
    list.remove(msg.tag)
})
```
