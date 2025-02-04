---
title: Publish and subscribe
description: Documentation of Meteor's publication and subscription API.
---

These functions control how Meteor servers publish sets of records and
how clients can subscribe to those sets.

If you prefer video watch [Meteor Publications 101](https://www.youtube.com/watch?v=RH2RxKgkPJY) in our [YouTube Channel](https://www.youtube.com/channel/UC3fBiJrFFMhKlsWM46AsAYw).

{% apibox "Meteor.publish" %}

To publish records to clients, call `Meteor.publish` on the server with
two parameters: the name of the record set, and a *publish function*
that Meteor will call each time a client subscribes to the name.

Publish functions can return a
[`Collection.Cursor`](#mongo_cursor), in which case Meteor
will publish that cursor's documents to each subscribed client. You can
also return an array of `Collection.Cursor`s, in which case Meteor will
publish all of the cursors.

{% pullquote 'warning' %}
If you return multiple cursors in an array, they currently must all be from
different collections. We hope to lift this restriction in a future release.
{% endpullquote %}

A client will see a document if the document is currently in the published
record set of any of its subscriptions. If multiple publications publish a
document with the same `_id` to the same collection the documents will be
merged for the client. If the values of any of the top level fields
conflict, the resulting value will be one of the published values, chosen
arbitrarily.

```js
// Server: Publish the `Rooms` collection, minus secret info...
Meteor.publish('rooms', function () {
  return Rooms.find({}, {
    fields: { secretInfo: 0 }
  });
});

// ...and publish secret info for rooms where the logged-in user is an admin. If
// the client subscribes to both publications, the records are merged together
// into the same documents in the `Rooms` collection. Note that currently object
// values are not recursively merged, so the fields that differ must be top
// level fields.
Meteor.publish('adminSecretInfo', function () {
  return Rooms.find({ admin: this.userId }, {
    fields: { secretInfo: 1 }
  });
});

// Publish dependent documents and simulate joins.
Meteor.publish('roomAndMessages', function (roomId) {
  check(roomId, String);

  return [
    Rooms.find({ _id: roomId }, {
      fields: { secretInfo: 0 }
    }),
    Messages.find({ roomId })
  ];
});
```

Alternatively, a publish function can directly control its published record set
by calling the functions [`added`](#publish_added) (to add a new document to the
published record set), [`changed`](#publish_changed) (to change or clear some
fields on a document already in the published record set), and
[`removed`](#publish_removed) (to remove documents from the published record
set).  These methods are provided by `this` in your publish function.

If a publish function does not return a cursor or array of cursors, it is
assumed to be using the low-level `added`/`changed`/`removed` interface, and it
**must also call [`ready`](#publish_ready) once the initial record set is
complete**.

Example (server):

```js
// Publish the current size of a collection.
Meteor.publish('countsByRoom', function (roomId) {
  check(roomId, String);

  let count = 0;
  let initializing = true;

  // `observeChanges` only returns after the initial `added` callbacks have run.
  // Until then, we don't want to send a lot of `changed` messages—hence
  // tracking the `initializing` state.
  const handle = Messages.find({ roomId }).observeChanges({
    added: (id) => {
      count += 1;

      if (!initializing) {
        this.changed('counts', roomId, { count });
      }
    },

    removed: (id) => {
      count -= 1;
      this.changed('counts', roomId, { count });
    }

    // We don't care about `changed` events.
  });

  // Instead, we'll send one `added` message right after `observeChanges` has
  // returned, and mark the subscription as ready.
  initializing = false;
  this.added('counts', roomId, { count });
  this.ready();

  // Stop observing the cursor when the client unsubscribes. Stopping a
  // subscription automatically takes care of sending the client any `removed`
  // messages.
  this.onStop(() => handle.stop());
});

// Sometimes publish a query, sometimes publish nothing.
Meteor.publish('secretData', function () {
  if (this.userId === 'superuser') {
    return SecretData.find();
  } else {
    // Declare that no data is being published. If you leave this line out,
    // Meteor will never consider the subscription ready because it thinks
    // you're using the `added/changed/removed` interface where you have to
    // explicitly call `this.ready`.
    return [];
  }
});
```

Example (client):

```js
// Declare a collection to hold the count object.
const Counts = new Mongo.Collection('counts');

// Subscribe to the count for the current room.
Tracker.autorun(() => {
  Meteor.subscribe('countsByRoom', Session.get('roomId'));
});

// Use the new collection.
const roomCount = Counts.findOne(Session.get('roomId')).count;
console.log(`Current room has ${roomCount} messages.`);
```

{% pullquote 'warning' %}
Meteor will emit a warning message if you call `Meteor.publish` in a
project that includes the `autopublish` package.  Your publish function
will still work.
{% endpullquote %}

Read more about publications and how to use them in the
[Data Loading](http://guide.meteor.com/data-loading.html) article in the Meteor Guide.

{% apibox "Subscription#userId" %}

This is constant. However, if the logged-in user changes, the publish
function is rerun with the new value, assuming it didn't throw an error at the previous run.

{% apibox "Subscription#added" %}
{% apibox "Subscription#changed" %}
{% apibox "Subscription#removed" %}
{% apibox "Subscription#ready" %}
{% apibox "Subscription#onStop" %}

If you call [`observe`](#observe) or [`observeChanges`](#observe_changes) in your
publish handler, this is the place to stop the observes.

{% apibox "Subscription#error" %}
{% apibox "Subscription#stop" %}
{% apibox "Subscription#connection" %}

{% apibox "Meteor.subscribe" %}

When you subscribe to a record set, it tells the server to send records to the
client.  The client stores these records in local [Minimongo
collections](#mongo_collection), with the same name as the `collection`
argument used in the publish handler's [`added`](#publish_added),
[`changed`](#publish_changed), and [`removed`](#publish_removed)
callbacks.  Meteor will queue incoming records until you declare the
[`Mongo.Collection`](#mongo_collection) on the client with the matching
collection name.

```js
// It's okay to subscribe (and possibly receive data) before declaring the
// client collection that will hold it. Assume 'allPlayers' publishes data from
// the server's 'players' collection.
Meteor.subscribe('allPlayers');
...

// The client queues incoming 'players' records until the collection is created:
const Players = new Mongo.Collection('players');
```

The client will see a document if the document is currently in the published
record set of any of its subscriptions. If multiple publications publish a
document with the same `_id` for the same collection the documents are merged for
the client. If the values of any of the top level fields conflict, the resulting
value will be one of the published values, chosen arbitrarily.

{% pullquote 'warning' %}
Currently, when multiple subscriptions publish the same document *only the top
level fields* are compared during the merge. This means that if the documents
include different sub-fields of the same top level field, not all of them will
be available on the client. We hope to lift this restriction in a future release.
{% endpullquote %}

The `onReady` callback is called with no arguments when the server [marks the
subscription as ready](#publish_ready). The `onStop` callback is called with
a [`Meteor.Error`](#meteor_error) if the subscription fails or is terminated by
the server. If the subscription is stopped by calling `stop` on the subscription
handle or inside the publication, `onStop` is called with no arguments.

`Meteor.subscribe` returns a subscription handle, which is an object with the
following properties:

<dl class="callbacks">
{% dtdd name:"stop()" %}
Cancel the subscription. This will typically result in the server directing the
client to remove the subscription's data from the client's cache.
{% enddtdd %}

{% dtdd name:"ready()" %}
True if the server has [marked the subscription as ready](#publish_ready). A
reactive data source.
{% enddtdd %}

{% dtdd name:"subscriptionId" %}
The `id` of the subscription this handle is for. When you run `Meteor.subscribe`
inside of `Tracker.autorun`, the handles you get will always have the same
`subscriptionId` field. You can use this to deduplicate subscription handles
if you are storing them in some data structure.
{% enddtdd %}
</dl>

If you call `Meteor.subscribe` within a [reactive computation](#reactivity),
for example using
[`Tracker.autorun`](#tracker_autorun), the subscription will automatically be
cancelled when the computation is invalidated or stopped; it is not necessary
to call `stop` on
subscriptions made from inside `autorun`. However, if the next iteration
of your run function subscribes to the same record set (same name and
parameters), Meteor is smart enough to skip a wasteful
unsubscribe/resubscribe. For example:

```js
Tracker.autorun(() => {
  Meteor.subscribe('chat', { room: Session.get('currentRoom') });
  Meteor.subscribe('privateMessages');
});
```

This subscribes you to the chat messages in the current room and to your private
messages. When you change rooms by calling `Session.set('currentRoom',
'newRoom')`, Meteor will subscribe to the new room's chat messages,
unsubscribe from the original room's chat messages, and continue to
stay subscribed to your private messages.
