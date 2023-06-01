+++
title = 'Rspec. Be Aware of expensive calls'
date = 2023-06-01
tags = ["ruby", "rspec", "test"]
+++

[Rspec](https://rspec.info/) is an excellent testing framework for Ruby. It allows you to use their custom DSL to test your ruby code.

Full disclosure this is not a post describing all the Rspec features. For that, please check the Rspec documentation.

As ruby developers, we follow the [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) principle (Don't Repeat Yourself). Generally, on a test, we will need some configuration that might have to run before or after the test. We will need a subject, the specific functionality we are trying to test, and other stuff.
Another mantra the ruby community follows is testing a single thing per test; doing so makes it easier to understand what the test is doing at a glance.

But as with all in life, sometimes following all mantras or best practices could lead to undesired results.

That was the case for some specs I maintain as part of my daily job.

The setup for the specs is reasonably complicated to describe here, so that I will use a simplified version.

```ruby
RSpec.describe 'Some random test' do
  before do
    # Some code needed to run before the specs
  end

  after do
    # Some code needed to run after the specs
  end

  context 'for our application' do
    let(:result) { get_result }

    shared_examples 'a GET 200 response' do
      it { expect(result.fetch('http.method')).to eq('GET') }
      it { expect(result.fetch('http.status_code')).to eq('200') }
      it { expect(result.status).to eq(0) }
    end

    before do
      response
    end

    describe 'GET request' do
      subject(:response) { get url, params, env }

      let(:url) { '/success' }
      let(:params) { {} }
      let(:headers) { {} }
      let(:env) { { 'REMOTE_ADDR' => remote_addr }.merge!(headers) }

      context 'with certain valules' do
        it { is_expected.to be_ok }

        it_behaves_like 'a GET 200 response'
      end
    end
  end
end
```

As you can see, the spec is well organized. It uses [describe and context](https://rspec.info/features/3-12/rspec-core/example-groups/basic-structure/) blocks. It uses [hooks](https://rspec.info/features/3-12/rspec-core/hooks/before-and-after-hooks/), [shared examples](https://rspec.info/features/3-12/rspec-core/example-groups/shared-examples/) and [subject](https://rspec.info/features/3-12/rspec-core/subject/one-liner-syntax/) to organize and DRY the spec.

As you can see, the subject for the spec is a GET request to an HTTP service.

Things look great once the spec is reviewed and merged in the main branch. We are confident our code has test coverage, and if we ever introduce a regression, our specs should be able to spot it. We continue to work on new features, so more and more test cases have to be tested, and our spec files grow.

When it comes to adding more tests, other developers, myself included, tend to copy what others have than before; This is called cargo cult.

Suddenly, our test suite takes longer and longer to run, and we need to figure out why.

One day after waiting for far too long for our test suite to complete, I decided that I was going to find out why our tests were so slow.

Since I have been working with the specs for quite a while, I did not consider the issue being on the spec definitions but rather on the underlying code that the specs were testing. After some exploration that led to nothing, I decided to check the spec definitions.

Our spec was doing far too many HTTP requests :sad:

So a simple change from:

```ruby
shared_examples 'a GET 200 response' do
  it { expect(result.fetch('http.method')).to eq('GET') }
  it { expect(result.fetch('http.status_code')).to eq('200') }
  it { expect(result.status).to eq(0) }
end
```

to

```ruby
shared_examples 'a GET 200 response' do
  it do
    expect(result.fetch('http.method')).to eq('GET')
    expect(result.fetch('http.status_code')).to eq('200')
    expect(result.status).to eq(0)
  end
end
```

Our test suite went from taking 33 minutes to 12 minutes. That is 67% faster :tada:

![Before change](./rspec_before.png)

![After change](./rspec_after.png)

So the advice I want to leave everyone with is that sometimes following best practices and cargo cult might lead to a poor outcome.

Here is the [PR](https://github.com/DataDog/dd-trace-rb/pull/2843) with my changes to improve the test suite performance.
