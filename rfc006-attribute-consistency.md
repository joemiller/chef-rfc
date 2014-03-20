# Attribute Consistency

This proposal has two parts:

1. Normal level attributes should not be persisted. A new API should be
   added to address the persistence use case.
2. System profiling attributes should be accessed by a different API
   than other node attributes

## Normal Attribute Persistence

Persisting 'normal' level attributes creates a confusing, surprising
special case in attribute behavior, which makes documentation and best
practices more complicated.

This results in the following user pain points:
* learning chef attributes is more complicated, since documentation and
  recommendations are more complicated to understand.
* "Phantom" attributes. Users remove an attribute, or modify a
  complicated data structure, but their changes are mysteriously not
  applied because the normal attributes from a previous run are not
  cleared.
* Recipes that generate node data that needs to be persisted must call
  `node.save` manually, to ensure that persisted data is not lost in the
  event that the chef-client run subsequently fails. This causes issues
  with search data consistency, increased server load, and other
  problems.

The benefit of normal attributes persistence behavior is that they are
persisted, not that they are attributes. Common use cases include
storing data generated on the node (such as autogenerated passwords) or
exposing a mechanism for triggering a different behavior in a cookbook
by changing a node attribute using the server API. These use cases would
be equally well served by having a completely separate API for
persisting node-specific data.

### Node Specific Persistent Data API.

When considering the user-facing API for persistent node data, we should
consider the following factors:

* When are updates stored on the server? We could write the feature so
that any assignment of new data triggers a save, users save changes
manually, or changes are stored in bulk at the conclusion of a
chef-client run.
* Should there be any sort of conflict detection? This may be important
if simultaneous modification of data (by both chef-client and by a human
user via HTTP API) is prevalent.
* What does search look like?

### Swag API

```ruby
node.storage.fetch(:namespace1, :namespace2, ...) # => value or nil

node.storage.set[:namespace1, :namespace2, ...] = value

node.storage.clear(:namespace1, :namespace1, ...)
```

This was designed quickly and probably has lots of flaws. The key point
is that the `set` API uses a single method call, which allows for
values to be saved immediately as part of the call to the
`[]=(*keyspace, value)` method call.

### Release Timing

This proposal is a breaking change. We have a few options:

1. Make the change all at once, for Chef 12
2. Introduce the new storage API ASAP, issue deprecation warnings
   whenever normal attributes are used in Chef 11.next, and remove
   normal attribute persistence in Chef 12.
3. Make the change all at once in Chef 12, but with a configuration
   option to enable the old behavior. Remove the option in Chef 13.
4. Deprecate normal attribute usage in Chef 12, and remove persistence
   in Chef 13.

One thing to note is that deprecation without behavior change
effectively makes normal level attributes unusable until the persistence
behavior is removed.


## Separate System Profiling Attributes

Currently node attributes set by users (via cookbooks, etc.) share a
namespace with those collected by ohai. This is problematic for several
reasons:

* Since ohai's attributes have the highest precedence level, any time a
new attribute is added (for example, adding a new plugin to core ohai),
there is risk that it may stomp on attributes users were already using
for cookbooks. Luckily we haven't had any major bugs arise from this,
but one can get a feel for the risk involved by imagining an ohai plugin
began providing a 'mysql' attribute.
* It mixes actual with desired state. Attributes set by roles,
cookbooks, etc. are "aspirational"--they reflect what the node should
become. Ohai's attributes are "informational"--they reflect data
collected from the actual system at some point in time.
* It mixes user-controlled namespace with system controlled namespace.
This is a conceptual issue as well as a potential source of bugs as
described above.

### System Attributes API SWAG

One problem here is that "system" is the obvious word to use, but this
is the name of a method defined on `Kernel`, so there could be some
negative consequences of using it. One alternative is to shorten it to
`sys`.

#### Accessed via `node`:

```ruby
node.sys[:kernel][:machine]
node.sys[:fqdn] 
```

#### Accessed via delegate method:

To reduce repetitive typing, we can delegate the `sys` method to `node`
in all places that `node` is defined in the DSL. Then you can access
these values like so:

```
sys[:kernel][:machine]
sys[:fqdn] 
```

### Release Timing

In this case there is little downside to taking a longer time to
deprecate and eventually remove the current behavior. I think the best
course of action is to deprecate access of ohai data via combined
attributes in Chef 12 and remove it in Chef 13. The new APIs for
accessing system data can be introduced at any time before Chef 12.

As part of the deprecation process, we can announce that potential
overlap between ohai attributes and user-defined attributes will no
longer be considered a breaking change during the Chef 12 release cycle,
which achieves all of the goals of this proposal without breaking
anything immediately upon the release of Chef 12.0.
