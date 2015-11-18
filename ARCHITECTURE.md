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

## Artifact sharing and types

The previous example brings up a problem though: While the artifact such as the `.tsd` for a dependency may be ready, where do we find it? 

Experience shows it's a very dirty practice for a build to know specifics of another build. So for instance, `/components/one/b` should absolutely not be hardcoded of where to look in the `target` directory for `/components/one`. If it does so and and that location changes, one and possibly many builds will break.

What this leads to is the concept of an artifact type discriminator. In the case of the `.tsd` file(s), `/components/one` should publish the file(s) to the repository, but with a type of `tsd` so that they can be located independently of the build. 

Taken a step further, artifact types can be used in different ways, SASS files come to mind. Based on the facilities of the framework or the skill levels of the team maintaining a project, it may (for instance) be much simpler for an application to work from a single CSS file that is generated as the composite of all the components. So if our final application is defined as the top folder in the hierarchy, the top-level build could use the dependency manager to determine that the SASS tool needs to know about all four builds below it and can even know what order they should be considered. A that point, the build tool could provide all the SASS files from these builds to the SASS compiler for the generation of the final output artifact (which itself is added to the repository with a `css` type).

## Triggering revisited

So far, we've created the basis of a multi-module build that blocks on specific artifacts of dependencies rather than waiting for the dependency to be completely built. What is the best way to set these dependencies?
  
One means that should be considered is instrumenting the repository such that a build can block on the repository artifact being updated rather than the blocking on an event from another build. This could be important for continuous integration environments where a large build may be split up across many machines. If we were to somewhow connect these machines over a network socket, how do we discover which machines are working on what builds? It seems much easier that a central repository that all builds are reporting to removes this synchronization difficulty because the discovery is to a single repository.

# Repositories! (and ugh...)

There's a lot of talk here about a repository hierarchy where artifacts are identified by a DNS-based `group`, local `name`, `version` and `type` for the unique aspect of a build we are interested in. The natural question is whether we should be considering four elements as a requirement at all, building a repository is not a simple task and getting it adopted can be even harder.
   
In past experience, tools drive repository adoption and not the other way around. If a tool can be an order of magnitude more productive than another because of a specific repository format, not only will that tool be quickly adopted due to it's relative benefits, the repository will gather significant support as well. 

So the first question is what kind of tool app-buildr is going to be. If it is just another incremental reimagining of the tools that the Javascript world has used to date, then it might be better to use the existing `npm` infrastructure. It's widespread and well-understood. There may also be a way to introduce the desirable search granularity and triggering we want into `npm`, but since the package.json file is really oriented toward encapsulation of a single artifact, so this may present too much complexity in the multi-machine build scenario.
 
There are other repository formats as well, Debian, YUM, Maven and Ivy to name a few. Both Maven and Ivy would support the functionality we need, but I think all four require some level of XML to be generated as a part of the identifying information. On the upside, we would not have to manage the repository infrastructure at all.

The other important consideration is probably if we want the repository to trigger based on the change of a given artifact. This is very possible by wrapping the repository in a local API abstraction, but it adds code to app-buildr that must understand filesystem change semantics. I don't know enough about JS APIs but assume that this is possible. Again, we need to consider this in the context of a multi-machine build as well.
