# Kwalify User's Guide (for Ruby)

makoto kuwata <kwa(at)kuwata-lab.com>  
last update: $Date: 2008-01-28 15:27:12 +0900 (Mon, 28 Jan 2008) $  

## Preface

Kwalify(*1) is a parser, schema validator, and data binding tool for YAML and
JSON. Kwalify enables you to handle YAML and JSON more easily and strictly.

Topics:

  * Schema validation for YAML and JSON
  * Class definition generation for Ruby, PHP, and Java
  * Data binding
  * Preceding alias

(*1)

    Pronounce as 'Qualify'.

### Table of Contents

  * Preface
    * Table of Contents
  * Schema Definition
    * Sequence
    * Mapping
    * Sequence of Mapping
    * Mapping of Sequence
    * Rule and Constraint
    * Unique constraint
  * Tips
    * JSON
    * Anchor and Alias
    * Default of Mapping
    * Merging Mappings
  * How to in Ruby
    * Validation
    * Parsing with Validation
    * Meta Validation
    * Validator#validator_hook()
    * Preceding Alias
    * Data Binding
  * Actions
    * Class Definition Generation
      * Ruby Class Definition
      * Java Class Definition
  * References
    * Usage in Command-Line

  
  

## Schema Definition

This section describes how to define schema definition of YAML.

### Sequence

`schema01.yaml` : sequence of string

    
    
    type:   seq
    sequence:
      - type:   str
    

`document01a.yaml` : valid document example

    
    
    - foo
    - bar
    - baz
    

validate

    
    
    $ kwalify -lf schema01.yaml document01a.yaml
    document01a.yaml#0: valid.
    

`document01b.yaml` : invalid document example

    
    
    - foo
    - 123
    - baz
    

validate

    
    
    $ kwalify -lf schema01.yaml document01b.yaml
    document01b.yaml#0: INVALID
      - (line 2) [/1] '123': not a string.
    

Default '`type:`' is `str` so you can omit '`type: str`'.

  

### Mapping

`schema02.yaml` : mapping of scalar

    
    
    type:       map
    mapping:
     "name":
        type:      str
        required:  yes
     "email":
        type:      str
        pattern:   /@/
     "age":
        type:      int
     "birth":
        type:      date
    

`document02a.yaml` : valid document example

    
    
    name:   foo
    email:  foo@mail.com
    age:    20
    birth:  1985-01-01
    

validate

    
    
    $ kwalify -lf schema02.yaml document02a.yaml
    document02a.yaml#0: valid.
    

`document02b.yaml` : invalid document example

    
    
    name:   foo
    email:  foo(at)mail.com
    age:    twenty
    birth:  Jun 01, 1985
    

validate

    
    
    $ kwalify -lf schema02.yaml document02b.yaml
    document02b.yaml#0: INVALID
      - (line 2) [/email] 'foo(at)mail.com': not matched to pattern /@/.
      - (line 3) [/age] 'twenty': not a integer.
      - (line 4) [/birth] 'Jun 01, 1985': not a date.
    

  

### Sequence of Mapping

`schema03.yaml` : sequence of mapping

    
    
    type:      seq
    sequence:
      - type:      map
        mapping:
         "name":
            type:      str
            required:  true
         "email":
            type:      str
    

`document03a.yaml` : valid document example

    
    
    - name:   foo
      email:  foo@mail.com
    - name:   bar
      email:  bar@mail.net
    - name:   baz
      email:  baz@mail.org
    

validate

    
    
    $ kwalify -lf schema03.yaml document03a.yaml
    document03a.yaml#0: valid.
    

`document03b.yaml` : invalid document example

    
    
    - name:   foo
      email:  foo@mail.com
    - naem:   bar
      email:  bar@mail.net
    - name:   baz
      mail:   baz@mail.org
    

validate

    
    
    $ kwalify -lf schema03.yaml document03b.yaml
    document03b.yaml#0: INVALID
      - (line 3) [/1] key 'name:' is required.
      - (line 3) [/1/naem] key 'naem:' is undefined.
      - (line 6) [/2/mail] key 'mail:' is undefined.
    

  

### Mapping of Sequence

`schema04.yaml` : mapping of sequence of mapping

    
    
    type:      map
    mapping:
     "company":
        type:      str
        required:  yes
     "email":
        type:      str
     "employees":
        type:      seq
        sequence:
          - type:    map
            mapping:
             "code":
                type:      int
                required:  yes
             "name":
                type:      str
                required:  yes
             "email":
                type:      str
    

`document04a.yaml` : valid document example

    
    
    company:    Kuwata lab.
    email:      webmaster@kuwata-lab.com
    employees:
      - code:   101
        name:   foo
        email:  foo@kuwata-lab.com
      - code:   102
        name:   bar
        email:  bar@kuwata-lab.com
    

validate

    
    
    $ kwalify -lf schema04.yaml document04a.yaml
    document04a.yaml#0: valid.
    

`document04b.yaml` : invalid document example

    
    
    company:    Kuwata Lab.
    email:      webmaster@kuwata-lab.com
    employees:
      - code:   A101
        name:   foo
        email:  foo@kuwata-lab.com
      - code:   102
        name:   bar
        mail:   bar@kuwata-lab.com
    

validate

    
    
    $ kwalify -lf schema04.yaml document04b.yaml
    document04b.yaml#0: INVALID
      - (line 4) [/employees/0/code] 'A101': not a integer.
      - (line 9) [/employees/1/mail] key 'mail:' is undefined.
    

  

### Rule and Constraint

`type:`, `required:`, `length`, ... are called **constraint** and set of
constraints are called **rule**.

  * Rule contains 'type:' constraint. If 'type:' is omitted, 'type: str' is used as default. 
  * 'sequence:' constraint takes a sequence of rule (the sequence can contain only a rule). 
  * 'mapping:' constraint takes a mapping which values are rules. 

