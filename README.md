# Magic is organizing chaos



Network Applied Communication Laboratory

Shugo Maeda

# Self introduction

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

## SKYACTIV-X

![SKYACTIV-X](skyactiv-x.jpg)

## More power for Refinements

* Block-level activation

```ruby
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

* Same block calls with different refinements break method cache

```ruby
f = Proc.new {
  ...
}
2.times do |i|
  Thread.start do
    1000.times do
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
  refine Integer do
    def /(other)
      quo(other)
    end
  end
end

def instance_eval_with_integer_div_ext(obj, &block)
  # using IntegerDivExt in the block (not Proc) represented by `block`
  block.using(IntegerDivExt)
  obj.instance_eval(&block)
end

# Necessary where blocks are written
using Proc::Refinements

p 1 / 2 #=> 0
instance_eval_with_integer_div_ext(1) do
  p self / 2 #=> (1/2)
end
p 1 / 2 #=> 0
```

## Thread safety

* Same block calls with different refinements are prohibited
* Once a block is called, new refinements cannot be added by Proc#using

```ruby
m1 = Module.new {
  refine String do
    def foo
      "m1:foo"
    end
  end
}
m2 = Module.new {
  refine String do
    def foo
      "m2:foo"
    end
  end
}
using Proc::Refinements
[m1, m2].each do |m|
  Proc.new {
    [m, "x".foo, Module.used_modules]
  }.using(m).call # error with m2
end
```

## instance_exec support

* Proc#using is independent from instance_eval and can be used with
  instance_exec

## Explicit activation

* `using Proc::Refinements` is necessary where blocks are written
    * For JRuby implementation
* Proc::Refinements is a special module that indicates blocks may be
  affected by refinements
* Actual refinements are specified by Proc#using

## Proof of Concept implementation

* For CRuby: https://github.com/shugo/ruby/pull/2
* For JRuby: https://github.com/shugo/jruby/pull/1

## 残課題

* ブロックが一度でも実行されたら新しいモジュールをProc#usingで追加できないようにしたい
* CRubyで違うクラスに対して同一ブロックでclass_evalした時にバグがあるのを修正したい
* MVM/Ractorでの挙動がどうあるべきか

## まとめ

* DSLを書きやすくするために先行事例の課題を解決するProc#usingを提案した
