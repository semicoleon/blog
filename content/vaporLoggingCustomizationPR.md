+++
title = "Vapor Logging Customization PR Retrospective"
date = 2026-08-20

[extra]
#toc = true

[taxonomies]
tags = [
    "PR Retrospective",
    "Vapor",
    "ConsoleKit",
    "Swift"
]
+++

I've spent a lot of time working in Vapor servers over the last 6 years, and more recently have found myself building new Vapor servers in the context of cloud applications. While Vapor's default logging system worked fine, it made some choices about what (and more importantly when) to log certain information that were causing problems for me in these newer projects<!-- more -->. A coworker had already created a new version of Vapor's[^notVaporDisclaimer] `ConsoleLogger` that included a timestamp, which addressed one major issue. I needed some additional information to be logged, and at the time the only thing I could do was create *another* copy of `ConsoleLogger` and customize it to do what I needed. There was nothing wrong with that approach in principle, but I was frustrated by the fact that every minor change needed a whole new `ConsoleLogger` implementation. Most of which was inevitably just copy pasted from the original, and therefore a great place for bugs to accumulate. I kept finding myself thinking about ways to improve the situation, and eventually [authored a PR](https://github.com/vapor/console-kit/pull/182) that increased the flexibility of `ConsoleKit`'s logging features.

[^notVaporDisclaimer]: I'm calling it "Vapor's" but it actually exists in the [`ConsoleKit`](https://github.com/vapor/console-kit) repository which belongs to the Vapor orginization

# The Problem

What I wanted was the ability to customize how log messages were constructed, but without reimplementing *all* of the logic `ConsoleLogger` was using to construct the default message. A minimum viable solution should:
1. Allow an end user to include pieces of the default log message format.
2. Be able to add their own completely custom code to the process of constructing the message. Ideally without needing to be concerned with the details of the logger type itself.
3. Since we're making changes anyway, it would be nice if an end user could easily prepend or append to a log message they're mostly happy with without doing additional work to the integrate the majority of the logging code they have with the bit they'd like to add

The old `ConsoleLogger` fails these requirements before it even gets to the starting line because it supports no message customization at all. Additionally the fact that all of the implementation details are private to the library means that we can't just create a new version of `ConsoleLogger` without having to copy a bunch of code out of the library that likely has nothing to do with the customizations we want to make.

<!-- While addressing the second point might have been sufficient to get my brain to let go of the problem, a straightforward patch along those lines would still have involved a fair amount of code copying for each customization to get the majority of the default message formatting into a custom logger. Since most of my "minimum viable solution" was concerned with message construction, I elected to address point one instead. -->

Let's take a look the contents of the original `ConsoleLogger`'s `log` method to get a better idea of what kind of solutions might fit our requirements:
```swift
var text: ConsoleText = ""

if self.logLevel <= .trace {
  text += "[ \(self.label) ] ".consoleText()
}
  
text += "[ \(level.name) ]".consoleText(level.style)
  + " "
  + message.description.consoleText()

let allMetadata = (metadata ?? [:])
  .merging(self.metadata, uniquingKeysWith: { (a, _) in a })
  .merging(self.metadataProvider?.get() ?? [:], uniquingKeysWith: { (a, _) in a })

if !allMetadata.isEmpty {
  // only log metadata if not empty
  text += " " + allMetadata.sortedDescriptionWithoutQuotes.consoleText()
}

// log file info if we are debug or lower
if self.logLevel <= .debug {
  // log the concise path + line
  let fileInfo = self.conciseSourcePath(file) + ":" + line.description
  text += " (" + fileInfo.consoleText() + ")"
}

self.console.output(text)
```

This code is straightforward. There are several independent fragments of text that are appended to the final log message. While there is some control flow, it really only determines if the fragment of text it wraps will be present in the final message. There's no complex bookkeeping or interdependencies between different fragments inside the method. None of the code needs to mutate any shared state either, so reordering the sections wouldn't be a problem.

