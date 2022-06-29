---
layout: post
title:  "Step by step setup of pitest on pull request"
author: henry
categories: [pitest, java]
image: assets/images/steps.png
description: "How to setup pitest on pull request"
featured: false
hidden: false
---

## Step 1. Add Pitest

This is usually straight forward, but sometimes there are issues in the test suite that prevent pitest from running. Before we run pitest on PR we need to make sure it runs locally.

First, configure the pitest plugin for the build. If you're using junit4 this is enough:

```xml
        <plugin>
          <groupId>org.pitest</groupId>
          <artifactId>pitest-maven</artifactId>
          <version>1.6.6</version>
          <configuration>
            <failWhenNoMutations>false</failWhenNoMutations>
            <timestampedReports>false</timestampedReports>
          </configuration>
        </plugin>
```

If you're using junit 5 it will need to look like this.

```xml
        <plugin>
          <groupId>org.pitest</groupId>
          <artifactId>pitest-maven</artifactId>
          <version>1.6.6</version>
          <dependencies>
            <dependency>
              <groupId>org.pitest</groupId>
              <artifactId>pitest-junit5-plugin</artifactId>
              <version>0.14</version>
            </dependency>
          </dependencies>
          <configuration>
            <failWhenNoMutations>false</failWhenNoMutations>
            <timestampedReports>false</timestampedReports>
          </configuration>
        </plugin>
```

To make it easy to run also add a profile

```xml
        <profile>
            <id>pitest</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.pitest</groupId>
                        <artifactId>pitest-maven</artifactId>
                        <executions>
                            <execution>
                                <id>pitest</id>
                                <phase>test-compile</phase>
                                <goals>
                                    <goal>mutationCoverage</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
```

Pitest can now be run with

```bash
mvn -Ppitest test-compile
```

Depending on your project, this may take a few seconds, a few minutes, or run for so long that you give up.

If it takes a few minutes you might want to configure it to use more threads. If you're in the last category, don't worry. We don't need to mutate the whole codebase unless we really want to. At the moment all we care about is knowing that pitest will run without an issue no matter which bit of code someone changes in a PR. We can find this out by running

```bash
mvn -Ppitest -Dfeatures="+CLASSLIMIT" test
```

This limits pitest to only creating 1 mutant for each class. 

