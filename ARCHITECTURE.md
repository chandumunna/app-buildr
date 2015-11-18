# Overview

It might be helpful if I discuss a popular server side build structure. In this structure, there is a command line executor that assumes the current directory has a `buildfile.ts`. This file is consumed and the contents used to initialize the build. 

## Dependencies

The metadata of every build contains a unique identifier known as a "group" that is based on DNS infrastructure, for instance `com.github.ludohenin`. This provides namespace separation between development entities since only one entity can own a given domain name. A second element is the artifact name, for instance `app-buildr`. By namespacing on the domain name, multiple projects can accidentally use a name (including poorly chosen names such as `util`). The last piece of the identifier is a version, which provides uniqueness of an artifact over time. 

With these three elements, it is always possible to uniquely locate an artifact from a repository. The use of the group in combination with the project name is not how current JS repositories work today, but there is signficant repository infrastructure in place that does do this and in the long term, most users are grateful to avoid the drama around projects that have taken a name and never do anything constructive with it.
  
So now we have a project that is uniquely identified. Most projects have dependencies on other projects, those projects have dependencies, and so on. These sub dependencies are called "transitive dependencies". NPM solves the transitive problem by downloading every transitive underneath it's direct dependency. Common dependencies such as `Require.js` could be downloaded hundreds of times. The problem with this situation for deployment is a browser cache only differentiates on URL, not content. Site load times are dramatically slowed in this manner. Bower was introduced to "flatten" this hierarchy, but does nothing to resolve version conflicts. A version conflict in a flattened hierarchy occurs when two transitives request and bind to different versions of a dependency. This can introduce very difficult to find problems, especially when working without the benefit of a language like TypeScript that can check type bindings at compile time. *Still, if two transitives depend on different versions, these will not be caught at compile time unless the build tool warns that multiple versions are in use.*

## Declarative versus imperative builds

Builds typically do the same thing, even if people don't recognize this. One can envision an absolute monster of a build and then compare it to smaller builds. More times than not, the larger build can be used for the smaller build by simply removing the parts that don't exist in the smaller build. 

If we take this at face value for a moment, let's also consider what it takes to understand a typical large build. Every time we approach a build as a developer, we have to digest a new strategy for what typically is exactly the same process. So let's try a couple of theories on for size: 

* If we always used the same strategy that we found in our monster build for every build on earth (but with the unnecessary bits removed), we could presumably approach a new build and understand it more for what is missing than understanding the build by reading every line of code. 
* If every build is the same, do we need to actually write code for it at all? Instead, couldn't we just list the steps that we want and leave it at that? 

## IDE support

Taken to the extreme, a big advantage of a declarative build is it can be parsed by an IDE. In the typical Gulp or Grunt build, all an IDE can do is parse for `task` calls and make them available in the IDE. This is because a lot of code can be put in either of these builds and there's no reasonable way for an IDE to parse all of this. When the build is declarative, it can be parsed and much more granularity can be extracted for use by the IDE (and developer). 

## Build plugins

And since we can assume that we don't know every tool that will ever exist, we need a means to plug in new tools that emerge over time. Note that these task plugins will contain imperative logic, but should present themselves declaratively.

Something subtle about plugins is everything beyond the load of the `buildfile.json` can be a plugin. Plugins are downloaded automatically in the same manner as other dependencies.

## Convention over configuration

In any descriptive environment, we'd like to reduce the amount of verbosity. Take the example of where source files are stored. If developers always put their source files in a folder called `src` by convention, we don't have to make an explicit configuration to set the same value every build. Though we may have different types of source in a single build, so we may need to have `src/js` and `src/scss` for instance. In doing so, both tools can be left unconfigured.
 
This seems like a minor detail, but it's not. When authors use the conventions, developers who follow know exactly where to find source files. It really speeds up adoption of a new build.

This doesn't work all the time. Imagine converting a Gulp build to app-buildr for a project under active development. Existing developers can't lose time while the build is converted, we need to run the old and new builds in parallel. Developers may be used to where the files are at and it would slow people down to change file locations, regardless of how bad things are. In that case, the convention cannot be used and we nave to use configuration for every directory instead.

## Example

With that in hand, let's get down to an example build:

```
.
├── buildfile.json
└── components
    ├── one
    │   ├── buildfile.json
    │   └── src
    ├── src
    ├── two
    │   ├── a
    │   │   ├── buildfile.json
    │   │   └── src
    │   ├── b
    │   │   ├── buildfile.json
    │   │   └── src
    │   ├── buildfile.json
    │   └── src
    └── src
```

Note:
* There are five projects here. A project root always has a `buildfile.json`.
* The convention is that a `src` folder contains anything checked in to SCM.
* Not shown is a `target` or `build` folder. Generated source and final artifacts are created to this folder, nothing from this folder is ever checked in. It follows that the `clean` task is the same as deleting this folder.

Ideally, the buildfile.json can be validated by [JSON Schema](http://json-schema.org). In doing so, we can "fail fast" when a build is not correct.
 
## Build dependencies revisited

As discussed above, dependencies between projects are inevitable. But what if a dependency has not been built yet and does not yet exist in the repository? This often happens in large builds. 
 
What we need to do is have a means of ordering the builds. Using the previous example, we might find the component at `/components/one/b` depends on `/components/one`. If the build tool approached the directories in an alphabetical manner, this build would work without problem, but would not if the directories were in a different lexicographical order.

There are two solutions for this:

* Build the dependencies manually. Many builds require this today as there's no other way to determine dependencies.
* Have the build tool read all of the `buildfile.json` files in a build to start with and scan for build dependencies using a graph. Using a simple graph sort, the order required for the build to complete without error is provided. (Note that we cannot sort a graph that is cyclic in any part, in practice this is not common or difficult to overcome.)

After considering this task of discovering all sub-builds and and ordering their execution by dependency, it's one great example of how a declarative (rather than imperative) build really helps keep things clean.

Note that there is a finer granularity than a build that is possible, covered in the next section.

# Task triggering and granularity

One of the difficult junctures for app-buildr is how and when to trigger tasks. Now that we've gotten a better understanding of of how to identify taks and recognizing that some tasks can't be accomplished until other tasks or builds are complete. So the question arises: In a multi-module build, do we really need to wait until a build is completely finished before starting on another? 

The answer is we do not, but we need to know exactly what we need from another build and trigger accordingly. In the previous question about `/components/one/b` depending on `/components/one`, what we care about is the final packaged artifact of `/components/one` being ready. Deeper introspection reveals that a build may only actually depend on the `.tsd` for a build being ready though. Some consideration should be given to such triggering since much higher parallelism can be offered if we know that we can fire all builds in parallel but block on completion of specific tasks in other builds rather than waiting for the entire build to be complete.