# Solutions
What immediately jumped out to me about this code was the lack of complex interdependencies. Each little section handles outputting (or not outputting) a self contained little fragment of the final log message. None of the sections needed to do complex control flow based on the what other fragments were doing, and for the most part the code is just doing some basic string formatting based on the metadata for the logged message. 

One possible way forward would be to replace the fragment producing sections with methods and allow the end user to call them however the liked. That would be simple, but it wouldn't compose very well. The overall shape of the logging method would be the same, and it would be easy to accidentally break things in a way that makes it difficult for an end user to prepend or append to a default message. If a simple method per fragment producing section doesn't quite meet our requirements, what's the next place to look for a solution? Types, of course!

# Going to pieces, er... fragments

My solution was to create a custom type for each fragment producer which conformed to a new `LoggerFragment` protocol. The goal of `LoggerFragment` is to allow each bit of code representing a part of the default log message to become a type that can be combined with others freely by users of the library. Users should also be able to implement their own fragments and have them work in concert with the default ones without much fuss. Using a protocol makes it easy to define things like combinators as provided methods on the protocol, which is a nice ergonomics advantage over a solution based on defining methods on a single logger type[^methodSolutionErgonomics]. The protocol currently only contains one (non-defaulted) method:
[^methodSolutionErgonomics]: Since Swift doesn't allow extensions on function types, you can't chain combinators the way you can with `Sequence`s and other protocol based combinators. That can hurt discoverability.
```swift
public protocol LoggerFragment: Sendable {
    // Leaving out the defaulted method, since it's not important to the functioning of the protocol
  
    /// Add this fragment's output to the console text.
    func write(_ record: inout LogRecord, to output: inout FragmentOutput)
}
```

The PR added some types to make this definition a little less chaotic:
- `LogRecord` which is just all of the information about the log message to render, which can be mutated if necessary. 
- `FragmentOutput` which contains the `ConsoleText` that the old logger built up in it's `log` method, plus a bit of state for managing separator insertion.
<!-- 
The protocol also provides a number of combinator methods, Not unlike `Sequence` and `Collection` which wrap the fragment the method is called with in a new fragment that performs some additional (and sometimes conditional) processing. These combinators are what allow the `LoggerFragment` system to easily support customization without having to reimplement all of the code that renders the default message. -->

Now that we have an idea of what the protocol looks like, let's take a look at what implementing it with the default logging information looks like.

## Level Logging

The old code that rendered the log level looked like this:
```swift
// ...
text += "[ \(level.name) ]".consoleText(level.style)
  + " "
  + message.description.consoleText()
// ...
```

This statement renders the log level, a separator, and then the message portion of the log record all together, but we only actually care about the first part of the expression. That section has been extracted into this new type

```swift
/// Writes the level of the logged message, and requests a separator for the next fragment.
public struct LevelFragment: LoggerFragment {
  public init() { }
  
  public func write(_ record: inout LogRecord, to output: inout FragmentOutput) {
    output += "[ \(record.level.name) ]".consoleText(record.level.style)
    output.needsSeparator = true
  }
}
```

Other than the fact that the code is in a method defined on a new type, it isn't much of a change really. The only significant change is that the separator isn't written directly. Instead we tell the output that we did write something that will require a separator if another fragment adds output after `LevelFragment` is done. This allows flexibility with separator selection, and also avoids tacking on useless trailing separators to the end of the output.


## Combinators

In addition to the basic functionality of the protocol, `LoggerFramgent` also provides methods to construct a number of different combinator types, wrapping them around the fragment the method is called on, similar to how some of the lazy `Sequence` and `Collection` extension methods work. For example, `separated` allows you to insert a separator before the fragment it's called on (though a separator will only be output if a previous fragment indicated that one is needed). The provided method on the protocol just wraps the `LogFragment` it's called on in the combinator type `SeparatorFragment`, passing along the separator text to use. That method looks like this:

