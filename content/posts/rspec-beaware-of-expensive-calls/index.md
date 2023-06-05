+++
title = 'RSpec. Beware of expensive calls'
date = 2023-06-01
tags = ["ruby", "rspec", "test"]
+++

[Rspec](https://rspec.info/) is an excellent testing framework for Ruby. It allows you to use a DSL to test your ruby code.

Generally, on a test, we will need some configuration that might have to run before or after the test.

When writing tests, we often strive to write as few assertions per spec as possible -- sometimes even just one. But, as with all best practices, we should beware of the trade-offs we're making.

That was the case for some specs I maintain as part of my daily job.

Here's a simplified version of these specs as an example.

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
      it { expect(result.fetch('HTTP.status_code')).to eq('200') }
      it { expect(result.status).to eq(0) }
    end

    before do
      response
    end

    describe 'GET request' do
      subject(:response) { get URL, params, env }

      context 'with certain values do
        it { is_expected.to be_ok }

        it_behaves_like 'a GET 200 response'
      end
    end
  end
end
```

The subject for this spec is a GET request to an HTTP service.

Things look great once the spec is reviewed and merged in the main branch. We are confident our code has test coverage, and if we ever introduce a regression, our specs should be able to spot it. We continue to work on new features, so more and more test cases have to be tested, and our spec files grow.

After a spec gets created in some way, it's common for other developers to copy the same pattern as more specs get added, especially within the same file.

Gradually, our test suite takes longer and longer to run, and we need to figure out why.

One day after waiting for far too long for our test suite to complete, I decided that I was going to find out why our tests were so slow.

Since I have been working with the specs for quite a while, I did not consider the issue being on the spec definitions but rather on the underlying code that the specs were testing. After some exploration that led to nothing, I decided to check the spec definitions.

The issue is with how we structure our specs. The HTTP request our spec is doing is slow. That, in isolation, is not the issue, but the fact that we are executing the same request for each `it` block makes the test run slow.

So a change from:

```ruby
shared_examples 'a GET 200 response' do
  it { expect(result.fetch('http.method')).to eq('GET') }
  it { expect(result.fetch('HTTP.status_code')).to eq('200') }
  it { expect(result.status).to eq(0) }
end
```

to

```ruby
shared_examples 'a GET 200 response' does
  it do
    expect(result.fetch('http.method')).to eq('GET')
    expect(result.fetch('HTTP.status_code')).to eq('200')
    expect(result.status).to eq(0)
  end
end
```

After the change, we execute the slow HTTP request once rather than thrice.

Our test suite went from taking 33 minutes to 12 minutes. That is 67% faster :tada:

![Before change](./rspec_before.png)

![After change](./rspec_after.png)

I want to share some advice with everyone. Simply following a pre-existing pattern without comprehending its underlying mechanisms can lead to compromises. What was effective in the past may not be effective in the future.

Here is the [PR](https://github.com/DataDog/dd-trace-rb/pull/2843) with my changes to improve the test suite performance.

**Update, 5th Jun 2023**: [Ivo Anjo](https://ivoanjo.me/) Reviewed the entry and offered some suggestions.
