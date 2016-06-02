# JSON implementation for Ruby {<img src="https://secure.travis-ci.org/flori/json.png" />}[http://travis-ci.org/flori/json]

## Description

This is a implementation of the JSON specification according to RFC 4627
http://www.ietf.org/rfc/rfc4627.txt . Starting from version 1.0.0 on there
will be two variants available:

* A pure ruby variant, that relies on the iconv and the stringscan
  extensions, which are both part of the ruby standard library.
* The quite a bit faster native extension variant, which is in parts
  implemented in C or Java and comes with its own unicode conversion
  functions and a parser generated by the ragel state machine compiler
  http://www.complang.org/ragel/ .

Both variants of the JSON generator generate UTF-8 character sequences by
default. If an :ascii\_only option with a true value is given, they escape all
non-ASCII and control characters with \uXXXX escape sequences, and support
UTF-16 surrogate pairs in order to be able to generate the whole range of
unicode code points.

All strings, that are to be encoded as JSON strings, should be UTF-8 byte
sequences on the Ruby side. To encode raw binary strings, that aren't UTF-8
encoded, please use the to\_json\_raw\_object method of String (which produces
an object, that contains a byte array) and decode the result on the receiving
endpoint.

## Installation

It's recommended to use the extension variant of JSON, because it's faster than
the pure ruby variant. If you cannot build it on your system, you can settle
for the latter.

Just type into the command line as root:

  # rake install

The above command will build the extensions and install them on your system.

  # rake install_pure

or

  # ruby install.rb

will just install the pure ruby implementation of JSON.

If you use Rubygems you can type

  # gem install json

instead, to install the newest JSON version.

There is also a pure ruby json only variant of the gem, that can be installed
with:

  # gem install json_pure

## Compiling the extensions yourself

If you want to create the parser.c file from its parser.rl file or draw nice
graphviz images of the state machines, you need ragel from:
http://www.complang.org/ragel/

## Usage

To use JSON you can
  require 'json'
to load the installed variant (either the extension 'json' or the pure
variant 'json\_pure'). If you have installed the extension variant, you can
pick either the extension variant or the pure variant by typing
  require 'json/ext'
or
  require 'json/pure'

Now you can parse a JSON document into a ruby data structure by calling

  JSON.parse(document)

If you want to generate a JSON document from a ruby data structure call
  JSON.generate(data)

You can also use the pretty\_generate method (which formats the output more
verbosely and nicely) or fast\_generate (which doesn't do any of the security
checks generate performs, e. g. nesting deepness checks).

There are also the JSON and JSON[] methods which use parse on a String or
generate a JSON document from an array or hash:

  document = JSON 'test'  => 23 # => "{\"test\":23}"
  document = JSON['test'] => 23 # => "{\"test\":23}"

and

  data = JSON '{"test":23}'  # => {"test"=>23}
  data = JSON['{"test":23}'] # => {"test"=>23}

You can choose to load a set of common additions to ruby core's objects if
you
  require 'json/add/core'

After requiring this you can, e. g., serialise/deserialise Ruby ranges:

  JSON JSON(1..10) # => 1..10

To find out how to add JSON support to other or your own classes, read the
section "More Examples" below.

To get the best compatibility to rails' JSON implementation, you can
  require 'json/add/rails'

Both of the additions attempt to require 'json' (like above) first, if it has
not been required yet.

## More Examples

To create a JSON document from a ruby data structure, you can call
JSON.generate like that:

 json = JSON.generate [1, 2, {"a"=>3.141}, false, true, nil, 4..10]
 # => "[1,2,{\"a\":3.141},false,true,null,\"4..10\"]"

To get back a ruby data structure from a JSON document, you have to call
JSON.parse on it:

 JSON.parse json
 # => [1, 2, {"a"=>3.141}, false, true, nil, "4..10"]

Note, that the range from the original data structure is a simple
string now. The reason for this is, that JSON doesn't support ranges
or arbitrary classes. In this case the json library falls back to call
Object#to\_json, which is the same as #to\_s.to\_json.

