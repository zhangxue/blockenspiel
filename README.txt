== Blockenspiel

Blockenspiel is a helper library designed to make it easy to implement DSL
blocks. It is designed to be comprehensive and robust, supporting most common
usage patterns, and working correctly in the presence of nested blocks and
multithreading.

=== What's a DSL block?

A DSL block is an API pattern in which a method call takes a block that can
provide further configuration for the call. A classic example is the
{Rails}[http://www.rubyonrails.org/] route definition:

 ActionController::Routing::Routes.draw do |map|
   map.connect ':controller/:action/:id'
   map.connect ':controller/:action/:id.:format'
 end

Some libraries go one step further and eliminate the need for a block
parameter. {RSpec}[http://rspec.info/] is a well-known example:

 describe Stack do
   before(:each) do
     @stack = Stack.new
   end
   describe "(empty)" do
     it { @stack.should be_empty }
     it "should complain when sent #peek" do
       lambda { @stack.peek }.should raise_error(StackUnderflowError)
     end
   end
 end

In both cases, the caller provides descriptive information in the block,
using a domain-specific language. The second form, which eliminates the block
parameter, often appears cleaner; however it is also sometimes less clear
what is actually going on.

=== How does one implement such a beast?

Implementing the first form is fairly straightforward. You would create a
class defining the methods (such as +connect+ in our Rails routing example
above) that should be available within the block. When, for example, the
<tt>draw</tt> method is called with a block, you instantiate the class and
yield it to the block.

The second form is perhaps more mystifying. Somehow you would need to make
the DSL methods available on the "self" object inside the block. There are
several plausible ways to do this, such as using <tt>instance_eval</tt>.
However, there are many subtle pitfalls in such techniques, and quite a bit
of discussion has taken place in the Ruby community regarding how--or
whether--to safely implement such a syntax.

I have included a critical survey of the debate in the document
{ImplementingDSLblocks.txt}[link:files/ImplementingDSLblocks\_txt.html] for
the curious. Blockenspiel takes what I consider the best of the solutions and
implements them in a comprehensive way, shielding you from the complexity of
the Ruby metaprogramming while offering a simple way to implement both forms
of DSL blocks.

=== So what _is_ Blockenspiel?

Blockenspiel operates on the following observations:

* Implementing a DSL block that takes a parameter is straightforward.
* Safely implementing a DSL block that <em>doesn't</em> take a parameter is tricky.

With that in mind, Blockenspiel provides a set of tools that allow you to
take an implementation of the first form of a DSL block, one that takes a
parameter, and turn it into an implementation of the second form, one that
doesn't take a parameter.

Suppose you wanted to write a simple DSL block that takes a parameter:

 configure_me do |config|
   config.add_foo(1)
   config.add_bar(2)
 end

You could write this as follows:

 class ConfigMethods
   def add_foo(value)
     # do something
   end
   def add_bar(value)
     # do something
   end
 end
 
 def configure_me
   yield ConfigMethods.new
 end

That was easy. However, now suppose you wanted to support usage _without_
the "config" parameter. e.g.

 configure_me do
   add_foo(1)
   add_bar(2)
 end

With Blockenspiel, you can do this in two quick steps.
First, tell Blockenspiel that your +ConfigMethods+ class is a DSL.

 class ConfigMethods
   include Blockenspiel::DSL   # <--- Add this line
   def add_foo(value)
     # do something
   end
   def add_bar(value)
     # do something
   end
 end

Next, write your <tt>configure_me</tt> method using Blockenspiel:

 def configure_me(&block)
   Blockenspiel.invoke(block, ConfigMethods.new)
 end

Now, your <tt>configure_me</tt> method supports _both_ DSL block forms. A
caller can opt to use the first form, with a parameter, simply by providing
a block that takes a parameter. Or, if the caller provides a block that
doesn't take a parameter, the second form without a parameter is used.

=== How does that help me? (Or, why not just use instance_eval?)

As noted earlier, some libraries that provide parameter-less DSL blocks use
<tt>instance_eval</tt>, and they could even support both the parameter and
parameter-less mechanisms by checking the block arity:

 def configure_me(&block)
   if block.arity == 1
     yield ConfigMethods.new
   else
     ConfigMethods.new.instance_eval(&block)
   end
 end

That seems like a simple and effective technique that doesn't require a
separate library, so why use Blockenspiel? Because <tt>instance_eval</tt>
introduces a number of surprising problems. I discuss these issues in detail
in {ImplementingDSLblocks.txt}[link:files/ImplementingDSLblocks\_txt.html],
but just to get your feet wet, suppose the caller wanted to call its own
methods inside the block:

 def callers_helper_method
   # ...
 end
 
 configure_me do
   add_foo(1)
   callers_helper_method  # Error! self is now an instance of ConfigMethods
                          # so this will fail with a NameError
   add_bar(2)
 end

Blockenspiel by default does _not_ use the <tt>instance_eval</tt> technique.
Instead, it implements a mechanism using mixin modules, a technique first
{proposed}[http://hackety.org/2008/10/06/mixingOurWayOutOfInstanceEval.html]
by Why. In this technique, the <tt>add_foo</tt> and <tt>add_bar</tt> methods
are temporarily mixed into the caller's +self+ object. That is, +self+ does
not change, as it would if we used <tt>instance_eval</tt>, so helper methods
like <tt>callers_helper_method</tt> still remain available as expected. But,
the <tt>add_foo</tt> and <tt>add_bar</tt> methods are also made available
temporarily for the duration of the block. When called, they are intercepted
and redirected to your +ConfigMethods+ instance just as if you had called
them directly via a block parameter. Blockenspiel handles the object
redirection behind the scenes so you do not have to think about it. With
Blockenspiel, the caller retains access to its helper methods, and even its
own instance variables, within the block, because +self+ has not been
modified.

=== Is that it?

Although the basic usage is very simple, Blockenspiel is designed to be
_comprehensive_. It supports all the use cases that I've run into during my
own implementation of DSL blocks. Notably:

By default, Blockenspiel lets the caller choose to use a parametered block
or a parameterless block, based on whether or not the block actually takes a
parameter. You can also disable one or the other, to force the use of either
a parametered or parameterless block.

You can control wich methods of the class are available from parameterless
blocks, and/or make some methods available under different names. Here are
a few examples:

 class ConfigMethods
   include Blockenspiel::DSL
   
   def add_foo         # automatically added to the dsl
     # do stuff...
   end
   
   def my_private_method
     # do stuff...
   end
   dsl_method :my_private_method, false   # remove from the dsl
   
   dsl_methods false   # stop automatically adding methods to the dsl
   
   def another_private_method  # not added
     # do stuff...
   end
   
   dsl_methods true    # resume automatically adding methods to the dsl
   
   def add_bar         # this method is automatically added
     # do stuff...
   end
   
   def add_baz
     # do stuff
   end
   dsl_method :add_baz_in_dsl, :add_baz  # Method named differently
                                         # in a parameterless block
 end

This is also useful, for example, when you use <tt>attr_writer</tt>.
Parameterless blocks do not support <tt>attr_writer</tt> (or, by corollary,
<tt>attr_accessor</tt>) well because methods with names of the form
"attribute=" are syntactically indistinguishable from variable assignments:

 configure_me do |config|
   config.foo = 1    # works fine when the block has a parameter
 end
 
 configure_me do
   # foo = 1     # <--- Doesn't work: looks like a variable assignment
   set_foo(1)    # <--- Renamed to this instead
 end
 
 # This is implemented like this::
 class ConfigMethods
   include Blockenspiel::DSL
   attr_writer :foo
   dsl_method :set_foo, :foo=    # Make "foo=" available as "set_foo"
 end

In some cases, you might want to dynamically generate a DSL object rather
than defining a static class. Blockenspiel provides a tool to do just that.
Here's an example:

 Blockenspiel.invoke(block) do
   add_method(:set_foo) do |value|
     my_foo = value
   end
   add_method(:set_things_using_block, :receive_block => true) do |value, blk|
     my_foo = value
     my_bar = blk.call
   end
 end

That API is in itself a DSL block, and yes, Blockenspiel uses itself to
implement this feature.

By default Blockenspiel uses mixins, which usually exhibit safe and
non-surprising behavior. However, there are a few cases when you might
want the <tt>instance_eval</tt> behavior anyway. RSpec is a good example of
such a case, since the DSL is being used to construct objects, so it makes
sense for instance variables inside the block to belong to the object
being constructed. Blockenspiel gives you the option of choosing
<tt>instance_eval</tt> in case you need it. Blockenspiel also provides a
compromise behavior that uses a proxy to dispatch methods to the DSL object
or the block's context.

Blockenspiel also correctly handles nested blocks. e.g.

 configure_me do
   set_foo(1)
   configure_another do     # A block within another block
     set_bar(2)
     configure_another do   # A block within itself
       set_bar(3)
     end
   end
 end

Finally, it is completely thread safe, correctly handling, for example, the
case of multiple threads trying to mix methods into the same object
concurrently.

=== Requirements

* Ruby 1.8.6 or later.
* Rubygems
* mixology gem.

=== Installation

 gem install blockenspiel

=== Known issues and limitations

* Implementing wildcard DSL methods using <tt>method_missing</tt> doesn't
  work. I haven't yet figured out the right semantics for this case.
* Doesn't yet work in Ruby 1.9 because the mixology gem is not compatible at this point.
* JRuby status not yet known.

=== Development and support

Documentation is available at http://virtuoso.rubyforge.org/blockenspiel

Source code is hosted by Github at http://github.com/dazuma/blockenspiel/tree

Report bugs on RubyForge at http://rubyforge.org/projects/virtuoso

Contact the author at dazuma at gmail dot com.

=== Author / Credits

Blockenspiel is written by Daniel Azuma (http://www.daniel-azuma.com).

The mixin implementation is based on a concept by Why The Lucky Stiff.
See his 6 October 2008 blog posting,
<em>{Mixing Our Way Out Of Instance Eval?}[http://hackety.org/2008/10/06/mixingOurWayOutOfInstanceEval.html]</em>
for further discussion.

=== License

Copyright 2008 Daniel Azuma.

All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice,
  this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.
* Neither the name of the copyright holder, nor the names of any other
  contributors to this software, may be used to endorse or promote products
  derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
