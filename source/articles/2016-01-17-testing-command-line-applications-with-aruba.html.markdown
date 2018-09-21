---
layout: post
title: "Testing Comamnd-Line Applications with Aruba"
date: 2016-01-17 23:08:23 +0100
comments: true
tags: [ruby, test]
---

In the last couple of weeks I have been working, with a little project of my own. I always love Command Line Tools, I don't know what they have but using them make feel more like a Hackers or someone that actually know what he is doing.

So I decided to build one, with the help of a gem called [Thor](https://github.com/erikhuda/thor), which by the way is a great gem, that help you build your CLI really easy.

READMORE

[Codewars](http://www.codewars.com/) provided a great service by letting us the programmer improve our coding skills, I decide to build a CLI to interact with it.

Probably I will write a post in future about creating a CLI but for now I want to focus on test.

My project is called [Codewars_Cli](https://github.com/GustavoCaso/codewars_cli), any one interested, is open for suggestions and pull requests.


### Testing

In my day to day I use [Rspec](https://github.com/rspec) for testing, so the idea of using [Cucumber](https://github.com/cucumber) really strike as a chance to learn a little more about this testing framework.

Wihch is focus on Behavior Driven Development.

### Aruba

Aruba is a great extension either for Rspec and Cucumber that makes testing command-lines tools meanigful, easy and fun.
It makes easy to manipulate the file system and the process envarioment, automatically reset state of file system.

### Configuration

First inside the `features/support` folder create a `env.rb` to load `aruba` and `your_application`

```ruby
require 'your_library_under_test'
require 'aruba/cucumber'
```

Now we are ready to start testing our application. By default `Aruba` will create a folder `tmp/aruba` where will perform its operations, you can change that in the `env.rb` with some hooks that `Aruba` provide.

```ruby
Before do
  @dirs = ['tmp/my_work']
end
```
For a list of all the available configuration [README](http://www.rubydoc.info/github/cucumber/aruba/master/frames)

I'm going to focus on testing my config commands. Normally this command will involve manipulating a config file store in your `HOME` folder.

If we don't want our test to actually modify that file, aruba can mock our config file as well.

`Feature` test example from my small project.

```ruby
Feature: Hability to store configuration settings
  Background:
    Given a mocked home directory
  Scenario: Setup the apikey
    Given the config file do not exists
    When I run `codewars config api_key test_api`
    Then the output should contain "Updating config file with api_key: test_api"
    And the config file contain:
      """
      :api_key: test_api
      :language: ''
      :folder: ''
      """
```

As you can see is really easy with `aruba` to achieve that.

Also I have declare some `step_definitions`:
```ruby
Given(/^the config file with:$/) do |string|
  step 'a file "~/.codewars.rc.yml" with:', string
end

Given(/^the config file do not exists$/) do
  step 'a file "~/.codewars.rc.yml" does not exist'
end

Then(/^the config file contain:$/) do |string|
  step 'the file "~/.codewars.rc.yml" should contain:', string
end
```

To help me test. There are many more methods that `aruba` bring us like `cd` into folder, create files, delete them etc... Here is another great resource about it [Getting Started](http://www.relishapp.com/cucumber/aruba/v/0-11-0/docs/getting-started)

### Stubbing External Services

In my case my application depends on an external service, so I don't want my test to actually make any real request, that would result in a slower test suite.

I usually use [webmock](https://github.com/bblimke/webmock) or [VCR](https://github.com/vcr/vcr) to stub my requests.

Aruba execute the command under test in a new child process, that it makes them slower and complicated to mock components. But there is a way to make the test run in the same process.

First we have to wrap our application, and execute the wrapper instead.

This is my `bin` file before introducing the wrapper class:
```ruby
!/usr/bin/env ruby -U

require 'codewars_cli'

CodewarsCli::Cli.start(ARGV)
```

After:
```ruby
#!/usr/bin/env ruby -U

lib = File.expand_path('../../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)

require 'codewars_cli/runner'
CodewarsCli::Runner.new(ARGV.dup).execute!
```

**CodewarsCli::Runner** the wrapper class has to respond to `execute!`

Now we have to modify the the `env.rb` inside our `features/support` folder.

```ruby
require 'codewars_cli/runner'
require 'aruba/cucumber'
require 'aruba/in_process'

Aruba.configure do |config|
  config.command_launcher = :in_process
  config.main_class = CodewarsCli::Runner
end
```

We have added `aruba/in_process` and pass some configuration to tell `aruba` to run the test in the same process.

Now to create our wrapper **CodewarsCli::Runner**

```ruby
require 'codewars_cli/cli'

module CodewarsCli
  class Runner
    # Allow everything fun to be injected from the outside while defaulting to normal implementations.
    def initialize(argv, stdin = STDIN, stdout = STDOUT, stderr = STDERR, kernel = Kernel)
      @argv, @stdin, @stdout, @stderr, @kernel = argv, stdin, stdout, stderr, kernel
    end

    def execute!
      exit_code = begin
        # Thor accesses these streams directly rather than letting them be injected, so we replace them...
        $stderr = @stderr
        $stdin = @stdin
        $stdout = @stdout

        # Run our normal Thor app the way we know and love.
        CodewarsCli::Cli.start(@argv)

        # Thor::Base#start does not have a return value, assume success if no exception is raised.
        0
      rescue StandardError => e
        # The ruby interpreter would pipe this to STDERR and exit 1 in the case of an unhandled exception
        b = e.backtrace
        @stderr.puts("#{b.shift}: #{e.message} (#{e.class})")
        @stderr.puts(b.map{|s| "\tfrom #{s}"}.join("\n"))
        1
      rescue SystemExit => e
        e.status
      ensure
        # TODO: reset your app here, free up resources, etc.
        # Examples:
        # MyApp.logger.flush
        # MyApp.logger.close
        # MyApp.logger = nil
        #
        # MyApp.reset_singleton_instance_variables

        # ...then we put the streams back.
        $stderr = STDERR
        $stdin = STDIN
        $stdout = STDOUT
      end

      # Proxy our exit code back to the injected kernel.
      @kernel.exit(exit_code)
    end
  end
end
```

Now we can use `VCR` to mock our external services.

We can do that with hooks example.

I have created `features/support/webmock.rb` file where I will store all my hooks.

```ruby
require 'webmock/cucumber'

CODEWARS_BASE = 'https://www.codewars.com'
CODEWARS_API = '/api/v1'

Before('@stub_user_response') do
  api_key = 'fake_api'
  stub_get("/users/GustavoCaso")
    .with(
      headers: { Authorization: api_key }
    ).to_return(json_response 'user.json')
end

def stub_get(url)
  stub_request(:get, "#{CODEWARS_BASE}#{CODEWARS_API}#{url}")
end

def json_response(file, status=200)
  {
    body: fixture(file),
    status: status,
    headers: { content_type: 'application/json; charset=utf-8' }
  }
end

def fixture(file)
  File.new(File.join(fixture_path,file))
end

def fixture_path
  File.expand_path('../../fixtures', __FILE__)
end
```

And inside the features test we can use this hook.

```ruby
Feature: Display the user information in the terminal
  Background:
    Given a mocked home directory
    Given the config file with:
      """
      :api_key: 'fake_api'
      :language: ''
      :folder: ''
      """

  @stub_user_response
  Scenario: Passing a valid username prints a correct message
    When I run `codewars user GustavoCaso`
    Then the output should contain "Displaying information about GustavoCaso"
```

This is all I have learn about `Aruba` for testing your applications, I think is a great tool, that remove us from the pain of having to create to many `step_definitions` and let us focus on testing our application. I know there is to much to learn about `Aruba` and testing in general and if you have any questions or comments about the subject please do not hesitate.
