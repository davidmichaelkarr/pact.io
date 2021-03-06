## Getting started

To get started, you will need to and install the relevant pact components for the language(s) you're using for your consumer (or provider). These need not be the same for both consumer and provider - part of the value of the Pact format is that different frameworks and languages can be used for consumers and providers.

You can find instructions for each language on the [implementation guides](/implementation_guides/README.md) page.

### An example scenario

Here we have an example describing Pact tests between a consumer (the Zoo App), and its provider (the Animal Service).

In the Consumer project, we're going to need:

* A model (the Alligator class) to represent the data returned from the Animal Service
* A client (the AnimalServiceClient) which will be responsible for making the HTTP calls to the Animal Service.

Note that to create a pact, you _do_ need to write the code that executes the HTTP requests to your service (in your client class), but you _don't_ need to write the full stack of consumer code (eg. the UI).

Ideally, the Pact tests should be "unit tests" for your client class, and they should just focus on ensuring that the request creation and response handling are correct. If you use pact for your UI tests, you'll end up with an explosion of redundant interactions that will make the verification process tedious. Remember that pact is for testing the contract used for communication, and not for testing particular UI behaviour or business logic.

![Example](../media/zoo_app-animal_service.png)

### In the Zoo App (consumer) project

#### 1. Start with your model

Imagine a model class that looks something like this. The attributes for a Alligator live on a remote server, and will need to be retrieved by an HTTP call to the Animal Service.

```ruby
class Alligator
  attr_reader :name

  def initialize name
    @name = name
  end

  def == other
    other.is_a?(Alligator) && other.name == name
  end
end
```

#### 2. Create a skeleton Animal Service client class

Perhaps we have an Animal Service client class that looks something like this (please excuse the use of httparty):

```ruby
require 'httparty'

class AnimalServiceClient
  include HTTParty
  base_uri 'http://animal-service.com'

  def get_alligator
    # Yet to be implemented because we're doing Test First Development...
  end
end
```
#### 3. Configure the mock Animal Service

The following code will create a mock service on `localhost:1234` which will respond to your application's queries over HTTP as if it were the real Animal Service app. It also creates a mock provider object which you will use to set up your expectations. The method name to access the mock service provider will be what ever name you give as the service argument - in this case `animal_service`.

```ruby
# In /spec/service_providers/pact_helper.rb

require 'pact/consumer/rspec'
# or require 'pact/consumer/minitest' if you are using Minitest

Pact.service_consumer "Zoo App" do
  has_pact_with "Animal Service" do
    mock_service :animal_service do
      port 1234
    end
  end
end
```

#### 4. Write a failing spec for the Animal Service client

```ruby
# In /spec/service_providers/animal_service_client_spec.rb

# When using RSpec, use the metadata `:pact => true` to include all the pact functionality in your spec.
# When using Minitest, include Pact::Consumer::Minitest in your spec.

describe AnimalServiceClient, :pact => true do

  before do
    # Configure your client to point to the stub service on localhost using the port you have specified
    AnimalServiceClient.base_uri 'localhost:1234'
  end

  subject { AnimalServiceClient.new }

  describe "get_alligator" do

    before do
      animal_service.given("an alligator exists").
        upon_receiving("a request for an alligator").
        with(method: :get, path: '/alligator', query: '').
        will_respond_with(
          status: 200,
          headers: {'Content-Type' => 'application/json'},
          body: {name: 'Betty'} )
    end

    it "returns a alligator" do
      expect(subject.get_alligator).to eq(Alligator.new('Betty'))
    end

  end

end
```

#### 5. Run the specs

Of course, the above specs will fail because the Animal Service client method is not implemented. No pact file has been generated yet because only interactions that were correctly executed will be written to the file, and we don't have any of those yet.

#### 6. Implement the Animal Service client consumer methods

```ruby
class AnimalServiceClient
  include HTTParty
  base_uri 'http://animal-service.com'

  def get_alligator
    name = JSON.parse(self.class.get("/alligator").body)['name']
    Alligator.new(name)
  end
end
```

#### 7. Run the specs again.

Green!

Running the passing AnimalServiceClient spec will generate a pact file in the configured pact dir (`spec/pacts` by default).
Logs will be output to the configured log dir (`log` by default) that can be useful when diagnosing problems.

You now have a pact file that can be used to verify your expectations of the Animal Service provider project.

Now, rinse and repeat for other likely status codes that may be returned. For example, consider how you want your client to respond to a:

* 404 (return null, or raise an error?)
* 400 (how should validation errors be handled, what will the body look like when there is one?)
* 500 (specifying that the response body should contain an error message, and ensuring that your client logs that error message will make your life much easier when things go wrong. Note that it may be hard to force your provider to generate a 500 error on demand if you are not using Ruby. You may need to collaborate with your provider team to create a known provider state that will artificially return a 500 error, or you may just wish to use a standard unit test without a pact to test this.)
* 401/403 if there is authorisation.

### In the Animal Service (provider) project

#### 1. Create the skeleton API classes

Create your API class using the framework of your choice (the Pact authors have a preference for [Webmachine][webmachine] and [Roar][roar]) - leave the methods unimplemented, we're doing Test First Develoment, remember?

[webmachine]: https://github.com/webmachine/webmachine-ruby
[roar]: https://github.com/trailblazer/roar

#### 2. Tell your provider that it needs to honour the pact file you made earlier

Require "pact/tasks" in your Rakefile.

```ruby
# In Rakefile
require 'pact/tasks'
```

Create a `pact_helper.rb` in your service provider project. The recommended place is `spec/service_consumers/pact_helper.rb`.

For more information, see [Verifying Pacts](/getting_started/verifying_pacts.md) and the provider configuration documentation for your Pact implementation language (if you are following this example, here is the [Ruby documentation](https://github.com/pact-foundation/pact-ruby/wiki/Verifying-pacts)).

```ruby
# In specs/service_consumers/pact_helper.rb

require 'pact/provider/rspec'

Pact.service_provider "Animal Service" do

  honours_pact_with 'Zoo App' do

    # This example points to a local file, however, on a real project with a continuous
    # integration box, you would use a [Pact Broker](https://github.com/bethesque/pact_broker) or publish your pacts as artifacts,
    # and point the pact_uri to the pact published by the last successful build.

    pact_uri '../zoo-app/specs/pacts/zoo_app-animal_service.json'
  end
end
```

#### 3. Run your failing specs

    $ rake pact:verify

Congratulations! You now have a failing spec to develop against.

At this stage, you'll want to be able to run your specs one at a time while you implement each feature. At the bottom of the failed pact:verify output you will see the commands to rerun each failed interaction individually. A command to run just one interaction will look like this:

    $ rake pact:verify PACT_DESCRIPTION="a request for an alligator" PACT_PROVIDER_STATE="an alligator exists"

#### 4. Implement enough to make your first interaction spec pass

Rinse and repeat.

#### 5. Keep going til you're green

Yay! Your Animal Service provider now honours the pact it has with your Zoo App consumer. You can now have confidence that your consumer and provider will play nicely together.
