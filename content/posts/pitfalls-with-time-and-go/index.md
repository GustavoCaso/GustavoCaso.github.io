+++
title =  "Pitfalls with time and Go"
date = 2025-09-16
tags = ["go", "dependencies"]
+++

I'm a macOS user, and I consistently find `launchd` confusing and challenging to use.

With so many process managers available, I decided to build my own instead of learning yet another one. 😁

The concept is simple: use a configuration file—either YAML or JSON—that is human-readable and easy to modify.

Here's an example configuration:

```yaml
programs:
  - name: backup services
    interval: 4h
    command: /usr/bin/rsync
    args:
	  - -av
      - --delete
      - pi5:services/
      - $HOME/Documents/services
```

There are more configuration options available. If you're interested, check out the messeks [repo](https://github.com/GustavoCaso/meeseeks).

This configuration backs up the `services` folder every four hours.

To implement this in Go, you can use `time.Ticker` like so:

```go
// Assume config has been parsed
// and `program` is set up for execution
ticker := time.Ticker(config.Interval)

for {
  select {
    case <-ctx.Done():
      return
    case <-ticker.C:
      program.Execute()
  }
}
```

This works as expected 🎉

But what if the computer goes to sleep? The ticker won't necessarily fire at the right time after sleep. This is a known [issue](https://github.com/golang/go/issues/75106) in Go.

That was my first pitfall with Go and `time`.

After learning this, I decided to continue using `time.Ticker`, but instead of relying solely on it, I'd check the last execution time. If the current time exceeds the next scheduled run, I execute the program:

```go
ticker := time.Ticker(1 * time.Second)
interval := program.Interval 

for {
  select {
    case <-ctx.Done():
      return
    case <-ticker.C:
     now := time.Now()
     lastRunAt := program.lastRunAt
     
     if now.Sub(lastRunAt) > interval {
       program.Execute()
     }
  }
}
```

Now, instead of relying on `time.Ticker`, we use `time` to check if we need to execute the program. 

Now, we're using `time` to decide if the program needs to run.

However, this doesn't always work, either. After some confusion, I checked the [time package documentation](https://pkg.go.dev/time), which states:

> On some systems the monotonic clock will stop if the computer goes to sleep. On such a system, t.Sub(u) may not accurately reflect the actual time that passed between t and u. The same applies to other functions and methods that subtract times, such as [Since](https://pkg.go.dev/time#Since), [Until](https://pkg.go.dev/time#Until), [Time.Before](https://pkg.go.dev/time#Time.Before), [Time.After](https://pkg.go.dev/time#Time.After), [Time.Add](https://pkg.go.dev/time#Time.Add), [Time.Equal](https://pkg.go.dev/time#Time.Equal) and [Time.Compare](https://pkg.go.dev/time#Time.Compare). In some cases, you may need to strip the monotonic clock to get accurate results.

So `time.Now()` is also affected by system sleep.

This was my second pitfall with Go and time. 

TThe solution: strip the monotonic clock using the [Round()](https://pkg.go.dev/time#Time.Round) function, passing zero as the argument:

```go
ticker := time.Ticker(1 * time.Second)
interval := program.Interval 

for {
  select {
    case <-ctx.Done():
      return
    case <-ticker.C:
     now := time.Now().Round(0)
     lastRunAt := program.lastRunAt.Round(0)
     
     if now.Sub(lastRunAt) > interval {
       program.Execute()
     }
  }
}
```

Now, our program works correctly, even when the system sleeps, by stripping monotonic clock info from the time objects.

In summary, always check Go's documentation—they've done a great job explaining these subtleties. 

If you're curious, check out the [meeseeks](https://github.com/GustavoCaso/meeseeks) project. It's a simple process manager you can use as a standalone CLI or a Go package in your project.