```swift
/// Appends the given separator text to the output before `self`'s output, as long as a separator is needed.
///
/// If the wrapped fragment reports that it has no content, no separator will be inserted.
func separated(_ text: ConsoleText) -> SeparatorFragment<Self> {
  SeparatorFragment(text, fragment: self)
}
```

To demonstrate how this works, we'll output the log message and then a separator followed by "b" using the `and` combinator to combine two fragments (the message and the literal with its separator):
```swift
MessageFragment().and(LiteralFragment("b").separated(" "))
```
<!-- TODO: Fix -->
> [!warning]
> `LiteralFragment` does not request a separator on its own. Generally if you're trying to use `separated` to add a separator between literals you could just... use one literal containing both literals and the separator instead.

If we log a message "a" with this fragment, the result will be "a b"[^ignoringNewlines].
[^ignoringNewlines]: There will also be a newline in the output, but that's added by the logger unconditionally so I'm ignoring it here.

# Old vs. New
Now that we've gone over the general idea of how fragments look, let's compare the old message rendering code to the new default fragment provided by the public function `defaultLoggerFragment()` to see the difference!

Here's the contents of the old `ConsoleLogger`'s `log` method once again:
```swift
var text: ConsoleText = ""

if self.logLevel <= .trace {
  text += "[ \(self.label) ] ".consoleText()
}
  
text += "[ \(level.name) ]".consoleText(level.style)
  + " "
  + message.description.consoleText()

let allMetadata = (metadata ?? [:])
  .merging(self.metadata, uniquingKeysWith: { (a, _) in a })
  .merging(self.metadataProvider?.get() ?? [:], uniquingKeysWith: { (a, _) in a })

if !allMetadata.isEmpty {
  // only log metadata if not empty
  text += " " + allMetadata.sortedDescriptionWithoutQuotes.consoleText()
}

// log file info if we are debug or lower
if self.logLevel <= .debug {
  // log the concise path + line
  let fileInfo = self.conciseSourcePath(file) + ":" + line.description
  text += " (" + fileInfo.consoleText() + ")"
}

self.console.output(text)
```

Now here's `defaultLoggingFragment()`
```swift
/// A `LoggerFragment` which implements the default logger message format.
public func defaultLoggerFragment() -> some LoggerFragment {
    LabelFragment().maxLevel(.trace)
        .and(LevelFragment().separated(" ").and(MessageFragment().separated(" ")))
        .and(MetadataFragment().separated(" "))
        .and(SourceLocationFragment().separated(" ").maxLevel(.debug))
}
```

While I don't think the old code is *bad*, a side by side comparison highlights how much easier it is to add to or modify the default format without having to reimplement literally all of the logic for rendering the log message into text. If you only want to add to the beginning or end of the message, you can even call `defaultLoggerFragment()` when building your fragment. In fact that's what the public `timestampDefaultLoggerFragment()` function does!

```swift
/// A `LoggerFragment` which implements the default logger message format with a timestamp at the front.
public func timestampDefaultLoggerFragment(
    timestampSource: some TimestampSource = SystemTimestampSource()
) -> some LoggerFragment {
    TimestampFragment(timestampSource).and(defaultLoggerFragment().separated(" "))
}
```

This kind of flexibility is exactly what I was looking for when I started thinking about this issue, so I'm quite happy to see it working as intended (and the PR merged). 

