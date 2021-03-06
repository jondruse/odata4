# OData4

The OData4 gem provides a simple wrapper around the OData Version 4.0 API protocol.
It has the ability to automatically inspect compliant APIs and expose the relevant Ruby objects dynamically.
It also provides a set of code generation tools for quickly bootstrapping more custom service libraries.

**This gem supports [OData Version 4.0](http://www.odata.org/documentation/). Support for older versions is not a goal.**

If you need a gem to integration with OData Version 3, you can use James Thompson's [original OData gem][ruby-odata], upon which this gem is based.

[![Build Status](https://app.codeship.com/projects/da1eb540-ce3f-0135-2ddc-161d5c3cc5fd/status?branch=master)](https://app.codeship.com/projects/262148)
[![Maintainability](https://api.codeclimate.com/v1/badges/f151944dc05b2c7268e5/maintainability)](https://codeclimate.com/github/wrstudios/odata4/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/f151944dc05b2c7268e5/test_coverage)](https://codeclimate.com/github/wrstudios/odata4/test_coverage)
[![Dependency Status](https://gemnasium.com/badges/github.com/wrstudios/odata4.svg)](https://gemnasium.com/github.com/wrstudios/odata4)
[![Documentation](http://inch-ci.org/github/wrstudios/odata4.png?branch=master)](http://www.rubydoc.info/github/wrstudios/odata4/master)

## Installation

Add this line to your application's `Gemfile`:

    gem 'odata4', git: 'https://github.com/wrstudios/odata4'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install odata4

## Usage

### Services & the Service Registry

The OData4 gem provides a number of core classes, the two most basic ones are the `OData4::Service` and the `OData4::ServiceRegistry`.
The only time you will need to worry about the `OData4::ServiceRegistry` is when you have multiple OData4
services you are interacting with that you want to keep straight easily.
The nice thing about `OData4::Service` is that it automatically registers with the registry on creation, so there is no manual interaction with the registry necessary.

To create an `OData4::Service` simply provide the location of a service endpoint to it like this:

```ruby
  OData4::Service.open('http://services.odata.org/V4/OData/OData.service')
```

You may also provide an options hash after the URL.
It is suggested that you supply a name for the service via this hash like so:

```ruby
  OData4::Service.open('http://services.odata.org/V4/OData/OData.service', name: 'ODataDemo')
```

This one call will setup the service and allow for the discovery of everything the other parts of the OData4 gem need to function.
The two methods you will want to remember from `OData4::Service` are `#service_url` and `#name`.
Both of these methods are available on instances and will allow for lookup in the `OData4::ServiceRegistry`, should you need it.

Using either the service URL or the name provided as an option when creating an `OData4::Service` will allow for quick lookup in the `OData4::ServiceRegistry` like such:

```ruby
  OData4::ServiceRegistry['http://services.odata.org/V4/OData/OData.service']
  OData4::ServiceRegistry['ODataDemo']
```

Both of the above calls would retrieve the same service from the registry.
At the moment there is no protection against name collisions provided in `OData4::ServiceRegistry`.
So, looking up services by their service URL is the most exact method, but lookup by name is provided for convenience.

### Authentication

When authenticating with your service you can set parameters to the Typhoeus gem which uses libcurl.
Use the **:typhoeus** option to set your authentication.

For example using **ntlm** authentication:

```ruby
  conn = OData4::Service.open('http://services.odata.org/V4/OData/OData.service', {
    name: 'ODataDemo',
    typhoeus: {
      username: 'username',
      password: 'password',
      httpauth: :ntlm
    }
  })
```

For more authentication options see [libcurl][libcurl] or [typhoeus][typhoeus].

[libcurl]: http://curl.haxx.se/libcurl/c/CURLOPT_HTTPAUTH.html
[typhoeus]: https://github.com/typhoeus/typhoeus

### Metadata File

Typically the metadata file of a service can be quite large.
You can speed your load time by forcing the service to load the metadata from a file rather than a URL.
This is only recommended for testing purposes, as the metadata file can change.

```ruby
  conn = OData4::Service.open('http://services.odata.org/V4/OData/OData.service', {
      name: 'ODataDemo',
      metadata_file: "metadata.xml",
  })
```

### Headers

You can set the headers with the **:typhoeus** param like so:

```ruby
  conn = OData4::Service.open('http://services.odata.org/V4/OData/OData.service', {
    name: 'ODataDemo',
    typhoeus: {
      headers: {
        "Authorization" => "Bearer #{token}"
      }
    }
  })
```

### Entity Sets

When it comes to reading data from an OData4 service the most typical way will be via `OData4::EntitySet` instances.
Under normal circumstances you should never need to worry about an `OData4::EntitySet` directly.
For example, to get an `OData4::EntitySet` for the products in the ODataDemo service simply access the entity set through the service like this:

```ruby
  service = OData4::Service.open('http://services.odata.org/V4/OData/OData.service')
  products = service['ProductsSet'] # => OData4::EntitySet
```

`OData4::EntitySet` instances implement the `Enumerable` module, meaning you can work with them very naturally, like this:

```ruby
  products.each do |entity|
    entity # => OData4::Entity for type Product
  end
```

You can get a list of all your entity sets like this:

```ruby
  service.entity_sets
```

#### Count
Some versions of Microsoft CRM do not support count.

```ruby
  products.count
```

#### Collections
You can you the following methods to grab a collection of Entities:

```ruby
  products.each do |entity|
    ...
  end
```

The first entity object returns a single entity object.

```ruby
  products.first
```

`first(x)` returns an array of entity objects.

```ruby
  products.first(x)
```

#### Find a certain Entity

```ruby
  service['ProductsSet']['<guid of entity>']
```


### Entities

`OData4::Entity` instances represent individual entities, or records, in a given service.
They are returned primarily through interaction with instances of `OData4::EntitySet`.
You can access individual properties on an `OData4::Entity` like so:

```ruby
  product = products.first # => OData4::Entity
  product['Name']  # => 'Bread'
  product['Price'] # => 2.5 (Float)
```

Individual properties on an `OData4::Entity` are automatically typecast by the gem, so you don't have to worry about too much when working with entities.
The way this is implemented internally guarantees that an `OData4::Entity` is always ready to save back to the service or `OData4::EntitySet`, which you do like so:

```ruby
  service['Products'] << product # Write back to the service
  products << product        # Write back to the Entity Set
```

You can get a list of all your entities like this:

```ruby
  service.entity_types
```

#### Entity Properties
Reading, parsing and instantiating all properties of an entity can add up to a significant amount of time, particularly for those entities with a large number of properties.
To speed this process up all properties are lazy loaded.
Which means it will store the name of the property, but will not parse and instantiate the property until you want to use it.

You can find all the property names of your entity with

```ruby
  product.property_names
```

You can grab the parsed value of the property as follows:

```ruby
  product["Name"]
```

or, you can get a hold of the property class instance using

```ruby
  product.get_property("Name")
```

This will parse and instantiate the property if it hasn't done so yet.

### Queries

`OData4::Query` instances form the base for finding specific entities within an `OData4::EntitySet`.
A query object exposes a number of capabilities based on
the [System Query Options](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part1-protocol/odata-v4.0-errata03-os-part1-protocol-complete.html#_Toc453752288) provided for in the OData V4.0 specification.
Below is just a partial example of what is possible:

```ruby
  query = service['Products'].query
  query.where(query[:Price].lt(15))
  query.where(query[:Rating].gt(3))
  query.limit(3)
  query.skip(2)
  query.order_by("Name")
  query.select("Name,CreatedBy")
  query.inline_count
  results = query.execute
  results.each {|product| puts product['Name']}
```

The process of querying is kept purposely verbose to allow for lazy behavior to be implemented at higher layers.
Internally, `OData4::Query` relies on the `OData4::Query::Criteria` for the way the `where` method works.
You should refer to the published RubyDocs for full details on the various capabilities:

 * [OData4::Query](http://rubydoc.info/github/wrstudios/odata4/master/OData4/Query)
 * [OData4::Query::Criteria](http://rubydoc.info/github/wrstudios/odata4/master/OData4/Query/Criteria)

## Contributing

1. Fork it (`https://github.com/[my-github-username]/odata/fork`)
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

## Credits

Many thanks go to [James Thompson][@plainprogrammer], who wrote the [original OData (Version 3.0) gem][ruby-odata].

[@plainprogrammer]: https://github.com/plainprogrammer
[ruby-odata]: https://github.com/ruby-odata/odata

Also, I would like to thank [W+R Studios][wrstudios] for generously allowing me to work on Open Source software like this. If you want to work on interesting challenges with an awesome team, check out our [open positions][wrcareers].

[wrstudios]: http://wrstudios.com/
[wrcareers]: http://wrstudios.com/careers
