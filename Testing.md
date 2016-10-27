# Testing

## Test suites with inter service calls

### Context

In a microservice architecture, interservice communication happens frequently and that is a problem when it comes to testing. Tests depend on the data their are using, and data is sometimes fetched by api calls. Making those api calls introduces one more level of complexity, which is using data that needs to be maintained in a separate service. We avoid that complexity by recording those services responses into [VCR cassettes](https://github.com/vcr/vcr) and using them from there on.

Using cassettes means to avoid relaying on dynamic data that is retuned from other services, and also a faster test suite with no api calls. Developers need to run the tests to update cassettes and commit them as part of every feature.    

### VCR configuration in Axiom Cloud

```ruby
# spec/support/config/vcr.rb

require 'vcr'

VCR.configure do |c|
  record_option = (ENV['RECORD'] || :none).to_sym

  c.cassette_library_dir = Rails.root.join('spec', 'cassettes')
  c.hook_into :typhoeus
  c.default_cassette_options = {
    record: record_option,
    match_requests_on: [:method, :uri, :query, :body]
  }
  c.configure_rspec_metadata!
  c.allow_http_connections_when_no_cassette = false
end
```

#### Library directory for cassettes
The `cassette_library_dir` configuration option sets a directory where VCR saves each cassette.

```ruby
c.cassette_library_dir = Rails.root.join('spec', 'cassettes')
```

This means that all cassettes are going to be recorded/replayed under `spec/cassettes` folder:

```bash
$ tree spec/cassettes

spec/cassettes
├── Account_attributes_page
│   └── index
│       ├── get_json_response_for_account_attributes
│       │   └── should_return_account_attributes_json.yml
│       ├── update_the_account_attributes
│       │   └── 1_1_4_1.yml
├── Account_billing_information
│   ├── invoicing_options
│   │   ├── delivery_method
...
```

#### `hook_into`
The `hook_into` configuration option determines how VCR hooks into the HTTP requests to record and replay them. There are currently 4 valid options which support many different HTTP libraries:

* `:webmock`
* `:typhoeus`
* `:excon`
* `:faraday`

We are using typhoeus gem in our project for ActiveService, so we decided to reuse it for VCR.

```ruby
c.hook_into :typhoeus
```

#### `default_cassette_options`
The `default_cassette_options` configuration option takes a hash that provides defaults for each cassette we use. Any cassette can override the defaults as well as set additional options.

```ruby
  ...
  record_option = (ENV['RECORD'] || :none).to_sym
  ...
  c.default_cassette_options = {
    record: record_option,
    match_requests_on: [:method, :uri, :query, :body]
  }
```

The `:match_requests_on` option tells VCR how to match requests that are replayed from cassettes while running specs. It defaults to `[:method, :uri]` when it has not been set.

Axiom Cloud hits API endpoints that share the same URI, but depending on the HTTP method they allow a body or query params to be sent. Thus, we configured this option to `[:method, :uri, :query, :body]`.

The `:record` option tells VCR the mode in which the cassettes will be recorded. It defaults to `:once` when it has not been set.

There are four record modes provided by VCR:
* `once`
* `all`
* `new_episodes`
* `none`

We suggest developers working in Axiom Cloud to use only `all` and `none` record modes.

The `all` mode is the one we use to record interactions in our cassettes. The way it works is simple: whenever we run a test and VCR finds requests being done, the cassette for that test will be overriden with all those requests. This means that no previously recorded requests will be replayed:

```bash
$ RECORD=all bundle exec rspec
```

The `none` mode is the one we use when we want to run our tests replaying interactions from cassettes. In case that a test does not have a corresponding cassette, then the request won't be found and an error will be raised.

```bash
$ RECORD=none bundle exec rspec
```
alias for:
```bash
$ bundle exec rspec
```
If we run a test using the `none` mode and VCR cannot find the interactions for that test, then VCR will make the test fail and render the following output:

```
$ bundle exec rspec spec/requests/companies_spec:24
F

VCR::Errors::UnhandledHTTPRequestError:
    ================================================================================
    An HTTP request has been made that VCR does not know how to handle:
      GET /companies/123

    VCR is currently using the following cassette:
      - /companies/index/1_1.yml
      - :record => :none
      - :match_requests_on => [:method, :uri, :query, :body]

    Under the current configuration VCR can not find a suitable HTTP interaction
    to replay and is prevented from recording new requests. There are a few ways
    you can deal with this:

      * If you're surprised VCR is raising this error
        and want insight about how VCR attempted to handle the request,
        ...
```

So, VCR is suggesting us to record the interactions for this test.

#### `configure_rspec_metadata!`
VCR provides easy integration with RSpec using metadata. To set this up, we call `configure_rspec_metadata!` in our VCR configuration block.

Once we do this, we can have an example group or example use VCR by passing `:vcr` as an additional argument after the description string. It will set the cassette name based on the example's full description.

If we need to override the cassette name or options, we can pass a hash `(:vcr => { ... })`.

For instance, let's say we have the following request spec:

```ruby
require 'rails_helper'

describe 'Account pages', :vcr do
  ...

  describe 'show' do
    ...

    context 'when basic account' do
      it 'renders account information' do
        ...
      end

      it { is_expected.not_to have_content('View master') }
      it { is_expected.to have_content('View something') }
...
```

The following cassettes will be recorded after running `rspec` with the `all` mode set:

```
spec/cassettes/Account_Pages/show/when_basic_account/renders_account_information.yml
spec/cassettes/Account_Pages/show/when_basic_account/1_1_1_1.yml
spec/cassettes/Account_Pages/show/when_basic_account/1_1_1_2.yml
```

As we can see from above, VCR records a cassette per example using the `describe`, `context` and `it` string descriptions. For those examples that do not have a string description, VCR generates the cassette name by using the depth level of the example.

There are some advantages of using this default approach of recording cassettes. For instance, each test stores it's own interactions, so it's very simple to find that cassette.

In the past, we were using a different approach for recording cassettes. We were recording all the accounts test's interactions on an single cassette which we were explicitly naming:

```ruby
require 'rails_helper'

describe 'Account pages', vcr: { cassette_name: 'accounts' } do
   ...
```

This is a good approach as well, but it was hard to maintain. For instance, whenever a test got deleted, it was very hard to find it's related requests on the big cassette to delete them.

The disadvantage about this new approach of recording cassettes is related to test examples that do not have description. For instance, let's say we have the following integration spec:

```ruby
require 'rails_helper'

describe 'Company billing information', :vcr do
  include_context 'sign in' do
    let(:user) { create(:user) }
  end

  describe 'tax information section', js: true do

    before do
      visit company_settings_billing_path(123)
    end

    it { expect(page).to have_content('Tax Information') }
    it { expect(page).to have_content('ABC1234') }
  end
  ...
```

we have 2 cassettes that correspond to the above 2 tests:

```
spec/cassettes/Company_billing_information/tax_information_section/1_1_1.yml
spec/cassettes/Company_billing_information/tax_information_section/1_1_2.yml
```

What would happen if a developer wanted to add another test example without a string description on top of our 2 previous tests?

```ruby
require 'rails_helper'

describe 'Company billing information', :vcr do
  include_context 'sign in' do
    let(:user) { create(:user) }
  end

  describe 'tax information section', js: true do

    before do
      visit company_settings_billing_path(123)
    end

    it { expect(page).to have_content('Companies') }
    it { expect(page).to have_content('Tax Information') }
    it { expect(page).to have_content('ABC1234') }
  end
  ...
```

VCR will then think that the interactions for our new tests are the ones stored on cassette `1_1_1.yml`.

We can take 2 approaches here:
* Renaming cassettes and next recording one for our new text example.
* Simpler: move the new test to the bottom.

#### Don't allow HTTP connections when no cassette

```
VCR.configure do |c|
  c.allow_http_connections_when_no_cassette = false
end
```

Since every api call should be recorded by a cassette, there should not be any http connection being generated to an external service when running the test suite with cassettes (by default). When running rspec tests with the `all` mode, developers will be able to make API calls and record to cassettes.

### Consequences of using VCR
We can see the following consequences of using VCR the way we configured it in Axiom Cloud:
* Test suite runs faster because it does not depend on external requests.
* Test suite does not depend on the network to do external requests.
* In case that an API changes, tests will still run green replaying out of date cassettes. So, at the point we update an API, we are responsible of re-recording the cassettes for the tests that hit this API.
* Whenever a test gets deleted or renamed, the cassette needs to be deleted/renamed. This does not sound like a happy solution, but we have an automated way of knowing which cassettes are not used by the test suite. So, we can delete those unused cassettes.
* We do not have much flexibility to use gems that provide random strings or numbers to use in our integration specs to fill in forms such as [faker](https://github.com/stympy/faker). If we were using random names and numbers, our requests would change and VCR would not be able to match the pre-recorded interactions in our cassettes.
* Something similar can happen with dates. If we are using dates for requests, we must stub them in our specs so that the request does not change.
* The API must be up and running in order use VCR.

### Identifying unused cassettes
Whenever we run rspec, VCR will try searching for interactions on a cassette that has a name that matches our test description. Sometimes, when we delete tests or rename their descriptions, we might forget to delete the old cassette.

So, we found an automated way to check which cassettes are unused by our test suite, based on an [issue](https://github.com/vcr/vcr/issues/283) reported to VCR gem.

Whenever we want to clean up unused cassettes, we need to run rspec with the environment variable `CHECK_CASSETTES` set to `true`:

```bash
$ CHECK_CASSETTES=true be rspec
Randomized with seed 57335

.......................................*.............................
.........*...........................................................
.....................................................................
....................................................*................
......*...............................................**....*.*******
********.**..........................................................

******************
**** WARNING! ****
******************
The following cassettes are not being used:
.../spec/cassettes/Account_pages/comments/when_a_comment_is_posted/and_there_is_initially_no_comment_text/should_show_an_error.yml
.../spec/cassettes/Batch_Job_pages/index/when_doing_a_search_and_advance_search/when_doing_a_basic_search/1_1_1_1_1.yml
.../spec/cassettes/company.yml
...
```

Then, rspec will suggest us which cassettes are unused so that we can delete them.

### A day in a developer's life

1. Make sure develop branch is running green before branching out of it.
    If it's not, **stop the line** until it becomes green again.
2. Code your feature and tests.
3. Make sure tests run green recording cassettes locally for our new tests or modified ones:
4. Make sure cassettes got generated/updated in previous step and add them to the commits.
5. Make sure tests run green when replaying interactions from cassettes locally.
6. Make sure no random failures were found while executing all the above.

#### Example
Let's suppose that we are going to work on a new feature that provides the companies pages in Axiom Cloud to update the Tax Information for a company.

First we need to make sure that develop branch is running green. So, we can check this in [teamcity](https://teamcity.cb.com/project.html?projectId=Axiom).

Next, we pull develop locally and create our feature branch from it:

```bash
$ git checkout develop
$ git pull origin develop
$ git checkout -b feature/B-12345-tax-information
```

We can now start working on our feature branch and adding tests for it, for instance:

```ruby
require 'rails_helper'

describe 'Company billing information', :vcr do
  include_context 'sign in' do
    let(:user) { create(:user) }
  end

  describe 'tax information section', js: true do

    before do
      visit company_settings_billing_path(123)
    end

    it { expect(page).to have_content('Tax Information') }
    it { expect(page).to have_content('ABC1234') }
  end
  ...
```

If we were running rspec for this new tests, our tests are going to fail in case we use the `none` record mode, because we did not record cassettes yet:

```bash
$ bundle exec rspec spec/requests/companies/settings/billings_spec.rb
FFFFF
```

And we will get an exception like the following one either as an output of rspec or in the test log file `logs/test.log`:
```bash
...
VCR::Errors::UnhandledHTTPRequestError:
    ================================================================================
    An HTTP request has been made that VCR does not know how to handle:
      GET /companies/123

    VCR is currently using the following cassette:
      - /companies/index/1_1.yml
      - :record => :none
      - :match_requests_on => [:method, :uri, :query, :body]

    Under the current configuration VCR can not find a suitable HTTP interaction
...
```

It's always suggested to run tests having the `test.log` visible. In Linux/Mac OS, we can do this by running:

```bash
$ tail -f log/test.log
```

So, it's time to run the test again but recording the cassette with the `all` mode:

```bash
$ RECORD=all rspec spec/requests/companies/settings/billings_spec.rb
.....
```

In case the test fails, probably our functionality is not working correctly or our test is not correct.
In case our test passes, the cassette will be recorded with all the interactions done.

```bash
$ RECORD=all bundle exec rspec spec/requests/accounts_spec.rb
.....
```

Now that the cassete is recorded, we need to make sure that the test passes when replaying interactions from it.

```bash
$ bundle exec rspec spec/requests/accounts_spec.rb
.....
```

If test run green, then it's time to push our branch, expect a green build from TC and create the PR! That's it :)

## Factory Girl
[factory_girl](https://github.com/thoughtbot/factory_girl) is a fixtures replacement with a straightforward definition syntax that allows us to create more readable tests.
### Building Factories
factory_girl supports different build strategies:
* ```build```
* ```create```
* ```attributes_for```
* ```build_stubbed```

```ruby
# Returns an Account instance that is not saved to the DB.
account = build(:account)

# Returns a saved Account instance.
account = create(:account)

# Returns a hash of attributes that can be used to build an Account instance.
attrs = attributes_for(:account)

# Returns an object with all defined attributes stubbed out.
account_stub = build_stubbed(:account)
```

No matter which strategy is used, it's possible to override the defined attributes by passing a hash:

```ruby
# Build an Account instance and override it's name and languages.
account = build(:account, name: 'Account Name', languages: build_list(:language, 2))
```

Note that ```build_list``` and ```create_list``` methods allow you to build/create lists of factories as well. Just remember to pass the number of instances as the second parameter.

Check [Factory Girl's GitHub](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md) page for more information.

## Factories Associations
It's possible to set up associations within factories. If the factory name is the same as the association name, the factory name can be left out.

```ruby
factory :revenue do
  contract
end
```

You can also specify a different factory or override attributes:

```ruby
factory :comment do
  association :user, factory: :employee, name: 'John'
end
```

The behavior of the association method varies depending on the build strategy used for the parent object:

```ruby
# Builds and saves a Comment and a User
comment = create(:comment)
comment.new_record?      # => false
comment.user.new_record? # => false

# Builds and saves a User, and then builds but does not save a Comment
comment = build(:comment)
comment.new_record?      # => true
comment.user.new_record? # => false
```

To not save the associated object, specify ```strategy: :build``` in the factory:

```ruby
factory :comment do
  association :user, factory: :employee, strategy: :build
end
```

Check [Factory Girl's GitHub](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md#associations) page for more information.

### Inheritance
You can create multiple factories for the same class without repeating common attributes by nesting factories.

```ruby
factory :applied_payment do
  amount 20

  factory :credit_payment do
    type 'CreditCard'
    credit_card_type 'Visa'
    credit true
  end

  factory :check_payment do
    type 'Check'
    credit false
  end
end
```

Check [Factory Girl's GitHub](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md#inheritance) page for more information.

### Callbacks
factory_girl makes available four callbacks for injecting some code:

* ```after(:build)```   - called after a factory is built (via ```build```, ```create```)
* ```before(:create)``` - called before a factory is saved (via ```create```)
* ```after(:create)```  - called after a factory is saved (via ```create```)
* ```after(:stub)```    - called after a factory is stubbed (via ```build_stubbed```)

For example, in Axiom we might ask a User for his roles. Instead of making a request to fetch those roles, we might as well stub them.

```ruby
factory :revenue_user do
  after(:build, :create) do |user|
    user.stub(:roles).and_return [build(:gaap_revenue_role)]
  end
end
```

Check [Factory Girl's GitHub](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md#callbacks) page for more information.

### Suggestions when using factories
#### Use just enough data
Try leaving only required data inside your factory. An unexpected bug could be introduced by putting unnecessary data inside the factory.

Consider the following example:

```ruby
factory :user do
  name 'John'
  admin true  # No model validation on this attribute
end
```

In your tests, you might need to build a basic user (not an admin). In this case, adding admin role to a user in the factory definition by default might result in a failing spec. If you need an admin user for your tests, you have two options:

* You can use factory inheritance and define a factory for the admin:
```ruby
factory :user do
  name 'John'

  factory :admin_user do
    admin true
  end
end
```
* You can pass the role as a param when building the factory: ```admin_user = build(:user, admin: true)```

#### Build over create
At times, you don't need to use the create method. Since it saves to the DB, it adds overhead to the test and it can slow down our suite.

Try using ```build``` or ```build_stubbed``` for tests that don't need to be written to the DB, tests that don't do queries, or tests that use stubs for abstracting away the complexity of the queries.

When using ```ActiveService``` gem, data is being stored on an external DB. Thus, when writing model and helper specs, we should avoid the creation of factories as much as possible.

As long as we know that we don't need to create a resource, we should try building it instead. For example, when testing a model method:

```ruby
# app/models/employee.rb
class User
...
  def full_name
    [first_name, last_name].join(' ')
  end
end

# spec/model/employee_spec.rb
describe Employee do
  subject(:employee) { build(:employee, first_name: 'John', last_name: 'Doe') }

  describe '#full_name' do
    its(:full_name) { should eq('John Doe') }
  end
end
```

#### Testing explicit data
Test expectations need to be explicit in order to provide useful information on the test. This means that the things that you want to test should be set in the test files, rather than rely on our factory.

This way, in case there are test failures, you can easily notice the failure by seeing the error message instead of just a stack trace of code.

```ruby
# BAD EXAMPLE
factory :operation do
  ...
  description 'Allows to search and view teams and team members'
end

describe 'Operation search pages' do
  let(:operation)   { create(:operation) }

  context 'when searching' do
    before do
      visit operations_path
      fill_in 'searchtext', with: operation.id
      click_on 'Search'
    end

    ....
    it { should have_content("#{operation.description}") }
  end
end

# GOOD EXAMPLE
describe 'Operation search pages' do
  let(:description) { 'Allows to search and view teams and team members' }
  let(:operation)   { create(:operation, description: description) }

  context 'when searching' do
    before do
      visit operations_path
      fill_in 'searchtext', with: operation.id
      click_on 'Search'
    end

    ....
    it { should have_content(description) }
  end
end
```

#### Explicit Time Based Testing
The relative time helpers from Rails such as: ```1.second.ago```, ```DateTime.now```, or other helpers, can cause split-second test inconsistencies when used to assert time-related data. To avoid this, try to manually specify the time.

Consider the following example:
```ruby
factory :batch_job do
  scheduled_at Date.parse('2015-01-09')
end
```

If you prefer using the relative time helper, consider using tools like [ActiveSupport TimeHelpers](http://api.rubyonrails.org/classes/ActiveSupport/Testing/TimeHelpers.html) or some gem.

When setting dates for factories attributes, try always using ```DateTime``` and ```Time``` objects instead of plain strings. That way you avoid having to stick to the same date format when doing expectations:

```ruby
before { json = JSON.parse(batch_job_response) }

it 'should start today' do
  expect(Date.parse(json.scheduled_at)).to eq(Date.today)
end
```

#### Define valid factories
When defining factories you should take into account that the result should always be valid. Otherwise, when writing tests we will not be assured that we are using valid data in the first place.

Consider the following example:
```ruby
class User do
  validates :role, presence: true
end

factory :user do
  name 'John'
end

u = build(:user)
u.valid?  # => false
```

You can use factories in your rails console if you want to play with them:
```ruby
$: bundle exec rails c
Loading development environment (Rails 4.0.2)
irb(main):001:0> require 'factory_girl'
=> true
irb(main):002:0> FactoryGirl.find_definitions
=> ["/axiom_cloud_dev/factories", "/axiom_cloud_dev/test/factories", "/axiom_cloud_dev/spec/factories"]
irb(main):003:0> user = FactoryGirl.build(:site_user)
=> #<SiteUser(users) id=nil company_name=nil email=nil first_name="John" last_name="Doe" phone=nil manager?=false>
```

### More information
* [Factory Girl GitHub](https://github.com/thoughtbot/factory_girl)
* [The RSpec Book](http://www.amazon.com/The-RSpec-Book-Behaviour-Development/dp/1934356379)

## RSpec 3

TODO: what it is and breaking changes…

### New syntax
RSpec was created in 2005 by Steven Baker, and it was inspired by BDD practices. Despite syntactic details have evolved since the original version of RSpec, the basic premise remains. We use RSpec to write executable examples of the expected behavior of a small bit of code in a controlled context.

One of the goals of BDD is getting the words right, to be expressive. RSpec 2 proposed the language of expectations over assertions by introducing the `should` matcher:

```rails
result.should eq(5)
```

There have been major changes on the syntax from version 2 to 3. RSpec 3 encourages tests to be even more expressive by using the `expect` matcher instead:

```rails
expect(result).to eq(5)
```

Following we’ll go over some of the most notorious changes. RSpec 3 is backwards compatible with many of these examples. However, we encourage people to use the new syntax instead.

#### Standard Expectations
| Old syntax | New syntax |
|----------------|-----------------|
| `object.should matcher` | `expect(object).to matcher` |
| `object.should_not matcher` | `expect(object).not_to matcher` |

#### One-liner Expectations
| Old syntax | New syntax |
|----------------|-----------------|
| `it { should matcher }` | `it { is_expected.to matcher }` |
| `it { should_not matcher }` | `it { is_expected.not_to matcher }` |

`is_expected.`to is designed for the consistency with the `expect` syntax. However, the one-liner `should` is still not deprecated in RSpec 3.0 and available even if the `should` syntax is disabled with `RSpec.configure`.

#### Operator matchers
| Old syntax | New syntax |
|----------------|-----------------|
| `1.should == 1` | `expect(1).to eq(1)` |
| `1.should < 2` | `expect(1).to be < 2` |
| `Integer.should === 1` | `expect(Integer).to be === 1` |
| `'string'.should =~ /^str/` | `expect('string').to match(/^str/)` |
| `[1, 2, 3].should =~ [2, 1, 3]` | `expect([1, 2, 3]).to match_array([2, 1, 3])` |

See [(Almost) All Matchers Are Supported - RSpec's New Expectation Syntax](http://rspec.info/blog/2012/06/rspecs-new-expectation-syntax/#almost-all-matchers-are-supported) for more information.

#### Boolean matchers
| Old syntax | New syntax |
|----------------|-----------------|
| `expect(object).to be_true` | `expect(object).to be_truthy` |
| `expect(object).to be_false` | `expect(object).to be_falsey` |

* `be_true` matcher passes if expectation subject is truthy in conditional semantics. (i.e. all objects except `false` and `nil`)
* `be_false` matcher passes if expectation subject is falsey in conditional semantics. (i.e. `false` or `nil`)
* `be_truthy` and `be_falsey` matchers are renamed version of `be_true` and `be_false` and their behaviors are same.
* `be true` and `be false` are not new things. These are combinations of be matcher and boolean literals. These pass if expectation subject is exactly equal to boolean value.
 
#### `be_close` matcher
| Old syntax | New syntax |
|----------------|-----------------|
| `expect(1.0 / 3.0).to be_close(0.333, 0.001)` | `expect(1.0 / 3.0).to be_within(0.001).of(0.333)` |

#### `have(n).items` matcher
| Old syntax | New syntax |
|----------------|-----------------|
| `expect(collection).to have(3).items` | `expect(collection.size).to eq(3)` |
| `expect(collection).to have_exactly(3).items` | `expect(collection.size).to eq(3)` |
| `expect(collection).to have_at_least(3).items` | `expect(collection.size).to be >= 3` |
| `expect(collection).to have_at_most(3).items` | `expect(collection.size).to be <= 3` |
| `collection.should have(3).items` | `collection.should have(3).items` |

##### Note about `expect(model).to have(n).errors_on(:attr)`
`expect(model).to have(n).errors_on(:attr)` in rspec-rails 2 consists of `have(n).items` matcher and a monkey-patch [`ActiveModel::Validations#errors_on`](https://github.com/rspec/rspec-rails/blob/v2.14.2/lib/rspec/rails/extensions/active_record/base.rb#L34-L57). In RSpec 2 the monkey-patch was provided by rspec-rails, but in RSpec 3 it's extracted to rspec-collection_matchers along with `have(n).items` matcher. So if we convert it to `expect(model.errors_on(:attr).size).to eq(2)` without rspec-collection_matchers, it fails with error `undefined method 'error_on' for #<Model ...>`.

Technically it can be converted to:

```rails
model.valid?
expect(model.errors[:attr].size).to eq(n)
```

#### One-liner expectations with `have(n).items` matcher
<table>
  <thead>
    <tr>
      <th>Old syntax</th>
      <th>New syntax</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><pre>it { should have(3).items }</pre></td>
      <td>
        <pre>
it 'has 3 items' do
  expect(subject.size).to eq(3)
end
        </pre>
      </td>
    </tr>
    <tr>
      <td><pre>it { should have_at_least(3).players }</pre></td>
      <td>
        <pre>
it 'has at least 3 players' do
  expect(subject.players.size).to be >= 3
end
        </pre>
      </td>
    </tr>
  </tbody>
</table>

#### Expectations on block
| Old syntax | New syntax |
|----------------|-----------------|
| `lambda { do_something }.should raise_error` | `expect { do_something }.to raise_error` |
| `proc { do_something }.should raise_error` | |
| `-> { do_something }.should raise_error` | |
| `expect { do_something }.should raise_error` | |

#### Expectations on attribute of subject with `its`
<table>
  <thead>
    <tr>
      <th>Old syntax</th>
      <th>New syntax</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <pre>
describe 'example' do
  subject { { foo: 1, bar: 2 } }
  its(:size) { should == 2 }
end
        </pre>
      </td>
      <td>
        <pre>
describe 'example' do
  subject(:object) { { foo: 1, bar: 2 } }

  describe '#size' do
    subject { object.size }
    it { is_expected.to eq(2) }
  end
end
        </pre>
      </td>
    </tr>
  </tbody>
</table>

Note that this conversion is a sort of first-aid and ideally the expectations should be rewritten to be more expressive by yourself. For further information read (this post)[https://gist.github.com/myronmarston/4503509].

#### Negative error expectations with specific error
<table>
  <thead>
    <tr>
      <th>Old syntax</th>
      <th>New syntax</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <pre>
expect { do_something }.not_to raise_error(SomeErrorClass)
expect { do_something }.not_to raise_error('message')
expect { do_something }.not_to raise_error(SomeErrorClass, 'message')
lambda { do_something }.should_not raise_error(SomeErrorClass)
        </pre>
      </td>
      <td>
        <pre>
expect { do_something }.not_to raise_error
        </pre>
      </td>
    </tr>
  </tbody>
</table>

#### Message expectations
| Old syntax | New syntax |
|----------------|-----------------|
| `object.should_receive(:message)` | `expect(object).to receive(:message)` |
| `Klass.any_instance.should_receive(:message)` | `expect_any_instance_of(Klass).to receive(:message)` |

#### Message expectations that are actually method stubs
| Old syntax | New syntax |
|----------------|-----------------|
| `object.should_receive(:message).any_number_of_times` | `allow(object).to receive(:message)` |
| `object.should_receive(:message).at_least(0)` | |
| `Klass.any_instance.should_receive(:message).any_number_of_times` | `allow_any_instance_of(Klass).to receive(:message)` |
| `Klass.any_instance.should_receive(:message).at_least(0)` | |

#### Method stubs
| Old syntax | New syntax |
|----------------|-----------------|
| `object.stub(:message)` | `allow(object).to receive(:message)` |
| `object.stub!(:message)` | |
| `object.stub_chain(:foo, :bar, :baz)` | `allow(object).to receive_message_chain(:foo, :bar, :baz)` |
| `Klass.any_instance.stub(:message)` | `allow_any_instance_of(Klass).to receive(:message)` |
| `object.unstub(:message)` | `allow(object).to receive(:message).and_call_original` |
| `object.unstub!(:message)` | |

For further reading see:
* (RSpec's new message expectation syntax)[http://www.teaisaweso.me/blog/2013/05/27/rspecs-new-message-expectation-syntax/]
* (Bring back stub_chain (receive_message_chain) · rspec/rspec-mocks)[https://github.com/rspec/rspec-mocks/issues/464]

#### Test double aliases
| Old syntax | New syntax |
|----------------|-----------------|
| `stub('something')` | `double('something')` |
| `mock('something')` ||

For further reading check (here)[https://gist.github.com/myronmarston/6576665]

#### Pending examples
<table>
  <thead>
    <tr>
      <th>Old syntax</th>
      <th>New syntax</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <pre>
describe 'example' do
  it 'is skipped', :pending => true do
    # This won't be run
  end

  pending 'is skipped' do
    # This won't be run
  end

  it 'is skipped' do
    pending
    # This won't be run
  end

  it 'is run and expected to fail' do
    pending do
      # This will be run and expected to fail
    end
  end
end        </pre>
      </td>
      <td>
        <pre>
describe 'example' do
  it 'is skipped', :skip => true do
    # This won't be run
  end

  skip 'is skipped' do
    # This won't be run
  end

  it 'is skipped' do
    skip
    # This won't be run
  end

  it 'is run and expected to fail' do
    pending # #pending with block is no longer supported
    # This will be run and expected to fail
  end
end
        </pre>
      </td>
    </tr>
  </tbody>
</table>

#### Custom matcher DSL
<table>
  <thead>
    <tr>
      <th>Old syntax</th>
      <th>New syntax</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <pre>
RSpec::Matchers.define :be_awesome do
  match_for_should { }
  match_for_should_not { }
  failure_message_for_should { }
  failure_message_for_should_not { }
end        </pre>
      </td>
      <td>
        <pre>
RSpec::Matchers.define :be_awesome do
  match { }
  match_when_negated { }
  failure_message { }
  failure_message_when_negated { }
end
        </pre>
      </td>
    </tr>
  </tbody>
</table>

For further reading see: (Expectations: Matcher protocol and custom matcher API changes - The Plan for RSpec 3)[http://rspec.info/blog/2013/07/the-plan-for-rspec-3/#expectations-matcher-protocol-and-custom-matcher-api-changes]

#### Implicit spec types in `rspec-rails`
<table>
  <thead>
    <tr>
      <th>Old syntax</th>
      <th>New syntax</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <pre>
# In spec/models/some_model_spec.rb
RSpec.configure do |rspec|
end

describe SomeModel do
end        </pre>
      </td>
      <td>
        <pre>
RSpec.configure do |rspec|
  # rspec-rails 3 will no longer automatically infer an example group's spec type
  # from the file location. You can explicitly opt-in to the feature using this
  # config option.
  # To explicitly tag specs without using automatic inference, set the `:type`
  # metadata manually:
  #
  #     describe ThingsController, :type => :controller do
  #       # Equivalent to being in spec/controllers
  #     end
  rspec.infer_spec_type_from_file_location!
end

describe SomeModel do
end
        </pre>
      </td>
    </tr>
  </tbody>
</table>

### Upgrading from RSpec 2 to RSpec 3
RSpec 3 includes many syntax breaking changes. However, there is a smooth way of doing the upgrade.

To assist with this process, the RSpec team developed RSpec 2.99 in tandem with RSpec 3. Every breaking change in 3.0 has a corresponding deprecation to 2.99. Rather than just giving us a generic upgrade document that describes all of the breaking changes, RSpec 2.99 gives us a detailed upgrade checklist tailored to our project.

In addition, a gem called (Transpec)[http://yujinakayama.me/transpec/] was created – an absolutely amazing tool that can automatically upgrade most RSpec suites.

#### Step by step instructions
1. Start with a green test suite in RSpec 2.X ;)
```bash
$: rspec
......................................
Finished in 0.01 seconds
40 examples, 0 failures

Randomized with seed 63479
```
2. Install RSpec 2.99
```rails
# Gemfile

...
gem 'rspec', '~> 2.99.0'
...
```
3. Check your test suite and ensure it's still green. (It should be, but we may have made a mistake -- if it breaks anything, please report a bug!). Now would be a good time to commit.
```bash
$: rspec
......................................
Finished in 0.01 seconds
40 examples, 0 failures

Randomized with seed 63479
$: git add Gemfile
$: git commit -m 'Upgrade to RSpec 2.99.0'
```
```
4. You'll notice a bunch of deprecation warnings printed off at the end of the spec run. These may be truncated for simplicity purposes. To get the full list of deprecations, you can pipe them into a file by passing the --deprecation-out path/to/file command line option.
```bash
$: rspec --deprecation-out log/deprecations.log
......................................
35 deprecations logged to log/deprecations.log

Finished in 0.01 seconds
40 examples, 0 failures

Randomized with seed 63479
```
5. If you want to understand all of what is being deprecated, it's a good idea to read through the deprecation messages. In some cases, you have choices -- such as continuing to use the have collection cardinality matchers via the extracted (`rspec-collection_matchers`)[https://github.com/rspec/rspec-collection_matchers] gem, or by rewriting the expectation expression to something like `expect(list.size).to eq(3)`.
6. Install transpec, as it will save you a heap of time. (Note that this need not go into your Gemfile: you run transpec as a standalone executable outside the context of your bundle).
```bash
$: gem install transpec
```
7. Run transpec on your project. Check `transpec --help` or the (README)[https://github.com/yujinakayama/transpec#transpec] for a full list of options.
```bash
$: transpec
Copying the project for dynamic analysis...
Running dynamic analysis with command "bundle exec rspec"...

......................................
35 deprecations warnings total

Finished in 0.01 seconds
40 examples, 0 failures

Randomized with seed 63479

Gathering the spec suite data...

Converting spec/models/car.rb
Converting spec/models/plain.rb
...
```
8. Run the test suite (it should still be green but it's always good to check!) and commit.
9. If there are any remaining deprecation warnings (`transpec` doesn't quite handle all of the warnings you may get), deal with them.
10. Once you've got a deprecation-free test suite running against RSpec 2.99, you're ready to upgrade to RSpec 3. Install RSpec 3.
```rails
# Gemfile

...
gem 'rspec', '~> 3.0'
...
```
11. Run your test suite. It should still be green.
12. Run `transpec` a second time. There are some changes that `transpec` is only able to make when your project is on RSpec 3.
13. Commit!

In case we are using RSpec on a Rails project we are probably using the (`rspec-rails`)[https://github.com/rspec/rspec-rails] gem. This gem used to have all the spec configuration on the `spec_helper.rb` file for RSpec 2. RSpec 3 introduces a new configuration file called `rails_helper.rb`.

For step by step instructions we can check (here)[https://relishapp.com/rspec/rspec-rails/docs/upgrade#default-helper-files].

### More information
* (Upgrading from RSpec 2 to RSpec 3)[http://rspec.info/upgrading-from-rspec-2/]
* (Transpec)[http://yujinakayama.me/transpec/]
