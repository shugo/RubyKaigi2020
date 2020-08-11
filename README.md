# Magic is organizing chaos



Network Applied Communication Laboratory

Shugo Maeda

# Self introduction

* Shugo Maeda
* Ruby committer since the last century
* Director at Network Applied Communication Laboratory Ltd.
* Secretary General at Ruby Association

## From Matsue Shimane Japan

![Tottori](tottori.jpg)

# What are refinements?

> Magic is organizing chaos. And while oceans of mystery remain,
> we have deduced that this requires two things. Balance and control.
> Without them, chaos will kill you.

* The Witcher Season 1 Episode 2

## Example

```ruby
def foo
  p 1 / 2
end

module IntegerDivExt
  refine Integer do
    def /(other)
      quo(other)
    end
  end
end

p 1 / 2 #=> 0
using IntegerDivExt
p 1 / 2 #=> (1/2)
foo     #=> 0
```

## More power for Refinements

```ruby
# Block-level activation
foo {
  p 1 / 2 #=> (1/2)
}
```

## Why block-level activation needed?

* Adventurous refinements in narrow scopes
* For DSLs

## Example 1

```ruby
it "should generate a 1 10% of the time (plus/minus 2%)" do
  result.occurences_of(1).should be_greather_than_or_equal_to(980)
  result.occurences_of(1).should be_less_than_or_equal_to(1020)
end
```

## Example 2

```ruby
User.where { :name == 'matz' }
User.where { :age >= 18 }
```

## Previous Study

* Feature #12086: using: option for instance_eval etc.
    * https://bugs.ruby-lang.org/issues/12086

## using: option for instance_eval etc.

```ruby
module IntegerDivExt
  refine Integer do
    def /(other)
      quo(other)
    end
  end
end

p 1 / 2 #=> 0
instance_eval(using: IntegerDivExt) do
  p 1 / 2 #=> (1/2)
end
p 1 / 2 #=> 0
```

## Why instance_eval?

* instance_eval is often used for DSLs
* Switching self and activating refinements at the same time

## Issues of Feature #12086

* Thread safety
* No instance_exec support
* Implicit activation

## Thread safety

```ruby
f = Proc.new {
  ...
}
2.times do |i|
  Thread.start do
    1000.times do
      # Same block calls with different refinements break method cache
      instance_eval(using: MODULES[i], &f)
    end
  end
end
```

## No instance_exec support

* instance_exec passes all arguments to block parameters
* Adding using: is confusing

## Implicit activation

* Refinements are activated implicitly

## New proposal

* Feature #16461: Proc#using
    * https://bugs.ruby-lang.org/issues/16461

## Proc#using

```ruby
module IntegerDivExt
  refine Integer do; def /(other); quo(other); end; end
end

def instance_eval_with_integer_div_ext(obj, &block)
  # using IntegerDivExt in the block (not Proc) represented by `block`
  block.using(IntegerDivExt)
  obj.instance_eval(&block)
end

using Proc::Refinements # Necessary where blocks are written
p 1 / 2 #=> 0
instance_eval_with_integer_div_ext(1) do
  p self / 2 #=> (1/2)
end
p 1 / 2 #=> 0
```

## Not Proc-level, but block-level

```


          { p 1 / 2 }
         +-----------+    proc1.using(IntegerDivExt)   +-------+
         |   block   |<--------------------------------| proc1 |
         +-----------+                                 +-------+
               ^
               |
               |         proc2.call #=> (1/2)          +-------+
               +---------------------------------------| proc2 |
                                                       +-------+

```

## Thread safety

```ruby
m1 = Module.new {
  refine String do; def foo; "m1:foo"; end; end
}
m2 = Module.new {
  refine String do; def foo; "m2:foo"; end; end
}
using Proc::Refinements
[m1, m2].each do |m|
  # Once a block is called, new refinements cannot be added by Proc#using
  Proc.new {
    [m, "x".foo, Module.used_modules]
  }.using(m).call # error with m2
end
```

## instance_exec support

```ruby
# Proc#using is independent from instance_eval and can be used with
# instance_exec
def foo(&block)
  block.using(M)
  @obj.instance_exec(@params, &block)
end
```

## Explicit activation

* `using Proc::Refinements` is necessary where blocks are written
    * For JRuby implementation
* Proc::Refinements is a special module that indicates blocks may be
  affected by refinements
* Actual refinements are specified by Proc#using

## Proof of Concept implementation

* For CRuby: https://github.com/shugo/ruby/pull/2
* For JRuby: https://github.com/shugo/jruby/pull/1

## Summary

* Proc#using brings more power for Refinements
* Issues of previous study are resolved
