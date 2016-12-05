# feathers-hooks-populate

- [populate](#populate)
    - [Schema](#schema)
    - [Added properties](#added-properties)
    - [Advanced examples](#advanced-examples)
        - [Selecting schema based on UI needs](#selecting-schema-based-on-ui-needs)
        - [Using permissions](#using-permissions)
    - [Relation](#relation)
- [dePopulate][#depopulate]

### populate
`populate(options: Object): HookFunc`

Populates items recursively. 1:1, 1:n and n:1 relationships are supported.

- Used as a **before** or **after** hook on any service method.
- Supports multiple result items, including paginated `find`.
- Provides performance profile information.
- Backward compatible with the old `populate` hook.

```javascript
const schema = {
  permissions: '...',
  include: [
    { // 1:1
      service: 'users',
      nameAs: 'authorItem',
      parentField: 'author',
      childField: 'id',
    },
    { // 1:n
      service: 'comments',
      parentField: 'id',
      childField: 'postId',
      query: {
        $limit: 5,
        $select: ['title', 'content', 'postId'],
        $sort: {createdAt: -1}
      },
      select: (hook, parent, depth) => ({ $limit: 6 }),
      asArray: true,
    },
    { // n:1
      service: 'users',
      permissions: '...',
      nameAs: 'readers',
      parentField: 'readers',
      childField: 'id'
    }
  ],
};

module.exports.after = {
  all: populate({ schema, checkPermissions, profile: true })
};
````

Options

- `schema` [required, object or function] How to populate the items. [Details are below.](#schema)
    - Function signature `(hook: Hook, options: Object): Object`
    - `hook` The hook.
    - `options` The `options` passed to the populate hook.
- `checkPermissions` [optional, default () => true] Function to check if the user is allowed to perform this populate,
or include this type of item. Called whenever a `permissions` property is found.
    - Function signature `(hook: Hook, service: string, permissions: any, depth: number): boolean`
    - `hook` The hook.
    - `service` The name of the service being included, e.g. users, messages.
    - `permissions` The value of the permissions property.
    - `depth` How deep the include is in the schema. Top of schema is 0.
    - Return truesy to allow the include.
- `profile` [optional, default false] If `true`, the populated result is to contain a performance profile.
Must be `true`, truesy is insufficient.

#### Schema

The data currently in the hook will be populated according to the schema. The schema starts with:

```javascript
const schema = {
  permissions: '...',
  include: [ ... ]
};
```

- `permissions` [optional, any type of value] Who is allowed to perform this populate. See `checkPermissions` above.
- `include` [optional, array] Which services to join to the data.

The `include` array has an element for each service to join. They each may have:

```javascript
{ service: 'comments',
  nameAs: 'commentItems',
  permissions: '...',
  parentField: 'id',
  childField: 'postId',
  query: {
    $limit: 5,
    $select: ['title', 'content', 'postId'],
    $sort: {createdAt: -1}
  },
  select: (hook, parent, depth) => ({ $limit: 6 }),
  asArray: true
}
```

- `service` [required, string] The name of the service providing the items.
- `nameAs` [optional, string, default is service] Where to place the items from the join.
- `permissions` [optional, any type of value] Who is allowed to perform this join. See `checkPermissions` above.
- `parentId` [required, string] The name of the field in the parent item for the [relation](#relation).
Dot notation is allowed.
- `childField` [required, string] The name of the field in the child item for the [relation](#relation).
Dot notation is allowed and will result in a query like `{ name.first: 'John' }`
which is not suitable for all DBs.
You may use `query` or `select` to create a query suitable for your DB.
- `query` [optional, object] An object to inject into the query in `service.find({ query: { ... } })`.
- `select` [optional, function] A function whose result in injected into the query.
    - Function signature `(hook: Hook, parentItem: Object, depth: number): Object`
    - `hook` The hook.
    - `parentItem` The parent item to which we are joining.
    - `depth` How deep the include is in the schema. Top of schema is 0.
- `asArray` [optional, boolean, default false] Force the joined item to be stored as an array.

#### Added properties

Some additional properties are added to populated items. The result may look like:

```javascript
{ ...
  _include: [ 'post' ],
  _elapsed: { post: 487947, total: 527118 },
  post:
    { ...
      _include: [ 'authorInfo', 'commentsInfo', 'readersInfo' ],
      _elapsed: { authorInfo: 321973, commentsInfo: 469375, readersInfo: 479874, total: 487947 },
      _computed: [ 'averageStars', 'views'],
      authorInfo: { ... },
      commentsInfo: [ { ... }, { ... } ],
      readersInfo: [ { ... }, { ... } ],
} }
```

- `_include` The property names containing joined items.
- `_elapsed` The elapsed time, in nano-secs, taken to join each item type,
as well as the total taken for them all.
This is normally all DB activity.
- `_computed` The property names containing values computed by the `serialize` hook.

The `depopulate` hook uses these fields to remove all joined and computed values.
This allows you to then `service.patch()` the item in the hook.

#### Advanced examples

##### Selecting schema based on UI needs

Consider a Purchase Order item.
An Accounting oriented UI will likely want to populate the PO with Invoice items.
A Receiving oriented UI will likely want to populate with Receiving Slips.

Using a function for `schema` allows you to select an appropriate schema based on the need.
The following example shows how the client can ask for the type of schema it needs.

```javascript
// on client
purchaseOrders.get(id, { query: { $nonQueryParams: { schema: 'po-acct' }}}) // pass schema to server
// or
purchaseOrders.get(id, { query: { $nonQueryParams: { schema: 'po-rec' }}})
````
```javascript
// on server
const poSchemas = {
  'po-acct': { /* schema for Accounting oriented PO */},
  'po-rec': { /* schema for Receiving oriented PO */}
};

purchaseOrders.before({
  all: hook => { // extract client schema info
    const nonQueryParams = hook.params.query.$nonQueryParams;
    if (nonQueryParams) {
      delete hook.params.query.$nonQueryParams;
      hook.parms = Object.assign({}, hook.params, nonQueryParams);
    }
  }
});
purchaseOrders.after({
  all: populate(() => poSchemas[hook.params.schema])
});
```

##### Using permissions

For a simplistic example,
assume `hook.params.users.roles` is an array of the service names the user may use,
e.g. `['invoices', 'billings']`.
These can be used to control which types of items the user can see.

```javascript
const schema = {
  include: [
    {
      service: 'invoices',
      permissions: 'invoices',
      ...
    }
  ]
};

purchaseOrders.after({
  all: populate(schema, (hook, service, permissions) => hook.params.user.roles.includes(permissions))
});
```

The populate above may only be performed by a user whose `permissions` contains `'invoices'`.

#### Relation

A 1:1 relation is when `parentField` and `childField` are primitive types.
Both are unique in their table.
Exactly one child item can be joined to the parent item.

A 1:n relation is when `parentField` and `childField` are primitive types,
and `childField` is not unique in its table.
Many child items may be joined to the parent item.

A n:1 relation is when `parentField` is an array and `childField` is a primitive type.
Many child items may be joined to the parent item.
A `$in` query operator is used for performance.


## dePopulate
`dePopulate()`

Removes joined and computed properties, as well any profile information.

- Used as a **before** or **after** hook on any service method.
- Supports multiple result items, including paginated `find`.
- Supports an array of keys in `field`.