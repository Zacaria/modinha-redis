The RedisDocument mixin for [Modinha](https://github.com/christiansmith/Modinha) defines a collection of persistence methods that map cleanly between HTTP semantics and Redis data structures.

### Usage

Suppose we've defined an Account model with Modinha like so:

```javascript
var Modinha = require('modinha')
  , RedisDocument = require('modinha-redis').RedisDocument

var Account = Modinha.define('accounts', {
  email: { type: 'string', required: true, unique: true },
  role:  { type: 'string', secondary: true, enum: ['admin', 'editor', 'author'] },
  hash:  { type: 'string', private: true }
});

Account.extend(RedisDocument);
```

RedisDocument will add the following persistence methods to the Account document.

```
HTTP                     MODEL METHOD

GET    /accounts         Account.list(options, callback)
GET    /accounts/id      Account.get(ids, options, callback)
POST   /accounts         Account.insert(data, options, callback)
PUT    /accounts/id      Account.replace(id, data, options, callback)
PATCH  /accounts/id      Account.patch(id, data, options, callback)
DELETE /accounts/id      Account.delete(id, callback)
```

Extending Account with RedisDocument will also define the following properties on Account.schema:

```javascript
_id:      { type: 'string', required: true, default: Model.defaults.uuid },
created:  { type: 'number', order: true, default: Model.defaults.timestamp },
modified: { type: 'number', order: true, default: Model.defaults.timestamp }
```

Since we defined `unique` and `secondary` properties on email and role, respectively, the mixin will also generate property specific methods for those indexes.

```javascript
Account.getByEmail(email, callback)
Account.listByRole(role, callback)
```

### More about indexing

We can index in a variety of ways with Redis hashes and sorted sets. For example, we could explicitly define our unique email index like so:

```javascript
Account.defineIndex({
  type:  'hash',
  key:   'accounts:email',
  field: 'email',
  value: '_id'
});
```

This tells the model to store an account's `_id` property in a hash named `accounts:email` with email as the field name. Because this is a very common use of the hash type index, the mixin also provides a helper method for defining unique indices:

```javascript
Account.indexUnique('email');
```

This is equivalent to adding `unique: true` to the property definition in our schema.

Sorted set indices get a little more interesting. We have a great deal of flexibility in how we can index our models. For example, suppose we have a `Video` model that has a `category` property and a `likes` property. We want to retrieve a list of videos for a specific category, sorted by the number of likes.

```javascript
Video.defineIndex({
  type:   'sorted',
  key:    ['videos:#:$', 'category', 'category'],
  score:  'likes',
  member: '_id'
});
```

When we index the following instance...

```javascript
{
  _id: 'r4nd0m',
  name: 'Awesome Presentation',
  url: 'https://youtube.com/wh4t3v3r'
  category: 'conferences',
  likes: 777
}
```

... the object's `_id` will be added to a sorted set in Redis called `videos:category:conferences`, with a score of 777. Notice the `key` property of the index definition: `['videos:#:$', 'category', 'category']`. The first element of this array is a template for a key name. In the template, the placeholders `#` and `$` will be replaced in order according to the remaining elements of the array. `#` will be replaced literally with element and `$` will be used to access a property on the object being indexed.

Like the hash-type index, there are a few very common indexing patterns for sorted sets. The mixin provides higher level methods for defining these, and in some cases, they can be created as part of a schema definition. Some examples:

```javascript
Model.indexSecondary(propertyName, [score]);
Video.indexSecondary('category', 'likes');                     // Same as previous example


Model.indexReference(propertyName, ReferencedModel, [score]);
Comment.indexReference('videoId', Video);                      // multi.zadd('videos:ID:comments', comment.created, comment._id);


Model.indexOrder(propertyName);
Comment.indexOrder('likes');


// video schema
{
  name:     { type: 'string', unique: true },
  url:      { type: 'string', unique: true },
  category: { type: 'string', enum: ['tutorial', 'presentation'], secondary: true },
  likes:    { type: 'string', order: true }
}
```


Unique values are enforced by the `insert`, `replace`, and `patch` methods. If you write custom methods, you can use `Account.enforceUnique(callback)` (for example) to generate a UniqueValueError.

The default timestamp methods define an ordered index for created and modified. `Account.list(options, callback)` uses the `accounts:created` index by default to deliver reverse chronological account listings.


