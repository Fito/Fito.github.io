---
layout: post
title:  "Unit Testing"
date:   2017-04-18 23:53:42 -0700
---
I joined Entelo about two and a half months ago and I'm excited to be working as part of a team that fully embraces testing as part our software development process.

Unit tests provide many benefits in the short and long run. However, instead of writing about these benefits, I will assume that you are already familiar with them. I will instead focus on some of the practices I follow when writing unit tests.
Tests are an investment into your future codebase. The benefits of your investment could be reaped at different points in time, maybe as soon as the next couple of hours of coding, or while making changes to your codebase months later.

However, like with any other type of investment, you should invest wisely. Tests are code, which means they require some level of maintenance. Also, slow tests increase development time and, when ran as part of a continuous integration system, can even get in the way of a deployment.
To get the most out of unit tests, while keeping their cost down, I apply the few rules of thumb:

#### Test only public interfaces
An object's public interface is the way other objects in the system can interact with it. It encompasses a contract with the rest of the codebase. When this contract is broken tests should fail, thus raising an alarm that there is a broken path through the codebase. However, private methods and the internals of a class should be allowed to change freely as long as the contract is not broken. Testing a private method adds unnecessary friction to the development process and provides little value, if at all.

Example:
```ruby
class Car
  # should only test this method
  def miles_per_dollar_amount(dollar_amount)
    (current_price_per_gallon * dollar_amount) / miles_per_gallon
  end

  private
  # this method is only used within this class and should not be tested
  def current_price_per_gallon
    # get current gas prices from some API
  end
end

describe Car do
  subject { Car.new }
  # other test setup

  describe "#miles_per_dollar_amount" do
    it "returns the miles a car will travel for the given amount" do
      expect(subject.miles_per_dollar_amount(dollars)).to eq(miles)
    end
  end
end
```

#### Write descriptive and eloquent tests
Tests play a larger role than just running your code. Tests are live documentation and as such, they should provide enough context about what's being tested. The next person updating a test should get enough information to confidently make a change by just reading the test statements. Running the test should give a developer complete and sensical sentences explaining what passed and what didn't.

Example of a not very descriptive test:
```ruby
describe Car do
  subject { Car.new }
  # other test setup
  describe "#move" do
    context "no gas" do
      # setup for an empty gas tank
      it "does not work" do
        expect { subject.run }.not_to change { subject.current_position }
      end
    end
  end
```

Example of a descriptive test:
```ruby
describe Car do
  subject { Car.new }
  # other test setup
  describe "#move" do
    context "when there is no gas left in the tank" do
      # setup for an empty gas tank
      it "does not change its current position" do
        expect { subject.run }.not_to change { subject.current_position }
      end
    end
  end
end

```

#### Mock/Stub dependencies
Often objects depend on other objects to accomplish their task. This dependency should be documented in tests via assertions about the right object receiving the right messages.

However, these tests should only assert that a message _is passed_, not what the receiver does or what it returns. These dependencies should have already been tested, and developers should trust that they do what they are supposed to do.

Testing what an object's dependency does creates unnecessary coupling between the test itself and the dependency.

Example of a test with unnecessary coupling:
```ruby
describe Mechanic do
  subject { Mechanic.new }
  let(:car) { Car.new(engine: engine) }
  let(:engine) { Engine.new }

  describe "#change_oil" do
    it "gets the car's oil level from its engine" do
      expect(engine).to receive(:oil_level)
      subject.change_oil(car)
    end
  end
end
```

A change in `Engine`'s interface would require this test to change as well, when in reality `Mechanic` only collaborates with `Car`.

Example of test stubbing a dependency:
```ruby
describe Mechanic do
  subject { Mechanic.new }
  let(:car) { double('car', oil_level: oil_level) }
  let(:oil_level) { 'some level' } # not our concern how car gets this

  describe "#change_oil" do
    it "measures the car's oil level" do
      expect(car).to receive(:oil_level).and_return(oil_level)
      subject.change_oil(car)
    end
  end
end
```

#### Avoid trips to the database as much as possible
When testing an object that collaborates with ActiveRecord objects (or any other ORM), it often seems easier to prepare the necessary test context by creating database records (often via factories).
However, this can lead to making unnecessary trips to the database. Every 'trip' to the database slows down tests significantly, which in turn slows down the software development process itself.

This slowdown becomes more noticeable as the codebase grows and the number of tests increases.
To avoid this problem, unit tests should treat ActiveRecord as any other dependency and test only the message passing.

Example of unnecessary database trips:
```ruby
describe Notification do
  # Group and User are ActiveRecord Models
  subject { Notification.new(group: users_group, message) }
  let(:message) { "Some Message" }
  let(:users_group) { Group.create }
  let(:user) { User.create }

  describe "#send" do
    before do
      allow(users_group).to receive(:users).and_return([user])
    end

    it "notifies all users in the group" do
      expect(user).to receive(:notify).with(message)
    end
  end
end
```
Example leveraging doubles to avoid database trips:
```ruby
describe Notification do
  # Group and User are ActiveRecord Models
  subject { Notification.new(group: users_group, message) }
  let(:message) { "Some Message" }
  let(:users_group) { double('group', users: [user]) }
  let(:user) { double('user', notify: true) }

  describe "#send" do
    it "notifies all users in the group" do
      expect(user).to receive(:notify).with(message)
    end
  end
end
```

#### Test logic, not configuration
This rule of thumb is better explained by an example.
Let's suppose we have a `CarSerializer` class, that inherits from `ActiveModel::Serializer` as follows:

```ruby
class CarSerializer < ActiveModel::Serializer
  attributes :color, :maker, :popularity

  def popularity
    # run some query calculate a car's popularity
  end
end
```

The `attributes` method allows us to configure what attributes we would like to see in the resulting serialized object. However, the parent class `ActiveModel::Serializer` takes care of sending those messages to the appropriate objects. The `attributes` method is _configuration_ and its behavior does not need to be tested in the `CarSerializer`'s test. The fact that the parent class sends the `color` and `maker` messages to the object being serialized does not need to be tested.

Example of unnecessary test:
```ruby
describe CarSerializer do
  subject { CarSerializer.new(car) }
  let(:car) { double('car', color: 'red', maker: 'some_maker') }

  describe "#color" do
    it "gets the color from the car" do
      expect(car).to receive(:color)
      subject.to_json
    end
  end
end
```
The `popularity` method on the other hand contains _logic_ that's specific to the `CardSerializer` class and, therefore, should be tested.

Example of necessary test:
```ruby
describe CarSerializer do
  subject { CarSerializer.new(car) }
  let(:car) { double('car') }

  describe "#popularity" do
    it "returns a car's popularity based on x, y, z factors" do
      # appropriate expectation based on test setup
    end
  end
end
```

Following these rules allows me to write high-value/low-cost tests. They allow me to move fast while giving me a safety net.
Of course, I did not come up with these rules myself. While they are heavily influenced by my experience, my co-workers and my personal style of software design, they are also based on various concepts borrowed from the works of Martin Fowler and Sandi Metz.

Lastly, these ideas constitute a guideline. I follow then as much as I can, but I compromise when necessary.