# Performance
No discussion of a fundamental change to a library would be complete without talking about performance. You can [see the final test code from the PR here](https://github.com/vapor/console-kit/pull/182/changes#diff-ddedf2001071b48687d64f13844d662a876f90f717f0a416e62fa308cfa0866e). My initial measurements, made on a relatively old 2019 2.3 GHz 8-Core Intel i9 MacBook Pro[^unknownSwiftVersion], showed that my feature branch was almost twice as fast as the main branch! That would be exciting, but none of the changes I've outlined here could possibly account for a 2x speedup so.... what??? 

[^unknownSwiftVersion]: Unfortunately I don't remember exactly which version of Swift was installed at the time

If you look at the pull request, you'll see that there are a *lot* of changes that are completely unrelated to the logger fragment code. There were a number of `Sendable` issues that needed to be resolved, and `ConsoleKit` had a number of other minor changes/fixes that really needed to be made. Some of these changes didn't *need* to be part of this PR, but since I was moving the old code into different places in the repo anyway it was pretty reasonable to just make those changes here rather than in a bunch of separate PRs. One of these changes was removing this function

```swift
private func conciseSourcePath(_ path: String) -> String {
  let separator: Substring = path.contains("Sources") ? "Sources" : "Tests"
  return path.split(separator: "/")
    .split(separator: separator)
    .last?
    .joined(separator: "/") ?? path
}
```

This was necessary in older versions of Swift to get a sensible file path (i.e. one that doesn't include the entire absolute path the binary was built at) out of the `#file` magic identifier[^macroOrMagic]. The exact behavior of the `#file(path|ID)?` magic identifiers varies based on the Swift language mode, [SE-0285](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0285-ease-pound-file-transition.md) has the details on the transition plan if you're curious. The short version is that my PR came after the Vapor team had bumped the minimum Swift compiler version supported to one after the introduction of `#fileID`[^swiftLogFileId], so `ConsoleKit` no longer needed to perform this processing at run time. Perhaps unsurprisingly, given that `conciseSourcePath` splits the path and then splits the resulting array before joining the result back together, this single change accounts for the 2x speedup between main and my feature branch.

[^macroOrMagic]: `#file` and friends used to be considered "magic identifiers", but now that macros exist they're considered macros. Does it make sense to call it a macro when talking about a time before Swift had macros?
[^swiftLogFileId]: `swift-log` uses the compiler version to determine whether to use `#fileId` so we don't have to do anything else to opt in to this behavior (This appears to no longer be the case, I assume swift-log has dropped support for Swift versions before `#fileId` was supported).

Once I modified my local copy of the main branch to no longer use `conciseSourcePath` the performance measurements were *much* more in line with my expectations. 

| Change                                    | main   | logger-fragment |
| ----------------------------------------- | ------ | --------------- |
| Original                                  | 0.9s   | 0.55s           |
| Removed `conciseSourcePath` from main     | 0.45s  | 0.55s           |
| Discarding output text from `TestConsole` | 0.356s | 0.483s          |
| Increasing amount of metadata             | 0.491s | 0.637s          |

> [!note] Updated performance numbers
>
> Out of curiosity, when writing this post I re-ran the performance tests on the same MacBook, and my Windows PC[^swiftWindows]. The results I got in both cases were not what I was expecting. On Windows I was seeing no consistent performance difference, and on the MacBook I was consistently seeing the `LoggerFragment` branch outperform the main branch from before it was merged. I haven't had time to dig into that any deeper, so it's possible I made a mistake somewhere. That being said, it also seems plausible that compiler improvements have legitimately made the slight performance deficit of the `LoggerFragment` branch disappear. I hope to have time to dig into that a little further soon.

Those numbers weren't bad (in fact they were still significantly faster than the original logger due to `conciseSourcePath`) so the Vapor team decided to merge the PR. I'm quite happy with how it turned out, and that the change was well received by the Vapor team.

[^swiftWindows]: Swift actually works on Windows now! It's weird!!
<!-- Out of curiosity, I re-ran the numbers on my Windows PC (Ryzen 9 5900X, Swift 6.3.2) and found that the numbers were more or less identical between the feature branch and the commit to main just before the branch was merged[^commitHash] (~0.331s average). There are a lot of possible explanations, but I imagine improvements to compiler optimizations are involved in some way.

[^commitHash]: `7d0898ed481e1855ec549924ab701bdc6f754b18` is the commit hash

Re-ran on MBP Swift 6.3.2 
tag 4.7.0 w/ conciseSourcePath removed & AnySendableHashable replacement in Console 0.696s
tag 4.11.0 no changes 0.652s
tag 4.8.0 no changes 0.638s -->