Unfortunately each of these mutants will be expensive (there's a fixed per class cost to mutation testing), but things will still run much faster. An hour long analysis for one of my example project drops down to a more palatable 8 minutes.

# Step 2. Fix Any Problems

If everything ran green that's great, skip to step 3, but you might have got an error like this

```
Execution of goal org.pitest:pitest-maven:1.6.6:mutationCoverage failed: 
32 tests did not pass without mutation when calculating line coverage. 
Mutation testing requires a green suite
```

Followed by a list of failing tests.

The most common reasons for this error are :-

* Configuration that is set for surefire, but not for pitest
* A test order dependency
* A problem with the tech stack

The first one is simple enough. If you have tests that are normally excluded from running, or that rely on system properties that are set in the surefire config, that config will need to be replicated for pitest (pitest tries convert the surefire config, but won't always get it right)

A test order dependency is more complex. This is when the tests fail if they are run in a different order than usual. Most people are adamant that they don't have test order dependencies, until they find them.

Pitest runs tests differently than surefire. It splits them into the smallest units of runnable code that it can so that the minimal amount of tests are run against each mutant. Because of this, the order of tests is likely different than with surefire. 

The thing to look out for is state. This could be a static variable, a database, or a file system. If your tests modify any of these and do not restore the original state afterwards they are a likely source of a test order dependency.

The last category is more tricky. The process of breaking tests down into small units is messy, and sometimes a custom runner or similar does something that pitest can't handle. If you find one of these, please report it as a bug. In the meantime you can exclude the test from being run by pitest.

# Step 3. Setup Git Integration

Once everything is running green on its own, we can start to integrate with Git. To do this we need two things

* A pitest plugin
* A licence file

The plugin is added to the pitest configuration (not the project dependencies) so it will look something like

```xml
      <plugin>
        <groupId>org.pitest</groupId>
        <artifactId>pitest-maven</artifactId>
        <version>1.6.6</version>
        <dependencies>
          <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>0.14</version>
          </dependency>
          <dependency>
            <groupId>com.groupcdg</groupId>
            <artifactId>pitest-git-plugin</artifactId>
            <version>${cdg.pitest.version}</version>
          </dependency>
        </dependencies>
        <configuration>
          <timestampedReports>false</timestampedReports>
          <failWhenNoMutations>false</failWhenNoMutations>
        </configuration>
      </plugin>
````

The version number has been placed in a property. We only use it here just now, but in step 4 we'll add some more dependencies to the pom that need the version to match.

To get a licence [send a mail](mailto:support@arcmutate.com) with the details of your project. They're free for OSS projects of any size, and we're giving away a limited number of licences to commercial teams. Don't be afraid to ask.

Once you have a licence place it in the root of your repo (it doesn't need to be kept secret) with the name `cdg-pitest-licence.txt`

To see if it's worked run

```bash
mvn -Ppitest -Dfeatures=+GIT test-compile
````

If the plugin has been successfully installed it should run for a bit, then most likely stop with a message saying that no mutations were found. This is because the default behaviour is to mutate only code you've modified locally, at the moment there aren't any. To see it in action, modify a line of production code so the tests still pass (perhaps add a `System.out.println` somewhere). Run the above command again, and you should see that pitest creates mutations only on the modified line of code.

# Step 4 - Setup Pitest on PR for Trusted Changes 

We now have pitest fully working with git integration. The next step is to get it running against PRs that come from branches within the main repo, so the person creating them is a trusted individual. If you're configuring an OSS project that accepts PRs from untrusted forks you could skip Step 4 and jump straight to step 5, but we recommend you don't. Setting that up is slightly more complex, so its best to flush out any issues while setting up this simpler workflow.

Add a new github action workflow using [this example](https://github.com/GroupCDG-Labs/pitest-github-demo/blob/main/.github/workflows/pitest-pr.yml) as a template. You'll also have to add a new plugin to your pom.

```xml
 <plugin>
   <groupId>com.groupcdg</groupId>
   <artifactId>pitest-github-maven-plugin</artifactId>
   <version>${cdg.pitest.version}</version>
 </plugin>
```

This plugin reads the output generated by pitest, and uses it to update the current PR. It requires a token with write access, which Github provides automatically for workflows originating from a branch.

Everything is brought together by this line in the action yml.

```bash
run: mvn -e -B -Ppitest -Dfeatures="+GIT(from[HEAD~1]), +gitci" test
```

This runs pitest, producing additional output which is then used by the maven plugin. Although at first glance it looks like only the most recent change is analysed, in fact all changes in the PR will be considered as github places them in a single commit. 

Create a branch for this change and open a PR. The workflow should run and the plugin should update the PR with a message saying no mutations were found. To see it fully working modify a line of production code (again, perhaps add a System.out.println call) and when the change is pushed, pitest should tell you that there is a surviving mutation.

# Step 5. Setup Pitest for untrusted forks

To safely analyse code from untrusted forks, we need a more complex two step process.

Add two new workflows based on [receive-pr](https://github.com/GroupCDG-Labs/pitest-github-demo/blob/main/.github/workflows/receive-pr.yml) and [update-pr](https://github.com/GroupCDG-Labs/pitest-github-demo/blob/main/.github/workflows/update-pr.yml) in the demo repo.

All the heavy lifting of mutation testing is done by `receive-pr` it is run without access to a write token, so it cannot update the PR. Instead it stores the mutation testing results as an artifact, along with some meta data about the PR. This is done by another plugin that must be added to the pom.

```xml
<plugin>
  <groupId>com.groupcdg</groupId>
  <artifactId>pitest-git-maven-plugin</artifactId>
  <version>${cdg.pitest.version}</version>
</plugin>
``` 

The `update-pr` workflow it kicked off when `receive-pr` completes (be careful that the names still match if you rename anything), and updates the PR with the results. It is important to understand that the version of this workflow that runs is always the one one the main branch. So it won't work at all until its been merged in.

Merge the new workflows in, then create a fork of the repo and create a PR from it. Both of the new workflows should run and update the PR as before.

Once that's all working, you may choose to delete the original `pitest-pr` workflow. If you remove the line

```bash
 if: github.event.pull_request.head.repo.full_name != github.repository
```

from `receive-pr` then it will run on pull requests from both fork and trusted PRs.

# Step 6. There Is No Step Six