It's possible to add JSON support serialization to arbitrary classes by
simply implementing a more specialized version of the #to\_json method, that
should return a JSON object (a hash converted to JSON with #to\_json) like
this (don't forget the *a for all the arguments):

 class Range
   def to_json(*a)
     {
       'json_class'   => self.class.name, # = 'Range'
       'data'         => [ first, last, exclude_end? ]
     }.to_json(*a)
   end
 end

The hash key 'json\_class' is the class, that will be asked to deserialise the
JSON representation later. In this case it's 'Range', but any namespace of
the form 'A::B' or '::A::B' will do. All other keys are arbitrary and can be
used to store the necessary data to configure the object to be deserialised.

If a the key 'json\_class' is found in a JSON object, the JSON parser checks
if the given class responds to the json\_create class method. If so, it is
called with the JSON object converted to a Ruby hash. So a range can
be deserialised by implementing Range.json\_create like this:

 class Range
   def self.json_create(o)
     new(*o['data'])
   end
 end

Now it possible to serialise/deserialise ranges as well:

 json = JSON.generate [1, 2, {"a"=>3.141}, false, true, nil, 4..10]
 # => "[1,2,{\"a\":3.141},false,true,null,{\"json_class\":\"Range\",\"data\":[4,10,false]}]"
 JSON.parse json
 # => [1, 2, {"a"=>3.141}, false, true, nil, 4..10]

JSON.generate always creates the shortest possible string representation of a
ruby data structure in one line. This is good for data storage or network
protocols, but not so good for humans to read. Fortunately there's also
JSON.pretty\_generate (or JSON.pretty\_generate) that creates a more readable
output:

 puts JSON.pretty_generate([1, 2, {"a"=>3.141}, false, true, nil, 4..10])
 [
   1,
   2,
   {
     "a": 3.141
   },
   false,
   true,
   null,
   {
     "json_class": "Range",
     "data": [
       4,
       10,
       false
     ]
   }
 ]

There are also the methods Kernel#j for generate, and Kernel#jj for
pretty\_generate output to the console, that work analogous to Core Ruby's p and
the pp library's pp methods.

The script tools/server.rb contains a small example if you want to test, how
receiving a JSON object from a webrick server in your browser with the
javasript prototype library http://www.prototypejs.org works.

## Speed Comparisons

I have created some benchmark results (see the benchmarks/data-p4-3Ghz
subdir of the package) for the JSON-parser to estimate the speed up in the C
extension:

 Comparing times (call_time_mean):
  1 ParserBenchmarkExt#parser   900 repeats:
        553.922304770 (  real) ->   21.500x 
          0.001805307
  2 ParserBenchmarkYAML#parser  1000 repeats:
        224.513358139 (  real) ->    8.714x 
          0.004454078
  3 ParserBenchmarkPure#parser  1000 repeats:
         26.755020642 (  real) ->    1.038x 
          0.037376163
  4 ParserBenchmarkRails#parser 1000 repeats:
         25.763381731 (  real) ->    1.000x 
          0.038814780
            calls/sec (  time) ->    speed  covers
            secs/call

In the table above 1 is JSON::Ext::Parser, 2 is YAML.load with YAML
compatbile JSON document, 3 is is JSON::Pure::Parser, and 4 is
ActiveSupport::JSON.decode. The ActiveSupport JSON-decoder converts the
input first to YAML and then uses the YAML-parser, the conversion seems to
slow it down so much that it is only as fast as the JSON::Pure::Parser!

If you look at the benchmark data you can see that this is mostly caused by
the frequent high outliers - the median of the Rails-parser runs is still
overall smaller than the median of the JSON::Pure::Parser runs:

 Comparing times (call_time_median):
  1 ParserBenchmarkExt#parser   900 repeats:
        800.592479481 (  real) ->   26.936x 
          0.001249075
  2 ParserBenchmarkYAML#parser  1000 repeats:
        271.002390644 (  real) ->    9.118x 
          0.003690004
  3 ParserBenchmarkRails#parser 1000 repeats:
         30.227910865 (  real) ->    1.017x 
          0.033082008
  4 ParserBenchmarkPure#parser  1000 repeats:
         29.722384421 (  real) ->    1.000x 
          0.033644676
            calls/sec (  time) ->    speed  covers
            secs/call

I have benchmarked the JSON-Generator as well. This generated a few more
values, because there are different modes that also influence the achieved
speed:

 Comparing times (call_time_mean):
  1 GeneratorBenchmarkExt#generator_fast    1000 repeats:
        547.354332608 (  real) ->   15.090x 
          0.001826970
  2 GeneratorBenchmarkExt#generator_safe    1000 repeats:
        443.968212317 (  real) ->   12.240x 
          0.002252414
  3 GeneratorBenchmarkExt#generator_pretty  900 repeats:
        375.104545883 (  real) ->   10.341x 
          0.002665923
  4 GeneratorBenchmarkPure#generator_fast   1000 repeats:
         49.978706968 (  real) ->    1.378x 
          0.020008521
  5 GeneratorBenchmarkRails#generator       1000 repeats:
         38.531868759 (  real) ->    1.062x 
          0.025952543
  6 GeneratorBenchmarkPure#generator_safe   1000 repeats:
         36.927649925 (  real) ->    1.018x 7 (>=3859)
          0.027079979
  7 GeneratorBenchmarkPure#generator_pretty 1000 repeats:
         36.272134441 (  real) ->    1.000x 6 (>=3859)
          0.027569373
            calls/sec (  time) ->    speed  covers
            secs/call

In the table above 1-3 are JSON::Ext::Generator methods. 4, 6, and 7 are
JSON::Pure::Generator methods and 5 is the Rails JSON generator. It is now a
bit faster than the generator\_safe and generator\_pretty methods of the pure
variant but slower than the others.

To achieve the fastest JSON document output, you can use the fast\_generate
method. Beware, that this will disable the checking for circular Ruby data
structures, which may cause JSON to go into an infinite loop.

Here are the median comparisons for completeness' sake:

 Comparing times (call_time_median):
  1 GeneratorBenchmarkExt#generator_fast    1000 repeats:
        708.258020939 (  real) ->   16.547x 
          0.001411915
  2 GeneratorBenchmarkExt#generator_safe    1000 repeats:
        569.105020353 (  real) ->   13.296x 
          0.001757145
  3 GeneratorBenchmarkExt#generator_pretty  900 repeats:
        482.825371244 (  real) ->   11.280x 
          0.002071142
  4 GeneratorBenchmarkPure#generator_fast   1000 repeats:
         62.717626652 (  real) ->    1.465x 
          0.015944481
  5 GeneratorBenchmarkRails#generator       1000 repeats:
         43.965681162 (  real) ->    1.027x 
          0.022745013
  6 GeneratorBenchmarkPure#generator_safe   1000 repeats:
         43.929073409 (  real) ->    1.026x 7 (>=3859)
          0.022763968
  7 GeneratorBenchmarkPure#generator_pretty 1000 repeats:
         42.802514491 (  real) ->    1.000x 6 (>=3859)
          0.023363113
            calls/sec (  time) ->    speed  covers
            secs/call

## Author

Florian Frank <mailto:flori@ping.de>

## License

Ruby License, see https://www.ruby-lang.org/en/about/license.txt.

## Download

The latest version of this library can be downloaded at

* https://rubygems.org/gems/json

Online Documentation should be located at

* http://json.rubyforge.org
