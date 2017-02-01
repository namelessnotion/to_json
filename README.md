# ToJson

A performant Ruby JSON Serializer DSL for Oj. ToJson uses the brand new
Oj StringSerializer to provide the fastest performance and lowest
possible memory footprint.

Why? Because current Ruby JSON serialisers take too long and use too
much memory or can't express **all** valid JSON structures.

ToJson is ORM and ruby web framework agnostic and designed for serving fast
and flexible JSON APIs.

ToJson is able to serialize an impressive 1.4 million operations a second
on a 4 core laptop when running multiple ruby processes.

## Installation

Add this line to your application's Gemfile:

Do this for now:

    gem 'to_json', github: 'ahacking/to_json'

Eventually:

    gem 'to_json'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install to_json

## Usage

## General Invocation with block

```ruby
  # args are optional
  ToJson::Serializer.json!(args...) do |args...|
    # DSL goes here, callers methods, helpers, instance variables and constants are all in scope
  end
end
```

### Invocation from Rails controller, respond_with and block

```ruby
def index
  @post = Post.all
  # the rails responder will call to_json on the ToJson object
  respond_with ToJson::Serializer.encode! do
    # DSL goes here, contoller methods, helpers, instance variables and
    # constants are all in scope
  end
end
```

### Invocation from Rails API controller, render with block (better)

```ruby
def index
  @post = Post.all
  # generate the json and pass it to render for sending to the client
  render json: ToJson::Serializer.json! do
    # DSL goes here, contoller methods, helpers, instance variables and
    # constants are all in scope
  end
end
```

### Invocation from Rails API controller with custom serializer class (recommended)

```ruby
def index
  # just pass the collection (instead of the controller) to better support
  # serializing Posts in different contexts and controllers. @foo is evil
  render json: PostsSerializer.json!(Post.all)
end
```

### JSON Objects

The `put` method is used to serialize named object values and
create arbitrarily nested objects.

All values will be serialized according to Oj processing rules.

#### Example creating an object with named values:

```ruby
put :title, @post.title
put :body, @post.body
```

#### Example with fields helper

```ruby
put_fields @post, :title,  :body
```

#### Example with fields helper and key mapping.

The DSL accepts array pairs, hashes, arrays containing any
mix of array or hash pairs.

The following examples are all equivalent and map 'title' to 'the_tile'
and 'created_at' to 'post_date' and leave 'body' as is.

```ruby
put_fields @post, [:title, :the_title], :body, [:created_at, :post_date]
put_fields @post, [[:title, :the_title], :body, [:created_at, :post_date]]
put_fields @post, {title: :the_title, body: nil, created_at: :post_date}
put_fields @post, [:title, :the_title], :body, {:created_at => :post_date}
put_fields @post, {title: :the_title}, :body, {created_at: :post_date}
```

#### Example with fields helper with condition.

There are helpers to serialize object fields conditionally.

```ruby
put_fields_unless_blank @post, :title: :body
put_fields_unless_nil @post, :title: :body
put_fields_unless :large?, @post, :title: :body
put_fields_if :allowed, @post, :title: :body
```

#### Example of serializing a single field

There are single field equivalents of the multiple field helpers. these
take an optional mapping key and just like put they accept a block.

```ruby
put_field @post, :title
put_field @post, :title, :the_title
put_field_unless_blank @post, :title, :the_title
put_field_unless_nil @post, :title, :the_title
put_field_unless :large? @post, :body
put_field_if :allowed? @post, :body
```

#### Example creating a nested object

The long way:

```ruby
put :post do
  put :title, @post.title
  put :body, @post.body
end

Using field helper:

```ruby
put :post do put_fields @post, :title :body end
```

### Example of a named object literal

The hash value under 'author' will be serialized directly by Oj.

```ruby
put :author, {name: 'Fred', email: 'fred@example.com', age: 27}
```

### Example of an object literal

The hash value will be serialized by Oj.

```ruby
value {name: 'Fred', email: 'fred@example.com', age: 27}
```

#### Example creating a nested object with argument passed to block

```ruby
put :latest_post, current_user.posts.order(:created_at: :desc).first do |post|
  put_fields post, :title, :body