![constraints and rules of schema definition.](img/fig01.png)

The following is a list of constraints.

**`required:` **

     Value is required when true (default is false). This is similar to not-null constraint in RDBMS. 
**`enum:` **

     List of available values. 
**`pattern:` **

     Specifies regular expression pattern of value. 
**`type:` **

     Type of value. The followings are available: 

  * `str`
  * `int`
  * `float`
  * `number` (== int or float) 
  * `text` (== str or number) 
  * `bool`
  * `date`
  * `time`
  * `timestamp`
  * `seq`
  * `map`
  * `scalar` (all but seq and map) 
  * `any` (means any data) 

**`range:` **

     Range of value between max/max-ex and min/min-ex. 

  * 'max' means 'max-inclusive'. 
  * 'min' means 'min-inclusive'. 
  * 'max-ex' means 'max-exclusive'. 
  * 'min-ex' means 'min-exclusive'. 

Type `seq`, `map`, `bool` and `any` are not available with `range:`.

**`length:` **

     Range of length of value between max/max-ex and min/min-ex. Only type `str` and `text` are available with `length:`. 
**`assert:` **

     String which represents validation expression. String should contain variable name `val` which repsents value. (This is an experimental function and not supported in Kwartz-java). 
**`unique:` **

     Value is unique for mapping or sequence. This is similar to unique constraint of RDBMS. See the next subsection for detail. 
**`name:` **

     Name of schema. 
**`desc:` **

     Description. This is not used for validation. 
**`class:` **

     Class name. This is for data-binding and is available only with type 'map'. This is also used in 'genclass' action. 
**`default:` **

     Default value. This is only for 'genclass' action, and have no effect to validation and parsing. Default value should be scalar and it is not available if `type:` is `map` or `seq`, and also not available when `required:` is true. 

`schema05.yaml` : rule examples

    
    
    type:      seq
    sequence:
      -
        type:      map
        mapping:
         "name":
            type:       str
            required:   yes
         "email":
            type:       str
            required:   yes
            pattern:    /@/
         "password":
            type:       text
            length:     { max: 16, min: 8 }
         "age":
            type:       int
            range:      { max: 30, min: 18 }
            # or assert: 18 <= val && val <= 30
         "blood":
            type:       str
            enum:       [A, B, O, AB]
         "birth":
            type:       date
         "memo":
            type:       any
         "deleted":
            type:       bool
            default:    false
    

`document05a.yaml` : valid document example

    
    
    - name:     foo
      email:    foo@mail.com
      password: xxx123456
      age:      20
      blood:    A
      birth:    1985-01-01
    - name:     bar
      email:    bar@mail.net
      age:      25
      blood:    AB
      birth:    1980-01-01
    

validate

    
    
    $ kwalify -lf schema05.yaml document05a.yaml
    document05a.yaml#0: valid.
    

`document05b.yaml` : invalid document example

    
    
    - name:     foo
      email:    foo(at)mail.com
      password: xxx123
      age:      twenty
      blood:    a
      birth:    1985-01-01
    - given-name:  bar
      family-name: Bar
      email:    bar@mail.net
      age:      15
      blood:    AB
      birth:    1980/01/01
    

validate

    
    
    $ kwalify -lf schema05.yaml document05b.yaml
    document05b.yaml#0: INVALID
      - (line 2) [/0/email] 'foo(at)mail.com': not matched to pattern /@/.
      - (line 3) [/0/password] 'xxx123': too short (length 6 < min 8).
      - (line 4) [/0/age] 'twenty': not a integer.
      - (line 5) [/0/blood] 'a': invalid blood value.
      - (line 7) [/1] key 'name:' is required.
      - (line 7) [/1/given-name] key 'given-name:' is undefined.
      - (line 8) [/1/family-name] key 'family-name:' is undefined.
      - (line 10) [/1/age] '15': too small (< min 18).
      - (line 12) [/1/birth] '1980/01/01': not a date.
    

  

### Unique constraint

'`unique:`' constraint is available with elements of sequence or mapping. This
is equivalent to unique constraint of RDBMS.

  * Type of rule which has '`unique:`' entry must be scalar (str, int, float, ...). 
  * Type of parent rule must be sequence or mapping. 

`schema06.yaml` : unique constraint entry with mapping and sequence

    
    
    type: seq
    sequence:
      - type:     map
        required: yes
        mapping:
         "name":   { type: str, required: yes, **unique: yes** }
         "email":  { type: str }
         "groups":
            type:     seq
            sequence:
              - { type: str, **unique: yes** }
    

`document06a.yaml` : valid document example

    
    
    - name:   foo
      email:  admin@mail.com
      groups:
        - users
        - foo
        - admin
    - name:   bar
      email:  admin@mail.com
      groups:
        - users
        - admin
    - name:   baz
      email:  baz@mail.com
      groups:
        - users
    

validate

    
    
    $ kwalify -lf schema06.yaml document06a.yaml
    document06a.yaml#0: valid.
    

`document06b.yaml` : invalid document example

    
    
    - name:   foo
      email:  admin@mail.com
      groups:
        - foo
        - users
        - admin
        - foo
    - name:   bar
      email:  admin@mail.com
      groups:
        - admin
        - users
    - name:   bar
      email:  baz@mail.com
      groups:
        - users
    

validate

    
    
    $ kwalify -lf schema06.yaml document06b.yaml
    document06b.yaml#0: INVALID
      - (line 7) [/0/groups/3] 'foo': is already used at '/0/groups/0'.
      - (line 13) [/2/name] 'bar': is already used at '/1/name'.
    

  
  

## Tips

### JSON

