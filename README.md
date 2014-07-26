# Sereth JSON Spec

  ***Beta Version - Use at your own risk***

  **Requires Ruby 2.0**

  Sereth JSON Spec is a ruby library based on the idea of treating data instances like
  context sensitive API objects exported through JSON. 

  The library may also be used to generate examples of blank data schemas to populate
  scripts, and to export to web-based API editors.
  
## Content
1. [Installation](#installation)
2. [Usage](#usage)
  - [Initialization](#initialization)
  - [Schemas](#schemas)
  - [Export](#export)
  - [Import](#import)
3. [API Overview](#api-overview)
  - [Basic Nodes](#basic-nodes)
      * [Dynamic Export](#dynamic-export)
      * [Typed Export](#typed-export)
      * [Default Export](#default-export)
      * [Basic Import](#basic-import)
      * [Extended Import](#extended-import)
      * [Typed Import](#typed-import)
  - [Collections](#collections)
      * [Non-Array Collections Export](#non-array-collections-export)
      * [Dynamic Collections Export](#dynamic-collections-export)
      * [Typed Collections Export](#typed-collections-export)
      * [Collection Import](#collection-import)
  - [Objects](#objects)
      * [Object Collections](#object-collections)
      * [Glue Object](#glue-object)
      * [Object Extension](#object-extension)
      * [Object Import](#object-import)
  - [Overrides](#overrides)
  - [Conditionals/Execution break-in](#conditionalsexecution-break-in)
4. [Benchmarks](#benchmakrs)

## Design Considerations

  A spec is defined on a ruby model. While this may be against Rails design
  principles, there is no easy way to accomplish all required features in views.

  > TODO: Allow for imports from views into the model classes. On access to the
  > to_json method compile all views for this object, and cache these values to
  > the model proper.

## Installation

Instructions necessary before you can prepend Sereth::JsonSpec in order to define a spec.

### Ruby
**Step 1:** Run `gem install sereth_json_spec`
**Step 2:** 
```ruby
require 'rubygems'
require 'sereth_json_spec'
```

### Rails
**Step 1:** Edit Gemfile and add `gem 'sereth_json_spec'`
**Step 2:** Run `bundle install`



## Usage
### Initialization
  Perspectives are configured within the scope of the Data object class. See the 
  [API](#api-overview) for configuration options.
```ruby
class Data
  # Some record initialization
  # ...
  
  # Enable JsonSpec for this object
  prepend Sereth::JsonSpec
  
  # Perspective initializations
  json_spec :name do
    # Configuration for spec imports and exports
  end
end
```

### Schemas
  The system may be used to generate an empty JSON schema, which lays out the key names,
  and JSON level data types (objects, arrays, values).
```ruby
  Data.json_spec_schema(:name)
```

  ***Note***: In the future this feature may be expanded to provide info about value data types,
  size constraints, and other bits of info that may be useful in an online API editor.


### Export
  May be used to render an object into a JSON output as configured by the spec.
```ruby
  render :json => data_inst.as_json(:spec => :name)
  # OR
  render :text => data_inst.to_json(:spec => :name)  
```

### Import
  May be used to load a hash generated by JSON.parse (or text to be passed into it)
  into a target object using the import 
```ruby
  # Existing Instance
  data_inst.from_json(text_or_hash, :spec => :name)

  # Save resulting data
  data_inst.save
```

### Perspective Path Gotcha
  The system will store all names spec at a given path. If path is not specified
  it will try to generate it from the collection name of the data object, or failing that
  the class name of the object.

  This means that any specs used in objects without collection names, and without
  a hard-coded path will overwrite other similarly named specs.

##
## API Overview
##
  Note, all values pass through `.to_json` unless otherwise noted
```ruby
json_spec :spec_name do
  ## Nodes
  # Key-Value Nodes
  node_name #=> "node_name": #{inst.node_name}
  node_name :symbol #=> "node_name": #{inst.symbol)}
  node_name get: proc, set: proc {|value|} #=> "node_name": #{inst.instance_eval(&proc)}

  # Typed Key-Value Nodes (Exception on invalid types)
  node_name type: Type, ... # Same operation as normal Key-Value nodes

  # Key-Array Nodes
  node_name Array #=> "node_name": [#{inst.node_name.each {|val| val}}]
  node_name Array, :symbol #=> "node_name": [#{inst.node_name.each(&:symbol)}]
  node_name Array, get: proc set: proc {|parsed_array|} #=> "node_name": [#{inst.instance_eval(&proc)}]

  # Typed Key-Array Nodes (Exception on invalid types)
  node_name Array, type: Type, ... # Same operation as normal Key-Array nodes

  # Key-Object Nodes 
  node_name do ... end 
    #=> "node_name": {#{json_spec.from(&block).apply(inst.node_name)}}

  # Key-Array of Object Nodes
  node_name Array do ... end 
    #=> "node_name": [#{inst.node_name.each {|item| json_spec.from(&block).apply(item)}}]

  ## Operations
  override! :node_name, ... # Identical functionality to above. Useful for ruby keyword names
  extends! :spec_name # Extend from a different spec in the same node
  extends! DataClass, :spec_name # Extend from a different spec for a different node
  extends! 'collection_name', :spec_name # Sambe as above, for rails collection names
  if! proc do ... end # Exted current spec if proc returns true in context of current inst
end
```

##
## Basic Nodes
##
  Raw data nodes can be named within the json_spec block
```ruby
# Definition
json_spec :basic do
  id
end

# Result
data_inst.to_json(spec: :basic) 
  #=> {"id": #{data_inst.id.to_json}}

data_inst.to_json(spec: :basic, escape: true) 
  #=> {\"id\": #{data_inst.id.to_json.to_json}}


# Schema Result
Data.json_spec_schema(:typed)
  #=> {"id": "BasicValue"}  
```

### Dynamic Export
  More fine grained control of the node value can be achieved with procs
```ruby
# Definition
json_spec :basic_proc do
  id get: proc {other_id}
  tag :email
end

# Result
data_inst.to_json(spec: :basic_proc) 
  #=> {
    # "id": "#{data_inst.other_id.to_json}",  
    # "tag": #{data_inst.email.to_json}}
```

### Typed Export
  Raw data nodes may specify a data type. This is used primarily for schema generation,
  for use with ustream editors. 

  However, the system will also verify the data type of the object before writing it into 
  JSON from an export, and will attempt to parse the JSON input into this type during import.
```ruby
# Definition
json_spec :typed do
  id type: Integer
end

# Result
data_inst.to_json(spec: :typed) 
  #=> {"id": #{data_inst.id.to_json}}  

# Schema Result
Data.json_spec_schema(:typed)
  #=> {"id": "Integer"}  
```

  ***Important***: If the data type does not match the generator will emit an error. 
```ruby
# Definition
json_spec :typed_inval do
  not_a_string type: String
end

# Result
data_inst.to_json(spec: :typed_inval) 
  #=> Error: not_a_string fails data type contraints
```

### Default Export
The Dynamic Node generation may also supply default values

```ruby
# Definition
json_spec :basic_def do
  word get: proc {"hello"}
end

# Result
data_inst.to_json(spec: :basic_def) 
  #=> {"word": #{"hello".to_json}}
```

### Basic Import
The spec system may also be used to update a data object based on an existing spec.

```ruby
  # Definition
json_spec :basic_set do
  custom set: proc {|value| set_custom('name', value, force: true)}
end

# Result
data_inst.from_json({custom: 1}, spec: :basic_set) 
# Same effect as: 
#   data_inst.set_custom('name', 1, force: true)
```

### Extended Import
Setter shorthands may be used to provide common functionality without using a proc.

```ruby
  # Definition
json_spec :extended_set do
  basic set: :set_basic
  complex set: :set_attribute.caller_shift(:complex)
  word set: :assign.caller_push(:word)
end

# Result
data_inst.from_json({basic: 1, complex: 2, word: "hi"}, spec: :extended_set) 
# Same effect as: 
#   data_inst.set_basic(1)
#   data_inst.set_attribute(:complex, "hi")
#   data_inst.assign("hi", :word)
```

### Typed Import
For typed nodes, if the typed class responds to the "from_json" signal, the value passed
to the setter will be parsed through this method.

```ruby
class Foo
  def self.from_json(value) ... end
end

  # Definition
json_spec :basic_set do
  typed type: Foo, set: :set_typed
end

# Result
data_inst.from_json({typed: "raw_data"}, spec: :basic_set) 
# Same effect as: 
#   data_inst.set_typed(Foo.from_json("raw_data"))
```

##
## Collections
##
  Collections must be denoted as such by speficying the Array data type after the node 
  name. 
```ruby
# Definition
json_spec :col do
  nodes Array
end

# Result
data_inst.to_json(spec: :col) #=> {"nodes": [nodes[0].to_json, nodes[1].to_json...]}  

# Schema Result
Data.json_spec_schema(:typed)
  #=> {"id": [... collection_schema ...]}  
```

### Non-Array Collections Export
  If a specified value is not an array, it will be added to a single-element Array.
```ruby
# Assume
data_inst.key = "asdf"

# Definition
json_spec :col_non_array do
  key Array
end

# Result
data_inst.to_json(spec: :col_non_array) #=> "{"key": ["asdf".to_json]}"
```

### Dynamic Collections Export
  Collection generation may be extended with blocks or generator functions.
```ruby
# Definition
json_spec :col_block do
  nodes Array, :symbol
  others Array, get: proc {gen_other}
  data_ids Array, get: proc{data.map(&:id)}
end

# Result
data_inst.to_json(spec: :col_block) #=> {
  # "nodes": [symbol[0].to_json, symbol[1].to_json...],
  # "others": [gen_other[0].to_json, gen_others[1].to_json...]}
  # "data_ids": [data[0].id.to_json, data[1].id.to_json...]}
```


### Typed Collections Export
  Data nodes in a collection may specify a data type. This will ensure that all members
  of the collection are of a given data type before generating the JSON object.

  Generating a schema from a typed collection will set the element of that collection to 
  that type name.
```ruby
# Definition
json_spec :typed do
  post_ids Array, type: Integer, get: proc {posts.map(&:id)}
end

# Result
#=> {"post_ids": [post1.id, post2.id...]}  

# Schema Result
Data.json_spec_schema(:typed)
  #=> {"post_id": ['Integer']}  
```

### Collection Import
  The spec system does not try to impose special rules upon array imports. 

  The setter will execute in the context of the array parent object, and the developer
  will receive access to an array of raw values, which may then be parsed as necessary.

```ruby
# Assume
data_inst.keys = [1, 3]

# Definition
json_spec :collection_import do
  keys Array, set: proc {|values| self.keys = self.keys | values}
end

# Result
data_inst.from_json('{keys: [2, 3, 4]}', spec: :collection_import) 
#=> data_inst.keys == [1, 2, 3, 4]


# Schema Result
Data.json_spec_schema(:typed)
  #=> {"post_id": ['Integer']}  
```

##
## Objects
##
  New objects may be embedded in an existing spec as blocks. The resulting context will
  determine if the data instance has a context of a given name, and will run within
  that context if present.

  **Important:** Objects can not be imported. All imports must be performed on object keys.

```ruby
# Assume
class Data
  has_attribute :key, :real_other_key
  not_an_attribute :no_key
end

# Definition
json_spec :obj do
  key do
    node_name_a
  end
  other_key get: proc {real_other_key} do
    node_name_b
  end
  no_key do 
    node_name_c
  end

end

# Result
data_inst.to_json(spec: :obj) #=> "{
  #"key": {"node_name_a": "#{data_inst.key.node_name_a.to_json}"},
  #"other_key": {"node_name_b": "#{data_inst.real_other_key.node_name_b.to_json}"},
  #"no_key": {"node_name_b": "#{data_inst.node_name_c.to_json}"}

# Schema Result
Data.json_spec_schema(:typed)
  #=> {"id": [... object_schema ...]}  
``` 


### Object Collections
  As with norman nodes, object nodes may be placed in a collection
```ruby
json_spec :obj_arr do
  node_name Array do
    key
  end
end

# Result
data_inst.to_json(spec: :obj_arr) 
  #=> {"node_name": [{"key": #{node1.key.to_json}}, {"key": #{node2.key.to_json}}...]}
```

### Glue Object
  Occasionally you may wish to create an object key for an attribute in the current context.
  This may be useful if the client expects a nested structure for attributes stored in one 
  object.

```ruby
# Assume
class Data
  has_attribute :key_a, :node_name_a
end

# Definition
json_spec :obj_glue do
  key_a get: proc {self} do
    node_name_a
  end
end

# Result
data_inst.to_json(spec: obj) 
#=> "{"key_a": {"node_name_a": "#{data_inst.node_name_a.to_json}"}
```

### Object Extension
  If another json_spec defines some or all of the behaviour needed for a node, that 
  spec can be used as a starting basis for a new spec. The extended pec will provide
  provide all configuration values not otherwise overridden in the new spec.

```ruby
# Definition
json_spec :obj_ovr do
  default! :keep, 1
  default! :replace, 2
end

json_spec :obj_ext do
  extends! :obj_ovr
  replace
end

# Result
data_inst.to_json(spec: :obj_ext) #=> {"keep": 1, "replace": #{data_inst.replace.to_json}}
```

  Object extensions may also be performed of json_specs defined for different objects. 
  This is useful when generating a complex json which requires multiple interacting 
  models.

```ruby
# Assume
class Second
  has_association :first
end

# Definition
class First
  json_spec :obj_ovr_path do
    default! :key, 1
  end
end

class Second
  json_spec :obj_ext_path do
    first do
      extends! First, :obj_ovr_path
    end
  end
end

# Result
second_inst.to_json(spec: :obj_ext_path) #=> {"first": {"key": 1}}
```

***Important***: Each block may be extended only once. New extensions override the prior ones.

##
## Overrides
##
  Sometimes a node_name will be defined in the global ruby function scope. In this situation the
  override method allows direct access to above features using a hard-coded function.

```ruby
# Definition
json_spec :ovr do
  override! :print
end

# Result
data_inst.to_json(spec: :ovr) 
#=> {"print": #{data_inst.print.to_json}
```

***Important***: The node name must be a symbol, or something that may be converted to a symbol.

##
## Conditionals/Execution break-in
##
  Occasionally some nodes may be conditionally required based on some outside criteria.
  For this purpose the json_spec supports operators to break directly into the 
  execution of the spec generator.

  ***Note***: All conditionals are executed within the context of the data_inst.
```ruby
# Definition
json_spec :cond do
  if!(proc {some_check}) do
    key
  end
end

# Result
data_inst.to_json(spec: :cond) 
#=> if data_inst.some_check
  # {"key": #{data_inst.key.jo_json}}
#=> if not data_inst.some_check
  # {}

# Schema Result
  # {"key": "ConditionalBasicType"}

```

## Caveats
  Redefining a node defined previously complete overrides it, even if done in the same 
  context. If complex node behaviour is required, please break it down into a proper
  series of extensions.

## Benchmakrs

JsonSpec is particularly effective when extracting complex information from Rails models and other similarly complex objects.

For the purpose of this benchmark I am exporting a single ActiveRecord result, with the following structure:

```ruby
record.to_json #=> "{\"id\":7,\"name\":\"Test Name\",\"description\":\"Test Description\",\"image\":{\"url\":\"/uploads/68e25af9f4e52ab8a28d47a8ce3c707192ae3a3b.jpg\"},\"created_at\":\"2014-07-24T15:05:16.000Z\",\"updated_at\":\"2014-07-24T15:05:16.000Z\"}" 
```

The Record class was initialized with the following spec:

```ruby 
class Record << ActiveRecord::Base
  prepend Sereth::JsonSpec
  json_spec :all do
    id
    name
    description
    image do
      url
    end
    created_at
    updated_at
  end
end
```

```ruby
# Scenario 1 - Active Support to_json
Benchmark.measure { 10000.times{record.to_json}  } #=> 8.710000   0.150000   8.860000 (  8.876099)

# Scenario 2 - Yajl Ruby
encoder = Yajl::Encoder.new
Benchmark.measure { 10000.times{encoder.encode record}  } #=> 8.890000   0.140000   9.030000 (  9.050803)

# Scenario 3 - JsonSpec
Benchmark.measure { 10000.times{record.to_json(spec: :all)}  } #=> 8.710000   0.150000   8.860000 (  8.876099)

