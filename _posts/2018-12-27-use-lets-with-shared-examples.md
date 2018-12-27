---
layout: post
title:  "Using lets as Arguments for Shared Examples"
date:   2018-12-27 12:00:00 -0500
categories: [RSpec, Ruby]
description: "RSpec: Lazy-loading let helpers in Shared Examples"
---
Like many, I often use RSpec's [Shared Examples](https://relishapp.com/rspec/rspec-core/docs/example-groups/shared-examples) to help DRY my tests and better communicate intent, especially when the Thing™ under test has side effects or many possible inputs.

Also like many, I make liberal use of [`let` helpers](https://relishapp.com/rspec/rspec-core/docs/helper-methods/let-and-let).

**This post describes one approach to combining those two concepts, stepping around a pitfall I encountered.**

### Shared Examples with Arguments

Consider this contrived hunk of test, for a `User` class with (apparently) two methods that do similar things:

#### Original RSpec

{% highlight ruby %}
describe User do
  describe "#full_name" do
    subject { described_class.new(first_name: "Foo", last_name: "Bar").full_name }
    it "combines first_name and last_name" do
      expect(subject).to eq "Foo Bar"
    end
  end

  describe "#full_city" do
    subject { described_class.new(city: "Columbus, OH", zip_code: "43202").full_city }
    it "combines city and zip_code" do
      expect(subject).to eq "Columbus, OH 43202"
    end
  end
end
{% endhighlight %}

In the opinion of many (myself included), this is perfectly fine and DRY enough. For the purposes of education, though, let's **pull out the notion of "combining" things** into a Shared Example block, passing that block the bits that need combined, like so:

#### Original RSpec with Shared Example

{% highlight ruby %}
describe User do
  shared_examples "combines" do |string, other_string|
    expect(subject).to eq "#{string} #{other_string}"
  end

  describe "#full_name" do
    subject { described_class.new(first_name: "Foo", last_name: "Bar").full_name }

    include_examples "combines", "Foo", "Bar"
  end

  describe "#full_city" do
    subject { described_class.new(city: "Columbus, OH", zip_code: "43202").full_city }

    include_examples "combines", "Columbus, OH", "43202"
  end
end
{% endhighlight %}

The `include_examples` helper method takes as arguments first the name of the `shared_examples` block, which is perhaps obviously used to identify the block, then any number of other arguments which are passed along to the block.

So, we've pulled the "combination" logic into its own bit of code, and have made the intent of the tests clearer (or, at least, we might have if our example were more complex).

## The Problem With let

What if our original RSpec was set up like this (which arguably it should have been from the start)?

#### Original RSpec with let

{% highlight ruby %}
describe User do
  describe "#full_name" do
    let(:first_name) { "Foo" }
    let(:last_name)  { "Bar" }

    subject { described_class.new(first_name: first_name, last_name: last_name).full_name }

    it "combines first_name and last_name" do
      expect(subject).to eq "#{first_name} #{last_name}"
    end
  end

  describe "#full_city" do
    let(:city)     { "Columbus, OH" }
    let(:zip_code) { "43202" }

    subject { described_class.new(city: city, zip_code: zip_code).full_city }

    it "combines city and zip_code" do
      expect(subject).to eq "#{city} #{zip_code}"
    end
  end
end
{% endhighlight %}

That's all well and good, and you **might expect to be able to apply Shared Examples** like this, passing the helper methods provided by `let` as parameters, to be evaluated at runtime:

#### What may seem natural

{% highlight ruby %}
# snip...
describe "#full_name" do
  let(:first_name) { "Foo" }
  let(:last_name)  { "Bar" }

  subject { described_class.new(first_name: first_name, last_name: last_name).full_name }

  include_examples "combines", first_name, last_name
end
# ...snip
{% endhighlight %}

However, **this doesn't work, and RSpec will scold you** in a very direct way: 

```
RSpec::Core::ExampleGroup::WrongScopeError:
`first_name` is not available on an example group (e.g. a `describe` or
`context` block). It is only available from within individual examples
(e.g. `it` blocks) or from constructs that run in the scope of an example
(e.g. `before`, `let`, etc).
```

Whoops 💥! At this point, it's a good idea to question whether you need Shared Examples at all, because _you might be doing something weird that should or could be addressed elsewhere_. Your implementation might be factored oddly, for instance.

Again, though, **we shall press on in the name of science**, and also because in the real world you can't always address such things _right now_.

First, you could define an additional `let` helper, something like the below, to define the "arguments" to the example outside of the example itself. I find this messy and it's not easy to intuit the fact that this `let` is important to the `include_examples` that follows.

#### A gross workaround

{% highlight ruby %}
# snip...
shared_examples "combines"
  expect(subject).to eq "#{to_be_combined[0]} #{to_be_combined[1]}"
end

describe "#full_name" do
  let(:first_name) { "Foo" }
  let(:last_name)  { "Bar" }

  subject { described_class.new(first_name: first_name, last_name: last_name).full_name }

  let(:to_be_combined) { [first_name, last_name] }
  include_examples "combines"
end
# ...snip
{% endhighlight %}

## Dynamically Evaluating let Helpers at Runtime

Here is _a way_ to make this work, which I have used with success when stuck in this and similar pickles:

{% highlight ruby %}
# snip...
shared_examples "combines" |string_identifier, other_string_identifier| # Don't love those names
  expect(subject).to eq "#{send(string_identifier)} #{send(other_string_identifier)}"
end

describe "#full_name" do
  let(:first_name) { "Foo" }
  let(:last_name)  { "Bar" }

  subject { described_class.new(first_name: first_name, last_name: last_name).full_name }

  include_examples "combines", :first_name, :last_name
end
# ...snip
{% endhighlight %}

This works because `let(:first_name) { "Foo" }` simply defines a (memoized) method called `first_name`wri, and `send` (part of Ruby itself) can be used to call methods in the current scope by their symbol name. RSpec won't complain about passing symbols as arguments, the Shared Example evaluates the helpers, and everything is green! 💯

**This, too, is not pretty, but I much prefer it to adding an extra `let`.**

Look for a follow up post describing a better Shared Example for this particular test.

Questions? Comments? What did I do wrong? Tell me on [Twitter](https://www.twitter.com/alexford).