[JSON](http://www.json.org) is a lightweight data-interchange format,
especially useful for JavaScript. JSON can be considered as a subset of YAML.
It means that YAML parser can parse JSON and Kwalify can validate JSON
document.

`schema12.json` : an example schema definition written in JSON format

    
    
    { "type": "map",
      "required": true,
      "mapping": {
        "name":   { "type": "str", "required": true },
        "email":  { "type": "str" },
        "age":    { "type": "int" },
        "gender": { "type": "str", "enum": ["M", "F"] },
        "favorite": { "type": "seq",
                      "sequence": [ { "type": "str" } ]
                    }
      }
    }
    

`document12a.json` : valid JSON document example

    
    
    { "name": "Foo",
      "email": "foo@mail.com",
      "age": 20,
      "gender": "F",
      "favorite": [
         "football",
         "basketball",
         "baseball"
      ]
    }
    

validate

    
    
    $ kwalify -lf schema12.json document12a.json
    document12a.json#0: valid.
    

`document12b.json` : invalid JSON document example

    
    
    {
      "mail": "foo@mail.com",
      "age": twenty,
      "gender": "X",
      "favorite": [ 123, 456 ]
    }
    

validate

    
    
    $ kwalify -lf schema12.json document12b.json
    document12b.json#0: INVALID
      - (line 1) [/] key 'name:' is required.
      - (line 2) [/mail] key 'mail:' is undefined.
      - (line 3) [/age] 'twenty': not a integer.
      - (line 4) [/gender] 'X': invalid gender value.
      - (line 5) [/favorite/0] '123': not a string.
      - (line 5) [/favorite/1] '456': not a string.
    

  

### Anchor and Alias

You can share rules by YAML anchor and alias.

`schema13.yaml` : anchor example

    
    
    type:   seq
    sequence:
      - **& employee**
        type:      map
        mapping:
         "given-name": **& name**
            type:     str
            required: yes
         "family-name": ***name**
         "post":
            type: str
            enum: [exective, manager, clerk]
         "supervisor":  ***employee**
    

Anchor and alias is also available in YAML document.

`document13a.yaml` : valid document example

    
    
    - **& foo**
      given-name:    foo
      family-name:   Foo
      post:          exective
    - **& bar**
      given-name:    bar
      family-name:   Bar
      post:          manager
      supervisor:    ***foo**
    - given-name:    baz
      family-name:   Baz
      post:          clerk
      supervisor:    ***bar**
    - given-name:    zak
      family-name:   Zak
      post:          clerk
      supervisor:    ***bar**
    

validate

    
    
    $ kwalify -lf schema13.yaml document13a.yaml
    document13a.yaml#0: valid.
    

  

### Default of Mapping

YAML allows user to specify default value of mapping.

For example, the following YAML document uses default value of mapping.

    
    
    A: 10
    B: 20
    **=: -1**      # default value
    

This is equal to the following Ruby code.

    
    
    map = ["A"=>10, "B"=>20]
    map.default = -1
    map
    

Kwalify allows user to specify default rule using default value of mapping. It
is useful when key names are unknown.

`schema14.yaml` : default rule example

    
    
    type: map
    mapping:
      **=:**              # default rule
        type: number
        range: { max: 1, min: -1 }
    

`document14a.yaml` : valid document example

    
    
    value1: 0
    value2: 0.5
    value3: -0.9
    

validate

    
    
    $ kwalify -lf schema14.yaml document14a.yaml
    document14a.yaml#0: valid.
    

`document14b.yaml` : invalid document example

    
    
    value1: 0
    value2: 1.1
    value3: -2.0
    

validate

    
    
    $ kwalify -lf schema14.yaml document14b.yaml
    document14b.yaml#0: INVALID
      - (line 2) [/value2] '1.1': too large (> max 1).
      - (line 3) [/value3] '-2.0': too small (< min -1).
    

  

### Merging Mappings

YAML allows user to merge mappings.

    
    
    - **& a1**
      A: 10
      B: 20
    - **< <: *a1**            # merge
      A: 15              # override
      C: 30              # add
    

This is equal to the following Ruby code.

    
    
    a1 = {"A"=>10, "B"=>20}
    tmp = {}
    tmp.update(a1)       # merge
    tmp["A"] = 15        # override
    tmp["C"] = 30        # add
    

This feature allows Kwalify to merge rule entries.

`schema15.yaml` : merging rule entries example

    
    
    type: map
    mapping:
     "group":
        type: map
        mapping:
         "name": **& name**
            type: str
            required: yes
         "email": **& email**
            type: str
            pattern: /@/
            required: no
     "user":
        type: map
        mapping:
         "name":
            **< <: *name**             # merge
            **length: { max: 16 }**   # add
         "email":
            **< <: *email**            # merge
            **required: yes**         # override
    

`document15a.yaml` : valid document example

    
    
    group:
      name: foo
      email: foo@mail.com
    user:
      name: bar
      email: bar@mail.com
    

validate

    
    
    $ kwalify -lf schema15.yaml document15a.yaml
    document15a.yaml#0: valid.
    

`document15b.yaml` : invalid document example

    
    
    group:
      name: foo
      email: foo@mail.com
    user:
      name: toooooo-looooong-name
    

validate

    
    
    $ kwalify -lf schema15.yaml document15b.yaml
    document15b.yaml#0: INVALID
      - (line 5) [/user] key 'email:' is required.
      - (line 5) [/user/name] 'toooooo-looooong-name': too long (length 21 > max 16).
    

  
  

## How to in Ruby

This section describes how to use Kwalify in Ruby.

### Validation

    
    
    require 'kwalify'
    #require 'yaml'
    
    ## load schema data
    schema = Kwalify::Yaml.load_file('schema.yaml')
    ## or
    #schema = YAML.load_file('schema.yaml')
    
    ## create validator
    validator = Kwalify::Validator.new(schema)
    
    ## load document
    document = Kwalify::Yaml.load_file('document.yaml')
    ## or
    #document = YAML.load_file('document.yaml')
    
    ## validate
    errors = validator.validate(document)
    
    ## show errors
    if errors && !errors.empty?
      for e in errors
        puts "[#{e.path}] #{e.message}"
      end
    end
    

  

### Parsing with Validation

From version 0.7, Kwalify supports parsing with validation.

    
    
    require 'kwalify'
    #require 'yaml'
    
    ## load schema data
    schema = Kwalify::Yaml.load_file('schema.yaml')
    ## or
    #schema = YAML.load_file('schema.yaml')
    
    ## create validator
    validator = Kwalify::Validator.new(schema)
    
    ## create parser with validator
    ## (if validator is ommitted, no validation executed.)
    parser = Kwalify:::Yaml::Parser.new(validator)
    
    ## parse document with validation
    filename = 'document.yaml'
    document = parser.parse_file(filename)
    ## or
    #document = parser.parse(File.read(filename), filename)
    
    ## show errors if exist
    errors = parser.errors()
    if errors && !errors.empty?
      for e in errors
        puts "#{e.linenum}:#{e.column} [#{e.path}] #{e.message}"
      end
    end
    

  

### Meta Validation

Meta validator is a validator which validates schema definition. The schema
definition is placed at 'kwalify/kwalify.schema.yaml'.

Kwalify also provides Kwalify::MetaValidator class which validates schema
defition.

    
    
    require 'kwalify'
    
    ## meta validator
    metavalidator = Kwalify::MetaValidator.instance
    
    ## validate schema definition
    parser = Kwalify::Yaml::Parser.new(metavalidator)
    errors = parser.parse_file('schema.yaml')
    for e in errors
      puts "#{e.linenum}:#{e.column} [#{e.path}] #{e.message}"
    end if errors && !errors.empty?
    

Meta validation is also available with command-line option '-m'.

    
    
    $ kwalify -m schema1.yaml schema2.yaml ...
    

  

### Validator#validator_hook()

You can extend Kwalify::Validator and override
Kwalify::Validator#validator_hook() method. This method is called by
Kwalify::Validator#validate().

answers-schema.yaml : 'name:' is important.

    
    
    type:      map
    mapping:
     "answers":
        type:      seq
        sequence:
          - type:      map
            **name:        Answer**
            mapping:
             "name":   { type: str, required: yes }
             "answer": { type: str, required: yes,
                         enum: [good, not bad, bad] }
             "reason": { type: str }
    

answers-validator.rb : validate script for Ruby

    
    
    #!/usr/bin/env ruby
    
    require 'kwalify'
    
    ## validator class for answers
    class AnswersValidator < Kwalify::Validator
    
       ## load schema definition
       @@schema = Kwalify::Yaml.load_file('answers-schema.yaml')
       ## or
       ##   require 'yaml'
       ##   @@schema = YAML.load_file('answers-schema.yaml')
    
       def initialize()
          super(@@schema)
       end
    
       ## hook method called by Validator#validate()
       **def validate_hook(value, rule, path, errors)**
          **case rule.name**
          **when 'Answer'**
             if value['answer'] == 'bad'
                reason = value['reason']
                if !reason || reason.empty?
                   msg = "reason is required when answer is 'bad'."
                   errors  << Kwalify::ValidationError.new(msg, path)
                end
             end
          end
       end
    
    end
    
    ## create validator
    validator = AnswersValidator.new
    
    ## parse and validate YAML document
    input = ARGF.read()
    parser = Kwalify::Yaml::Parser.new(validator)
    document = parser.parse(input)
    
    ## show errors
    errors = parser.errors()
    if !errors || errors.empty?
       puts "Valid."
    else
       puts "*** INVALID!"
       for e in errors
          # e.class == Kwalify::ValidationError
          puts "#{e.linenum}:#{e.column} [#{e.path}] #{e.message}"
       end
    end
    

`document07a.yaml` : valid document example

    
    
    answers:
      - name:      Foo
        answer:    good
        reason:    I like this style.
      - name:      Bar
        answer:    not bad
      - name:      Baz
        answer:    bad
        reason:    I don't like this style.
    

validate

    
    
    $ ruby answers-validator.rb document07a.yaml
    Valid.
    

`document07b.yaml` : invalid document example

    
    
    answers:
      - name:    Foo
        answer:  good
      - name:    Bar
        answer:  bad
      - name:    Baz
        answer:  not bad
    

validate

    
    
    $ ruby answers-validator.rb document07b.yaml
    *** INVALID!
    4:3 [/answers/1] reason is required when answer is 'bad'.
    

You can validate some document by a Validator instance because Validator class
and Validator#validate() method are stateless. If you use instance variables
in custom validator_hook() method, it becomes to be stateful.

  

### Preceding Alias

From version 0.7, Kwalify allows aliases to appear before corresponding
anchors are now appeared. These aliases are called as 'preceding alias'.

howto3.yaml

    
    
    - name: Foo
      parent: *bar        # preceding alias
    - &bar
      name: Bar
      parent: *baz        # preceding alias
    - &baz
      name: Baz
      parent: null
    

To enable preceding alias, set Kwalify::Yaml::Parser#preceding_alias to true.

howto3.rb

    
    
    require 'kwalify'
    parser = Kwalify::Yaml::Parser.new
    **parser.preceding_alias = true**   # enable preceding alias
    ydoc = parser.parse_file('howto3.yaml')
    require 'pp'
    pp ydoc
    

result

    
    
    $ ruby howto3.rb
    [{"name"=>"Foo",
      "parent"=>{"name"=>"Bar", "parent"=>{"name"=>"Baz", "parent"=>nil}}},
     {"name"=>"Bar", "parent"=>{"name"=>"Baz", "parent"=>nil}},
     {"name"=>"Baz", "parent"=>nil}]
    

Command-line option '-P' also enables preceding alias.

Preceding alias is very useful when document is complex.

  

### Data Binding

From version 0.7, Kwalify supports data binding. * To enable data binding, set
Kwlaify::Yaml::Parser#data_binding to true. * It is required to specify class
name in schema definition. (Notice that 'class:' constraint is avaialbe only
with rule which type is 'map'.) * Also instance methods '[]', '[]=', and
'keys?' must be defined in the classes. (Including Kwalify::Util::HashLike
modules is easy way to define them.)

config.schema.yaml: schema definition file

    
    
    type:  map
    **class: Config**
    mapping:
     "host": { type: str, required: true }
     "port": { type: int }
     "user": { type: str, required: true }
     "pass": { type: str, required: true }
    

config.yaml: data file

    
    
    host:  localhost
    port:  8080
    user:  user1
    pass:  password1
    

loadconfig.rb: ruby program

    
    
    ## class definition
    require 'kwalify/util/hashlike'
    **class Config**
      **include Kwalify::Util::HashLike**  # defines [], []=, and keys?
      attr_accessor :host, :posrt, :user, :pass
    **end**
    ## create validator object
    require 'kwalify'
    schema = Kwalify::Yaml.load_file('config.schema.yaml')
    validator = Kwalify::Validator.new(schema)
    ## parse configuration file with data binding
    parser = Kwalify::Yaml::Parser.new(validator)
    **parser.data_binding = true**    # enable data binding
    config = parser.parse_file('config.yaml')
    require 'pp'
    pp config
    

result

    
    
    $ ruby loadconfig.rb
    #<Config:
     @host="localhost",
     @pass="password1",
     @port=8080,
     @user="user1">
    

Data binding is available even when data is more complex. Preceding alias is
also available.

For example, the following data is complex because it uses anchor and alias
(including preceding alias).

BABEL.data.yaml

    
    
    teams:
      - &thechildren
        name:   The Children
        desc:   Level 7 ESPers
        chief:  *minamoto                  # preceding alias
        members:  [*kaoru, *aoi, *shiho]   # preceding aliases
    
    members:
      - &minamoto
        name:   Kohichi Minamoto
        desc:   Scientist
        team:   *thechildren
      - &kaoru
        name:   Kaoru Akashi
        desc:   Psychokino
        team:   *thechildren
      - &aoi
        name:   Aoi Nogami
        desc:   Teleporter
        team:   *thechildren
      - &shiho
        name:   Shiho Sannomiya
        desc:   Psycometrer
        team:   *thechildren
    

Here is the schema definition. (Notice that 'class:' constraint is avaialbe
only with rule which type is 'map'.)

BABEL.schema.yaml

    
    
    type: map
    required: yes
    mapping:
     "teams":
        type: seq
        required: yes
        sequence:
          - &team
            type:  map
            required: yes
            **class:  Team**
            mapping:
             "name":  {type: str, required: yes, unique: yes}
             "desc":  {type: str}
             "chief":  *member       # preceding alias
             "members":
                type: seq
                sequence: [*member]  # preceding alias
     "members":
        type: seq
        required: yes
        sequence:
          -  &member
            type:  map
            required: yes
            **class:  Member**
            mapping:
             "name":  {type: str, required: yes, unique: yes}
             "desc":  {type: str}
             "team":  *team
    

It is required to define class 'Team' and 'Member' for data-binding. Command-
line option '-a genclass-ruby' will help you to generate class definitions
from schema definition. Try 'kwalify -ha genclass-ruby' for more details about
'genclass-ruby' action.

    
    
    $ kwalify -a genclass-ruby -P -f BABEL.schema.yaml \
        --hashlike --initialize=false --module=Babel
    require 'kwalify/util/hashlike'
    
    module Babel
    
      ## 
      class Team
        include Kwalify::Util::HashLike
        attr_accessor :name             # str
        attr_accessor :desc             # str
        attr_accessor :chief            # map
        attr_accessor :members          # seq
      end
    
      ## 
      class Member
        include Kwalify::Util::HashLike
        attr_accessor :name             # str
        attr_accessor :desc             # str
        attr_accessor :team             # map
      end
    
    end
    $ kwalify -a genclass-ruby -P -f BABEL.schema.yaml  \
        --hashlike --initialize=false --module=Babel > models.rb
    

Here is the ruby program.

loadbabel.rb

    
    
    require 'kwalify'
    **require 'models'**
    
    ## load schema definition
    schema = Kwalify::Yaml.load_file('BABEL.schema.yaml',
                                     :untabify= >true,
                                     :preceding_alias=>true)
    
    ## add module name to 'class:'
    Kwalify::Util.traverse_schema(schema) do |rulehash|
      if rulehash['class']
        rulehash['class'] = 'Babel::' + rulehash['class']
      end
    end
    
    ## create validator
    validator = Kwalify::Validator.new(schema)
    
    ## parse with data-binding
    parser = Kwalify::Yaml::Parser.new(validator)
    parser.preceding_alias = true
    **parser.data_binding = true**
    ydoc = parser.parse_file('BABEL.data.yaml', :untabify= >true)
    
    ## show document
    require 'pp'
    pp ydoc
    

result

    
    
    $ ruby loadbabel.rb
    {"teams"=>
      [#<Babel::Team:0x53e0f8
        @chief=
         #<Babel::Member:0x53d5e0
          @desc="Scientist",
          @name="Kohichi Minamoto",
          @team=#<Babel::Team:0x53e0f8 ...>>,
        @desc="Level 7 ESPers",
        @members=
         [#<Babel::Member:0x53d018
           @desc="Psychokino",
           @name="Kaoru Akashi",
           @team=#<Babel::Team:0x53e0f8 ...>>,
          #<Babel::Member:0x53ca50
           @desc="Teleporter",
           @name="Aoi Nogami",
           @team=#<Babel::Team:0x53e0f8 ...>>,
          #<Babel::Member:0x53c488
           @desc="Psycometrer",
           @name="Shiho Sannomiya",
           @team=#<Babel::Team:0x53e0f8 ...>>],
        @name="The Children">],
     "members"=>
      [#<Babel::Member:0x53d5e0
        @desc="Scientist",
        @name="Kohichi Minamoto",
        @team=
         #<Babel::Team:0x53e0f8
          @chief=#<Babel::Member:0x53d5e0 ...>,
          @desc="Level 7 ESPers",
          @members=
           [#<Babel::Member:0x53d018
             @desc="Psychokino",
             @name="Kaoru Akashi",
             @team=#<Babel::Team:0x53e0f8 ...>>,
            #<Babel::Member:0x53ca50
             @desc="Teleporter",
             @name="Aoi Nogami",
             @team=#<Babel::Team:0x53e0f8 ...>>,
            #<Babel::Member:0x53c488
             @desc="Psycometrer",
             @name="Shiho Sannomiya",
             @team=#<Babel::Team:0x53e0f8 ...>>],
          @name="The Children">>,
       #<Babel::Member:0x53d018
        @desc="Psychokino",
        @name="Kaoru Akashi",
        @team=
         #<Babel::Team:0x53e0f8
          @chief=
           #<Babel::Member:0x53d5e0
            @desc="Scientist",
            @name="Kohichi Minamoto",
            @team=#<Babel::Team:0x53e0f8 ...>>,
          @desc="Level 7 ESPers",
          @members=
           [#<Babel::Member:0x53d018 ...>,
            #<Babel::Member:0x53ca50
             @desc="Teleporter",
             @name="Aoi Nogami",
             @team=#<Babel::Team:0x53e0f8 ...>>,
            #<Babel::Member:0x53c488
             @desc="Psycometrer",
             @name="Shiho Sannomiya",
             @team=#<Babel::Team:0x53e0f8 ...>>],
          @name="The Children">>,
       #<Babel::Member:0x53ca50
        @desc="Teleporter",
        @name="Aoi Nogami",
        @team=
         #<Babel::Team:0x53e0f8
          @chief=
           #<Babel::Member:0x53d5e0
            @desc="Scientist",
            @name="Kohichi Minamoto",
            @team=#<Babel::Team:0x53e0f8 ...>>,
          @desc="Level 7 ESPers",
          @members=
           [#<Babel::Member:0x53d018
             @desc="Psychokino",
             @name="Kaoru Akashi",
             @team=#<Babel::Team:0x53e0f8 ...>>,
            #<Babel::Member:0x53ca50 ...>,
            #<Babel::Member:0x53c488
             @desc="Psycometrer",
             @name="Shiho Sannomiya",
             @team=#<Babel::Team:0x53e0f8 ...>>],
          @name="The Children">>,
       #<Babel::Member:0x53c488
        @desc="Psycometrer",
        @name="Shiho Sannomiya",
        @team=
         #<Babel::Team:0x53e0f8
          @chief=
           #<Babel::Member:0x53d5e0
            @desc="Scientist",
            @name="Kohichi Minamoto",
            @team=#<Babel::Team:0x53e0f8 ...>>,
          @desc="Level 7 ESPers",
          @members=
           [#<Babel::Member:0x53d018
             @desc="Psychokino",
             @name="Kaoru Akashi",
             @team=#<Babel::Team:0x53e0f8 ...>>,
            #<Babel::Member:0x53ca50
             @desc="Teleporter",
             @name="Aoi Nogami",
             @team=#<Babel::Team:0x53e0f8 ...>>,
            #<Babel::Member:0x53c488 ...>],
          @name="The Children">>]}
    

  
  

## Actions

Kwalify has the command-line '-a _action_ ' which perform a certain action to
schema definition. Currently only the following actions are provided.

genclass-ruby

    

Generate class definitions in Ruby.

genclass-java

    

Generate class definitions in Java.

genclass-php

    

Generate class definitions in Ruby.

In fact action name represents template filename. For example, action
'genclass-ruby' invokes template file 'kwalify/templates/genclass-ruby.eruby'.

Each action can accept some command-line properties. For example, action
'genclass-ruby' can accept the command-line properties '--module= _name_ ', '
--parent= _name_ ', and so on. Type 'kwalify -h -a _action_ ' to show the list
of command-line properties the action can accept.

It is able to add your on action template file. The command-line option '-I'
(template path) will help you.

### Class Definition Generation

Command-line option '-a genclass-ruby' or '-a genclass-java' generates class
definition automatically from schema definition in Ruby or Java.

Assume the following data file and schema definition.

`address_book.yaml` : data file

    
    
    groups:
    
      - name:   family
        desc:   my family
    
      - name:   friend
        desc:   my friends
    
      - name:   business
        desc:   those who works together
    
    people:
    
      - name:   Sumire
        group:  family
        birth:  2000-01-01
        blood:  A
    
      - name:   Shiina
        group:  friend
        birth:  1995-01-01
        email:  shiina@mail.org
    
      - name:   Sakura
        group:  business
        email:  cherry@mail.net
        phone:  012-345-6789
    

`address_book.schema.yaml` : schema definition file

    
    
    type:  map
    **class:  AddressBook**
    desc:  address-book class
    mapping:
     "groups":
        type:  seq
        sequence:
          - type:  map
            **class:  Group**
            desc:  group class
            mapping:
             "name":  { type: str,  required: yes }
             "desc":  { type: str }
     "people":
        type:  seq
        sequence:
          - type:  map
            **class:  Person**
            desc:  person class
            mapping:
             "name":  { type: str, required: yes }
             "desc":  { type: str }
             "group": { type: str }
             "email": { type: str, pattern: '/@/' }
             "phone": { type: str }
             "birth": { type: date }
             "blood": { type: str, enum: [A, B, O, AB] }
             "deleted": { type: bool, **default: false** }
    

#### Ruby Class Definition

generate class definition

    
    
    $ kwalify **-a genclass-ruby** -tf address_book.schema.yaml  > address_book.rb
    

`address_book.rb` : generated class definition

    
    
    ## address-book class
    class AddressBook
      def initialize(hash=nil)
        if hash.nil?
          return
        end
        @groups           = (v=hash['groups']) ? v.map!{|e| e.is_a?(Group) ? e : Group.new(e)} : v
        @people           = (v=hash['people']) ? v.map!{|e| e.is_a?(Person) ? e : Person.new(e)} : v
      end
      attr_accessor :groups           # seq
      attr_accessor :people           # seq
    end
    
    ## group class
    class Group
      def initialize(hash=nil)
        if hash.nil?
          return
        end
        @name             = hash['name']
        @desc             = hash['desc']
      end
      attr_accessor :name             # str
      attr_accessor :desc             # str
    end
    
    ## person class
    class Person
      def initialize(hash=nil)
        if hash.nil?
          @deleted          = false
          return
        end
        @name             = hash['name']
        @desc             = hash['desc']
        @group            = hash['group']
        @email            = hash['email']
        @phone            = hash['phone']
        @birth            = hash['birth']
        @blood            = hash['blood']
        @deleted          = (v=hash['deleted']).nil? ? false : v
      end
      attr_accessor :name             # str
      attr_accessor :desc             # str
      attr_accessor :group            # str
      attr_accessor :email            # str
      attr_accessor :phone            # str
      attr_accessor :birth            # date
      attr_accessor :blood            # str
      attr_accessor :deleted          # bool
      def deleted?      ;  @deleted      ; end
    end
    

`example_address_book.rb` : example code of using address-book.rb

    
    
    require 'address_book'
    require 'yaml'
    require 'pp'
    
    str = File.read('address_book.yaml')
    ydoc = YAML.load(str)
    **addrbook = AddressBook.new(ydoc)**
    
    pp **addrbook.groups**
    pp **addrbook.people**
    

result

    
    
    $ ruby example_address_book.rb
    [#<Group:0xddf24 @desc="my family", @name="family">,
     #<Group:0xddf10 @desc="my friends", @name="friend">,
     #<Group:0xdde84 @desc="those who works together", @name="business">]
    [#<Person:0xdefdc
      @birth=#<Date: 4903089/2,0,2299161>,
      @blood="A",
      @deleted=false,
      @desc=nil,
      @email=nil,
      @group="family",
      @name="Sumire",
      @phone=nil>,
     #<Person:0xdee9c
      @birth=#<Date: 4899437/2,0,2299161>,
      @blood=nil,
      @deleted=false,
      @desc=nil,
      @email="shiina@mail.org",
      @group="friend",
      @name="Shiina",
      @phone=nil>,
     #<Person:0xde8e8
      @birth=nil,
      @blood=nil,
      @deleted=false,
      @desc=nil,
      @email="cherry@mail.net",
      @group="business",
      @name="Sakura",
      @phone="012-345-6789">]
    

Command-line option '`-h -a genclass-ruby`' shows the commpand-line properties
that template can accept.

show command-line properties

    
    
    $ kwalify -ha genclass-ruby
      --module=name   :  module name in which class defined
      --parent=name   :  parent class name
      --include=name  :  module name which all classes include
      --initialize=false :  not print initialize() method
      --hashlike      :  include Kwalify::Util::HashLike module
    

example of command-line properties

    
    
    $ kwalify -a genclass-ruby --module=My --hashlike
    

If command-line property '--hashlike' (== '--hashlike=true') is specified,
module Kwalify::Util::HashLike is included for each classes generated. That
module is defined in 'kwalify/util/hashlike.rb'

  

#### Java Class Definition

generate java class definition

    
    
    $ kwalify **-a genclass-java** -tf address_book.schema.yaml
    generating ./AddressBook.java...done.
    generating ./Group.java...done.
    generating ./Person.java...done.
    

`AddressBook.java` : generated class definition

    
    
    // generated by kwalify from address_book.schema.yaml
    
    import java.util.*;
    
    /**
     *  address-book class
     */
    public class AddressBook {
    
        private List _groups;
        private List _people;
    
        public AddressBook() {}
    
        public AddressBook(Map map) {
            List seq;
            Object obj;
            if ((seq = (List)map.get("groups")) != null) {
                for (int i = 0; i < seq.size(); i++) {
                    if ((obj = seq.get(i)) instanceof Map) {
                        seq.set(i, new Group((Map)obj));
                    }
                }
            }
            _groups       = seq;
            if ((seq = (List)map.get("people")) != null) {
                for (int i = 0; i < seq.size(); i++) {
                    if ((obj = seq.get(i)) instanceof Map) {
                        seq.set(i, new Person((Map)obj));
                    }
                }
            }
            _people       = seq;
        }
    
        public List getGroups() { return _groups; }
        public void setGroups(List groups_) { _groups = groups_; }
        public List getPeople() { return _people; }
        public void setPeople(List people_) { _people = people_; }
    }
    

`Group.java` : generated class definition

    
    
    // generated by kwalify from address_book.schema.yaml
    
    import java.util.*;
    
    /**
     *  group class
     */
    public class Group {
    
        private String _name;
        private String _desc;
    
        public Group() {}
    
        public Group(Map map) {
            _name         = (String)map.get("name");
            _desc         = (String)map.get("desc");
        }
    
        public String getName() { return _name; }
        public void setName(String name_) { _name = name_; }
        public String getDesc() { return _desc; }
        public void setDesc(String desc_) { _desc = desc_; }
    }
    

`Person.java` : generated class definition

    
    
    // generated by kwalify from address_book.schema.yaml
    
    import java.util.*;
    
    /**
     *  person class
     */
    public class Person {
    
        private String _name;
        private String _desc;
        private String _group;
        private String _email;
        private String _phone;
        private Date _birth;
        private String _blood;
    
        public Person() {}
    
        public Person(Map map) {
            _name         = (String)map.get("name");
            _desc         = (String)map.get("desc");
            _group        = (String)map.get("group");
            _email        = (String)map.get("email");
            _phone        = (String)map.get("phone");
            _birth        = (Date)map.get("birth");
            _blood        = (String)map.get("blood");
        }
    
        public String getName() { return _name; }
        public void setName(String name_) { _name = name_; }
        public String getDesc() { return _desc; }
        public void setDesc(String desc_) { _desc = desc_; }
        public String getGroup() { return _group; }
        public void setGroup(String group_) { _group = group_; }
        public String getEmail() { return _email; }
        public void setEmail(String email_) { _email = email_; }
        public String getPhone() { return _phone; }
        public void setPhone(String phone_) { _phone = phone_; }
        public Date getBirth() { return _birth; }
        public void setBirth(Date birth_) { _birth = birth_; }
        public String getBlood() { return _blood; }
        public void setBlood(String blood_) { _blood = blood_; }
    }
    

`ExampleAddressBook.java` : example code of using *.java

    
    
    import java.util.*;
    import kwalify.*;
    
    public class ExampleAddressBook {
        public static void main(String args[]) throws Exception {
            // read schema
            String schema_str = Util.readFile("address_book.schema.yaml");
            schema_str = Util.untabify(schema_str);
            Object schema = new YamlParser(schema_str).parse();
    
            // read document file
            String document_str = Util.readFile("address_book.yaml");
            document_str = Util.untabify(document_str);
            YamlParser parser = new YamlParser(document_str);
            Object document = parser.parse();
    
            // create address book object
            AddressBook addrbook = new AddressBook((Map)document);
    
            // show groups
            List groups = addrbook.getGroups();
            if (groups != null) {
                for (Iterator it = groups.iterator(); it.hasNext(); ) {
                    Group group = (Group)it.next();
                    System.out.println("group name: " + group.getName());
                    System.out.println("group desc: " + group.getDesc());
                    System.out.println();
                }
            }
    
            // show people
            List people = addrbook.getPeople();
            if (people != null) {
                for (Iterator it = people.iterator(); it.hasNext(); ) {
                    Person person = (Person)it.next();
                    System.out.println("person name:  " + person.getName());
                    System.out.println("person group: " + person.getGroup());
                    System.out.println("person email: " + person.getEmail());
                    System.out.println("person phone: " + person.getPhone());
                    System.out.println("person blood: " + person.getBlood());
                    System.out.println("person birth: " + person.getBirth());
                    System.out.println();
                }
            }
        }
    
    }
    

result

    
    
    $ javac -classpath '.:kwalify.jar' *.java
    $ java  -classpath '.:kwalify.jar' ExampleAddressBook
    group name: family
    group desc: my family
    
    group name: friend
    group desc: my friends
    
    group name: business
    group desc: those who works together
    
    person name:  Sumire
    person group: family
    person email: null
    person phone: null
    person blood: A
    person birth: Tue Feb 01 00:00:00 JST 2000
    
    person name:  Shiina
    person group: friend
    person email: shiina@mail.org
    person phone: null
    person blood: null
    person birth: Wed Feb 01 00:00:00 JST 1995
    
    person name:  Sakura
    person group: business
    person email: cherry@mail.net
    person phone: 012-345-6789
    person blood: null
    person birth: null
    
    

Command-line option '`-h -a genclass-java`' shows the commpand-line properties
that template can accept.

show command-line properties

    
    
    $ kwalify -ha genclass-java
      --package=name        :  package name
      --extends=name        :  class name to extend
      --implements=name,... :  interface names to implement
      --dir=path            :  directory to locate output file
      --basedir=path        :  base directory to locate output file
      --constructor=false   :  not print initialize() method
    

example of command-line properties

    
    
    $ kwalify -a genclass-java --package=com.example.my --implements=Serializable --basedir=src
    

  
  
  

## References

### Usage in Command-Line

    
    
    ### usage1: validate YAML document in command-line
    $ kwalify -f schema.yaml document.yaml [document2.yaml ...]
    ### usage2: validate schema definition in command-line
    $ kwalify -m schema.yaml [schema2.yaml ...]
    

Command-line options:

**`-h`, `--help` **

     Print help message. 
**`-v` **

     Print version. 
**`-q` **

     Quiet mode. 
**`-s` **

     (Obsolete. Use '-q' instead.) Silent mode. 
**`-f _schema.yaml_` **

     Specify schema definition file. 
**`-m` **

     Meta-validation of schema definition. 
**`-t` **

     Expand tab characters to spaces automatically. 
**`-l` **

     Show linenumber on which error found. 
**`-E` **

     Show errors in Emacs-compatible style (implies '-l' option). 
**`-a action` **

     Do action. Currently supported action is 'genclass-ruby' and 'genclass-java'. Try '-ha action' to get help about the action. 
**`-I path1,path2,...` **

     Template path (for '-a'). 
**`-P` **

     Enable preceding alias. 
  
 