end
```

### JSON Arrays

Arrays provide aggregation in JSON and are created with the `array` method.  Array
elements can be created through:
+ literal value(s) passed to `array` without a block
+ evaluating blocks over the argument passed to array (similar to `each_with_index`)
+ evaluating a block with no argument

Within the array block, array elements can be created using `value`, however this is
called implicitly for you when using `put` or `array` inside the array block.

### Example of an array literal

The literal array value will be passed to Oj for serialization.

```ruby
array ['Fred', 'fred@example.com', 27]
```

### Example of an array collection

The @posts collection will be passed to Oj for serialization.

```ruby
array @posts
```

### Example of array with block for custom object serialization

```ruby
array @posts do |post|
  # calling put/put_* inside an array does an implicit 'value' call
  # placing all named values into a single object
  put_fields post, :title, post.body
end
```

### Example of array with block and item index for custom object serialization

```ruby
array @posts do |post, index|
  put_fields post, :title, post.body
  put :position, index
end
```

### Example collecting post author emails into a single array.

Each post item will be processed and the email addresses of the author
serialized.

```ruby
array @posts do |post|
  @post.author.emails.each do |email|
    value email.address
  end
end
```

### Example creating array element values explicitly

The following example will an array containing 3 elements.

```ruby
array do
  value 'one'
  value 2
  value do
    put label: 'three'
  end
end
```

### Example creating array with a nested object and nested collection

```ruby
array do
  value do
    put :total_entries, @posts.total_entries
    put :total_pages, @posts.total_pages
  end
  array @posts do
    put :title, post.title
    put :body, post.body
  end
end
```

### Example creating a paged collection as per the HAL specification:

```ruby
put :meta do
  put_fields @posts, :total_entries, :total_pages
end
put :collection do
  array @posts do |post| put_fields post, :title, :body end
end
put :_links do
  put :self { put :href, url_for(page: @posts.current_page) }
  put :first { put :href, url_for(page: 1) }
  put :previous { @posts.current_page <= 1 ? nil : put :href, url_for(page: @posts.current_page-1) }
  put :next { current_page_num >= @posts.total_pages ? nil : put :href, url_for(page: @posts.current_page+1) }
  put :last { put :href, url_for(page: @posts.total_pages) }
end
```

### Example of nested arrays, and dynamic array value generation:

```ruby
array do
  # this nested array is a single value in the outer array
  array do
    value 'a'
    value 'b'
    value 'b'
  end
  # this nested array is a single value in the outer array
  array (1..3)
    (1..4).each do |count|
      # generate 'count' values in the nested array
      count.times { value "item #{count}" }
    end
  end
end
```

### Example of defining and using a helper

```ruby
def fullname(*names)
  names.join(' ')
end

put :author, fullname(@post.author.first_name, @post.author.last_name)
```

### Example of class based serialization and composition:


```ruby
# A Post model serializer, using ::ToJson::Serializer inheritance
class PostSerializer < ::ToJson::Serializer
  include PostSerialization

  # override the serialize method and use the ToJson DSL
  # any arguments passed to encode! or json! are passed into serialize
  def serialize(model)
    put_post_nested model
  end
end

# A Post collection serializer using include ToJson::Serialize approach
class PostsSerializer
  include PostSerialization

  def serialize(collection)
    put_posts collection
  end
end

# define a module so we can mixin Post model serialization concerns
anywhere and avoid temporary serializer objects for collection items
module PostSerialization
  include  ::ToJson::Serialize

  # formatting helper
  def fullname(*names)
    names.join(' ')
  end

  def put_post(post)
    put :title, post.title
    put :body, post.body
    put :author, fullname(post.author.first_name, post.author.last_name)
    put :comments, CommentsSerializer.new(post.comments)
  end

  def put_post_nested(post)
    put :post do
      put_post(post)
    end
  end

  def serialize_posts(posts)
    put :meta do
      put :total_entries, posts.total_entries
      put :total_pages, posts.total_pages
    end
    put :collection, posts do |post|
      put_post post
    end

  end
end
```

## ToDo

+ Tests and more tests.
+ API Documentation.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
