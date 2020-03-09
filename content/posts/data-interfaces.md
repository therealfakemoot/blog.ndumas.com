---
title: "Data Interfaces"
date: 2019-02-06T14:58:22Z
draft: false
toc: false
images:
tags:
  - go
---

# interfaces

I'm a fan of Go's interfaces. They're really simple and don't require a lot of legwork.

{{< highlight go "linenos=table">}}
type Mover interface {
  func Move(x, y int) (int, int)
}

type Dog struct {
  Name string
}

func (d Dog) Move(x, y int) (int, int) {
  return x, y
}
{{< / highlight >}}

Dog is now a Mover! No need for keywords like `implements`. The compiler just checks at the various boundaries in your app, like struct definitions and function signatures.

{{< highlight go "linenos=table">}}
type Map struct {
  Actors []Mover
}

func something(m Mover, x,y int) bool {
  // do something
}
{{< / highlight >}}

# Functionality vs Data

This is where things get tricky. Interfaces describe *functionality*. What if you want the compiler to enforce the existence of specific members of a struct? I encountered this problem in a project of mine recently and I'll use it as a case study for a few possible solutions.

## Concrete Types

If your only expectation is that the compiler enforce the existence of specific struct members, specifying a concrete type works nicely.

{{< highlight go "linenos=table">}}
type Issue struct {
        Key     string
        Title   string
        Created time.Time
        Updated time.Time
        Body    string
        Attrs   map[string][]string
}

type IssueService interface {
        Get() []Issue
}
{{< / highlight >}}

There's a few benefits to this. Because Go will automatically zero out all members of a struct on initialization, one only has to fill in what you explicitly want or need to provide. For example, the `Issue` type may represent a Jira ticket, or a Gitlab ticket, or possibly something as simple as lines in a TODO.txt file in a project's root directory.

In the context of this project, the user provides their own functions which "process" these issues. By virtue of being responsible for both the production and consumption of issues, the user/caller doesn't have to worry about mysteriously unpopulated data causing issues.

Where this falls apart, though, is when you want to allow for flexible/arbitrary implementations of functionality while still enforcing presence of "data" members.

## Composition

I think the correct solution involves breaking your struct apart: you have the `Data` and the `Thinger` that needs the data. Instead of making `Issue` itself have methods, `Issue` can have a member that satisfies a given interface. This seems to offer the best of both worlds. You don't lose type safety, while allowing consumers of your API to plug-and-play their own concrete implementations of functionality while still respecting your requirements.

{{< highlight go "linenos=table">}}
type Issue struct {
        Key     string
        Title   string
        Created time.Time
        Updated time.Time
        Body    string
        Checker IssueChecker
        Attrs   map[string][]string
}

type IssueChecker interface {
        Check(Issue) bool
}

{{< / highlight >}}
