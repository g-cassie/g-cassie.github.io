---
title: "Putting Pants On: One Thing We Did Right After 5 Years with Django"
date: 2021-09-13T21:22:55-07:00
draft: true
tags:
- python
- django
- monorepo
- pants
---

## Putting Pants On: One Thing We Did Right After 5 Years with Django

** **

In 2019, we had been actively working on our Django application for over 5 years and it was starting to show.  The tales I had heard of successful software companies becoming bogged down in technical debt to the point of crisis were beginning to sound far more plausible to me.  As a result, we began a quest to figure out how we could evolve our development practices to enable a sustained high rate of output far into the future.  This has taken us on many adventures in different areas of software development with widely varying degrees of success. But what I’m here to write about today is one discrete change which produced a number of significant wins for us: moving our code to a monorepo using the [Pants build framework](https://www.pantsbuild.org/).

There are countless blogs debating the ups and downs of monorepos so I will try to avoid rehashing those discussions.  Having said that, by moving to Pants, we consolidated five repos into one. As a result, some of the motivations I’ll mention below are rooted in the benefits of monorepos generally.

So, what was running through our heads when we decided to devote a month of development time to migrate our codebase to the Pants build tool?  Well, in no particular order, here were our motivations.


## Scaling Tests

As our codebase grew, I was becoming more and more concerned about the trend of our test suite. For one, a full test suite run had hit 20 minutes and continued to grow.  We’d offset this a bit by introducing parallelization in our CI, but it was obvious this was only a temporary bandage as our test suite continued to balloon.

What was more frustrating though was the test start time. To run a single test there were often over five second delays for the test runner to bootstrap, even when the test was a simple hello world. The culprit was a complex web of dependencies that had emerged which meant that often the entirety of our codebase was loaded to execute a single simple test.  Around this time, I stumbled upon [this post from Instagram](https://instagram-engineering.com/python-at-scale-strict-modules-c0bb9245c834) describing how developers routinely wait 20-60 seconds to run a single unittest.  The author notes, “most of that time is spent literally just importing modules, creating function and class objects.”

Pants promised to solve this problem in a variety of ways. First, Pants treats individual test files as targets and traces their dependency trees. When you ask to run the entire test suite, Pants can determine which test targets have the potential for a new result based on the files changed since the last run and return cache hits for all others. This drastically abates the problems from an ever-growing test suite by ensuring that tests that don’t need to be run, don’t get run.  Additionally, each individual test target can run as a standalone process, allowing pants to parallelize your test runs out of the box.

Second, Pants makes it possible to properly isolate dependencies. When a pants test target is run, it will only include the python files that are part of the dependency chain, leaving everything else out. This means discrete modules with limited dependencies have the fast test times you’d expect, even if they are living in a large repository. 

Perhaps more importantly though, Pants equips developers with the tools they need to make good decisions about dependencies.  For example, in a typical Django project, the natural order is for all code to live together in Django apps even if it doesn’t actually have anything to do with the framework.  In Pants, it is trivial to create new standalone modules with no external dependencies and then import them across your codebase as needed.  This empowers developers to keep modules as pure as possible.   Pants even provides CLI commands which allow you to introspect dependency chains in your app and identify areas where your project is beginning to devolve into a ball of yarn.


## Safe Builds

Prior to moving to Pants, our deploy process involved zipping our code base, pushing it to S3 and then triggering a deploy through AWS CodeDeploy.  On each server, the CodeDeploy agent would pull down the deploy artifact, unzip it and then run pip install to ensure it the latest dependencies.  Can you spot the problem?

Running pip install on each server created a number of risks. Most obviously, if pypi was down we couldn’t deploy.  What was more worrying though was the possibility that the contents of pypi would change in a way that was not caught by our dependency pinning. We could deploy a release successfully and then when a new server spun up a few days later, that exact same release could fail.  This did in fact happen when one Monday morning new servers spun up in an autoscale event and picked up a new version of Kombu that was not supported by the version of Celery we were using.  Because Kombu was not a direct dependency of our application we did not have it pinned.

The solution was to package the requirements from pypi in the deploy artifact.  Of course, this can be done manually, but with Pants you get it for free.  Out of the box, pants builds `pex` files. These are executable files that contain everything needed to run the python application.  The host machine only needs to have the correct version of python installed.  Today, all of our applications are built by the Pants CLI as .pex files and pushed to S3. We have complete confidence that at any time we can rollback to any release and have a perfectly functioning deploy artifact.

 


## Zero Effort Code Sharing

Early in our life as a startup, we built a standalone authentication service.  Because we were moving quickly and not thinking too much, we simply started a new repo and began another Django project.  As a result, we now had two Django applications which lived in separate repositories.  We hadn’t set up any method of sharing code between these so in some cases we were literally copying and pasting code between the repos.  Once both these projects were moved into a monorepo it was trivial to begin moving duplicate code into standalone modules that were imported by both Django applications.


## Simplify SOC 2 Compliance

In my experience with a SOC2 Audit, the less things you have to audit, the better.  In the case of code repos, each additional repo you have means more repo level settings to keep track of and ensure meet the requirements of your internal controls. A single repo with the wrong settings could impact your entire SOC2 assessment.  After going through the process myself, I was very eager to reduce our repo count and using a monorepo made that possible.  With substantially all of our development now taking place in a single repo, it is trivial to check that settings for approvals, code checks and other compliance items remain properly configured.


## Enforce Best Practices Across the Codebase

Pants makes it trivial to enforce code style requirements across all of your organization’s code.  We were easily able to add flake8 and black style formatting with a few lines of config.  We don’t need to worry about keeping these configs consistent across multiple repos. Pants also makes it possible to enforce your target python versions. This avoids surprises when a developer is using python 3.9 and in production you are running python 3.8.


## What about the alternatives?

Before moving ahead with Pants we did do our research. In particular, we looked into Buck and Bazel – massive scale build tools managed by Google and Facebook respectively.  The problem with these tools is that they were very much developed by these companies for their own specific needs.  As a result, unless your development practices are similar to Google or Facebook’s (and if you’re not them then they probably aren’t…), you will struggle to align your project with assumptions of these tools. To give one example, consuming third-party dependency artifacts from public repositories like PyPI is not a major use-case at a company like Google, but extremely common everywhere else.  Therefore Pants has robust, first-class support for such external dependencies, and it's even possible to manage multiple versions of the same dependency.

What was perhaps more acute for us though is that these tools are very unapproachable. As we explored the inscrutable documentation, we found no clear way to dip our toes in and try building a simple python app.  Compare this to Pants where a single page of the docs explained how to get a python codebase building self-contained executable files with a few simple lines of config. Pants’ first-class support for Python means it’s trivial to get up and running with common python tooling like pip, setuptools, pytest, black, isort, flake8, and more. 


## How did we migrate?

** **Migrating to an active code repository without fully pausing development requires a little bit of thought.  In our case we had five repos to move – four of which were fairly small projects and one of which represented the bulk of our application.  We stood up the monorepo with Pants and focused on moving the four small services one by one, in order of least complex to most complex.  In each case we brought the service over to the monorepo, set up a full deploy pipeline and began deploying it to production from the Pants repository before we moved onto the next.  We kept the old repos and the new repos live so we could move back and forth between Pants builds and legacy builds at any time.  Where we identified risk in the transition, we would put a Pants build into production alongside a legacy build and actively monitor how they performed before phasing out the legacy build.

For our largest repo, we needed to be able to continue to develop and release features off the legacy repo while we moved to Pants. Unfortunately, porting this repo over to Pants revealed an immense tangle of dependencies of files importing each other in endless cycles.  Before we could move our code over, we would need to sort this mess out.

We began by adding Pants BUILD files into the codebase and committing them to the legacy repo. We then wrote a bash script consisting mostly of sed commands which would port the legacy repo over to Pants in its current state. This allowed us to try running tests in the Pants environment, identify issues with dependency cycles, and then committing fixes to the legacy repo. We could then rinse and repeat, working iteratively until running the bash script on the legacy repository produced a perfectly functioning Pants build.  Of course, we didn’t have to work completely by trial and error; using Pants CLI commands we could print off lists of dependencies and create a list of all the cycles that needed to be broken. We then went down the list and untangled the mess, dependency by dependency.

While I am very grateful for our team having gone through the exercise of untangling this ball of dependencies, since we migrated, Pants has introduced dependency inference and tolerance for dependency cycles. This means if we were to migrate today, we could move to Pants immediately and then use it’s tooling to ameliorate our dependency mess at our own pace - avoiding the split legacy and Pants environments like I described above.


## Conclusion

At this point, we are approaching two years of working with Pants and we couldn’t be happier.  Our test suite has continued to grow but, with Pants caching, our CI test times have remained constant (or decreased in some cases).  Using pex files as deploy artifacts has drastically simplified our deploy process and eliminated a huge amount of risk with how we were doing things before.  I think most importantly however, our development team is thinking far more proactively about how to manage dependencies and making smarter decisions about separating code into sensible modules than ever before. 

And it wouldn’t be a technical blog post if I didn’t mention we are hiring at iManage Closing Folders. If you want to help us figure out how to build the highest performing software development team [check out the job postings](https://imanage.com/about/careers/) for our Toronto office (remote friendly!).
