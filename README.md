# Holycorn: PostgreSQL multi-purpose Ruby data wrapper

(m)Ruby + PostgreSQL = &lt;3

Holycorn makes it easy to implement a Foreign Data Wrapper using Ruby.

It is based on top of mruby, that provides sandboxing capabilities the regular
Ruby VM "MRI/CRuby" does not provide.

[![Join the chat at https://gitter.im/franckverrot/holycorn](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/franckverrot/holycorn?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
## Built-in Wrappers

Holycorn embeds its own gems at compile-time, as this is the way to work with
gems with mruby.

All the following wrappers are currently linked against Holycorn:

  * `Redis`, using the `mruby-redis` gem

## INSTALLATION

### Prerequisites

* PostgreSQL 9.1+

### Setup

Simply run

    rake

to vendor and build `mruby`.

Now that `mruby` is built, building `holycorn` requires to run

    make

and installing it only requires to run

    make install

Now connect to PostgreSQL and install the extension:

    λ psql
    psql (9.4.1)
    Type "help" for help.

    DROP EXTENSION holycorn CASCADE;
    CREATE EXTENSION holycorn;

#### Using Builtin Foreign Data Wrappers

A set of builtin FDW are distributed with Holycorn for an easy setup. All one
needs to provide are the options that will allow the FDW to be configured:

    λ psql
    psql (9.4.1)
    Type "help" for help.

    CREATE SERVER holycorn_server FOREIGN DATA WRAPPER holycorn;
    CREATE FOREIGN TABLE redis_table (key text, value text)
      SERVER holycorn_server
      OPTIONS (wrapper_class 'HolycornRedis', host '127.0.0.1', port '6379', db '0');

As `Holycorn` doesn't support `INSERT`s yet, let's create some manually:

```console
λ redis-cli
127.0.0.1:6379> select 0
OK
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> set foo 1
OK
127.0.0.1:6379> set bar 2
OK
127.0.0.1:6379> set baz 3
OK
127.0.0.1:6379> keys *
1) "bar"
2) "foo"
3) "baz"
```

Now that the table has been created and we have some data in Redis, we can
select data from the foreign table.

```sql
SELECT * from redis_table;

 key | value
-----+-------
 bar | 2
 foo | 1
 baz | 3
(3 rows)
```

#### Using custom scripts

Alternatively, custom scripts can be used as the source for a Foreign Data Wrapper:

```sql
DROP EXTENSION holycorn CASCADE;
CREATE EXTENSION holycorn;
CREATE SERVER holycorn_server FOREIGN DATA WRAPPER holycorn;
CREATE FOREIGN TABLE holytable (some_date timestampz) \
  SERVER holycorn_server
  OPTIONS (wrapper_path '/tmp/source.rb');
```

And the source file of the wrapper:

```ruby
# /tmp/source.rb
class Producer
  def initialize(env = {}) # env contains informations provided by Holycorn
  end

  def each
    @enum ||= Enumerator.new do |y|
      10.times do |t|
        y.yield [ Time.now ]
      end
    end
    @enum.next
  end
  self
end
```

Now you can select data out of the wrapper:

    λ psql
    psql (9.4.1)
    Type "help" for help.

    franck=# SELECT * FROM holytable;
          some_date
    ---------------------
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
     2015-06-21 22:39:24
    (10 rows)

Pretty neat.

# SUPPORTED SCRIPTS

## General rules

Any type of Ruby object can act as a FDW. The only requirements are that it can
receive `.new` (with arity = 1) and return an object that can receive `each` (arity = 0).

It doesn't **have** to be a `Class`, and there's currently no will to provide a
superclass to be inherited from.

In future versions, there will be many more callbacks to interact with PG's FDW
infrastructure through `Holycorn`.

Also, the script can only be a single word - like `MyClass` - as long as
`MyClass` has been defined and exists within your compilation of `mruby`.


## Environment

A hash is passed by `Holycorn` to the Ruby script. Its current keys are:

* `PG_VERSION`
* `PG_VERSION_NUM`
* `PACKAGE_STRING`
* `PACKAGE_VERSION`
* `MRUBY_RUBY_VERSION`
* `WRAPPER_PATH`


# SUPPORTED TYPES (Ruby => PG)

## Builtin types

  * `MRB_TT_FREE`      => `null`
  * `MRB_TT_FALSE`     => `Boolean`
  * `MRB_TT_TRUE`      => `Boolean`
  * `MRB_TT_FIXNUM`    => `Int64`
  * `MRB_TT_SYMBOL`    => `Text`
  * `MRB_TT_UNDEF`     => Unsupported
  * `MRB_TT_FLOAT`     => `Float8`
  * `MRB_TT_CPTR`      => Unsupported
  * `MRB_TT_OBJECT`    => `Text` (`to_s` is called)
  * `MRB_TT_CLASS`     => `Text` (`class.to_s` is called)
  * `MRB_TT_MODULE`    => `Text` (`to_s` is called)
  * `MRB_TT_ICLASS`    => Unsupported
  * `MRB_TT_SCLASS`    => Unsupported
  * `MRB_TT_PROC`      => `Text` (`inspect` is called)
  * `MRB_TT_ARRAY`     => `Text` (`inspect` is called)
  * `MRB_TT_HASH`      => `Text` (`inspect` is called)
  * `MRB_TT_STRING`    => `Text`
  * `MRB_TT_RANGE`     => `Text` (`inspect` is called)
  * `MRB_TT_EXCEPTION` => Unsupported
  * `MRB_TT_FILE`      => Unsupported
  * `MRB_TT_ENV`       => Unsupported
  * `MRB_TT_DATA`      => See "Arbitraty Ruby objects" section
  * `MRB_TT_FIBER`     => `Text` (`inspect` is called)
  * `MRB_TT_MAXDEFINE` => Unsupported

## Arbitraty Ruby objects

  * Time (Ruby) => `timestampz`

## CONFIGURATION

### Server

None (yet).

### Foreign Table

Either `wrapper_class` or `wrapper_path` can be used to defined whether an
external script or a built-in wrapper will manage the foreign table.

* `wrapper_class`: Name of the built-in wrapper class
* `wrapper_path`: Path of a custom script

In both case, any other option will be pushed down to the wrapper class via the
constructor.


## TODO

- [ ] Array type
- [ ] JSON type
- [ ] Range type
- [ ] Support PG 9.5's `IMPORT FOREIGN SCHEMA` for easy setup

## Note on Patches/Pull Requests

* Fork the project.
* Make your feature addition or bug fix.
* Add tests for it. This is important so I don't break it in a future version unintentionally.
* Commit, do not mess with version or history. (if you want to have your own version, that is fine but bump version in a commit by itself I can ignore when I pull)
* Send me a pull request. Bonus points for topic branches.

## LICENSE

Please see [LICENSE][license] for licensing details.

## AUTHOR

Franck Verrot, @franckverrot

[license]: https://github.com/franckverrot/holycorn/blob/master/LICENSE.md
