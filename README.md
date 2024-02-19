# Fuzz testing: Beginner’s guide


## Introduction to Fuzzing

When considering the broad landscape of software testing, it's useful to categorize the various approaches into three primary types: *conformance testing*, *functional testing*, and *non-functional testing*. 

This mindset provides an understanding of how different testing strategies work together to ensure software quality and reliability, and how especially _fuzzing_ fits in this context. 
***Fuzzing*** is a specific testing technique often associated with both security testing (a subset of non-functional testing) and negative testing (a specialized form of functional testing).

*Negative testing* attempts to test that the software application is functioning according to the requirements and can handle unwanted input and user behavior.
This has been typically done manually by adding test cases that the developer thinks will be invalid or cause an error for the software.
However, the developer cannot always know beforehand all possible scenarios, and this is where fuzzing comes to the rescue. 

*Fuzzing*, in short, is about inserting malformed, unexpected, or even random, inputs into a program in the hopes of triggering new or unforeseen code paths, and bugs. Because fuzzing involves feeding the target with a large number of test cases, it is often at least partially automated, in contrast to negative testing. Fuzzing can, and should, be used to test every interface that accepts some form of input. In practice, fuzzing should be used to test at least every interface that takes in input from potentially malicious sources, i.e. Internet or user-provided files.

Fuzzing is a technique that complements other testing techniques. Issues revealed by fuzzing are often triggered by inputs that would have been unlikely to be constructed by a developer. For example, issues in the handling of corner cases, input sanitation and validation and error-handling routines. In regular test automation, fuzzing increases code coverage and even with high code coverage tests, unexpected inputs from fuzzing often trigger execution flows that are not otherwise triggered. 

Needs for implementing fuzzing can come from many sources. It is part of your Systems Development Life Cycle (SDLC), someone has ordered you to do it, you need to assure someone that systematic work is done to improve the target, or you are just interested in squashing some bugs. How much to invest in fuzzing depends on your endgame and resources, this document tries to help you to get more out of your fuzzing effort.

## Before starting

At this point, your hands are already itching to get started. Organizations, and individuals, often rush to implement fuzzing based on a great idea from a blog post or a cool demo someone saw in a conference. It is not necessarily a bad thing, but too often we see a large amount of work invested into a fuzz test system that is only used until the author is assigned another task or some change in the target breaks compatibility. Even worse are fuzz test systems consuming HW resources for years without any other result than a filled checkbox in SDLC checklist. Like any other part of a living software project, the testing automation, fuzzing or not, requires planning, maintenance and commitment.

### What do you want to fuzz?

This is a simple question, but the answer is not always as obvious as you would think. If you already have a target in mind, great. If not, you are looking for interfaces that accept input. Interfaces can be external, like a network connection, they can be files, or they can be purely internal like a function calling convention in a utility library your code is using. Fuzzing is about creating inputs for your chosen interface and seeing how well the implementation behind that interface handles those inputs when they are pushed to the extreme. Threat modeling and sketching a data flow diagram are techniques that can help you discover potential interfaces your target has. 

There can be many layers of software behind each interface and defining which layer you want to fuzz is crucial because for the input to reach there, **it has to pass through all checks on previous layers**. 

For example, let’s consider an HTTP server that accepts a signed blob of data. In that blob, we have a JSON string that contains values that are used by our application. In this example, we have four potential layers to fuzz:

1. HTTP messages for the server
2. Signing check for our data blob
3. JSON string parsing
4. Our code, which handles the actual values

Let’s assume that our HTTP, signing and JSON libraries are robust and we don’t want to target them. To fuzz our own code, we would need to generate the values, pack the values into JSON string, sign the blob, create an HTTP message and send it to the target implementation. Unless we already have a test automation that we can reuse, building the test case injection alone would take a considerable amount of time. Fuzzing through the whole stack will also cause constant overhead and is more prone to be broken when some of the layers are changed. 

In fuzzing, the throughput of test cases is also important, and you should consider if the target has functionalities that could be disabled or bypassed to reduce the overhead and increase coverage achieved by the fuzzer. Typically most of the benefits can be achieved by implementing a small program that calls our target code directly with the fuzzed values. In our example above, creating a program that directly passes values to our handling code, would bypass the network message, a couple of hash calculations, the cryptographic check, JSON stringify and JSON parsing. In more optimized fuzzing environments, things like unnecessary logging, CRC checks, I/O to files and calls to remote resources are disabled in a “fuzz-friendly mode”. A “Fuzz-friendly mode” can be implemented with ifdefs, mock functions, or other build configurations that are enabled only for builds used for fuzzing. Of course, when implementing fuzzing optimizations that alter the behavior of the target, you must ensure that these modifications do not hide, or create, any bugs.

Still, in the beginning, do not worry too much about having an efficient fuzzing strategy that injects thousands of test cases per second into an optimized fuzzing environment. A very effective way to start fuzzing is to spew random (or bit-flipped) data to any interfaces you find. If this quickly finds issues, you have found your first target interface! 

### What do you want to find?

Many times when you fuzz, the target may crash and burn, which is hard to miss. However, to get the best out of your effort, you want to look for other error conditions as well. Your target has its functional requirements, that define what the program should do, and from there you can derive what it shouldn't do. In addition to that, all programs can have logic flaws that can lead to leaks or excessive CPU and memory consumption. Depending on underlying technologies, the target might also be prone to memory corruption, command injections, or other classes of issues that should be noted. 

Initially, all potential issue types and their impact should be written down. There are existing detection tools and techniques for many different issue types, but some are complex to apply or have high execution overhead. Impact assessment helps when reviewing if using a tool or a technique is worth an investment. For example, incorrect calculation for a color value in image compression can be of low impact and detection is hard to implement. Whereas bypassing authentication with fuzzed, and invalid, credentials is fatal and easy to detect if you just remember to look for it.

When researching different tools and techniques, also consider your other testing automation. For example, in many cases, you will find out that your unit tests have been triggering some bug again and again, but you haven’t noticed it because you weren’t looking for it.

### How are you going to fuzz?

Fuzzing is a technique that can be executed by an individual using a single machine. In an average-size implementation, fuzzing can be executed as a part of a Continuous Integration(CI) system, running bursts of fuzzing a couple of times per day for different projects. On a large scale, fuzzing can be executed by a cloud fuzz test automation using hundreds, or thousands, of machines in parallel. All those environments have different sets of requirements that the final system has to fulfill. Too often, great fuzzers are abandoned because originally no one thought about usability with another target, with another instrumentation, or on a larger scale.

Like in all testing, the larger the scale, the more important automation becomes. Fuzzing your own program with a single instance is pretty straightforward. You inject fuzzed inputs into the target program until a bug is triggered. Fix the bug. Rinse and repeat. When doing the same with a hundred, or a thousand, instances in parallel, you start to see why features like duplicate filtering are important. When running fuzzing in a large organization where fuzzing runs parallel in CI for different builds, you might start to miss automatic issue reporting, test case minimization and patch verification.

## Getting started

At this point, you should have a rough requirement specification for three main components required for fuzzing: test case generation, test case injection and instrumentation. Now you can finally get started with the real work.

The Internet is full of open-source and commercial fuzzing solutions. Some only implement test case generation, some combine test case generation and injection, and some have full stack including instrumentation and automation. In general, commercial products have better usability and often complete solutions are readily available for most common use cases. Especially for an organization that wants to start fuzzing multiple products fast, a commercial solution is a real option. For individuals wanting to squash bugs, for fun and profit, those solutions are often outside of the budget.

Whether you decide to go with an existing solution or implement your own, some issues always come up. 

### Fuzzer flexibility

Especially when your endgame is to use the same tooling with different targets, make sure that the solution you are going to use is flexible enough to cover all those cases. A lot of time can be wasted, if the whole system has to be refactored, or in the worst case another system has to be built when fuzzing is needed for another target. Different tools also reveal different issues and in the long run, there will always be new tools that reveal again new issues, so switching of components, especially instrumentation, is a valuable feature to have.     

### Handling results

If you are setting up a fuzzing system, but you are not one of the developers actually fixing the issues found, contact the people who are going to work with the results your system spews out. What kind of information do they want to have in a bug report from fuzz testing? Developers rarely appreciate if the first thing they see in the morning is an inbox full of new bug reports along these lines:

>Title: Crash in program X

>Description: Attachment makes program X crash.

>Attachment: fuzz-test-case-1337 (22MB)


By default, a bug report should contain at least all the information required to reproduce the issue. Things like, but not limited to: configuration, operating system used, version or build number of the target, CPU and memory information and, where applicable, compiler options and debug flags. 

The setup used in fuzzing should be easily reproducible for the developer, and you should have an explanation ready for every fuzzing optimization done to the target. For example, a developer might not want to fix an issue that only reproduces when CRC checks are disabled unless you can explain how an input could be crafted to reproduce also when the CRC checks are enabled.

Fuzz test automation can also include bucketing of similar issues, test case minimization, regression range finding, fix verification, and can even provide the testing environment as a container, or VM, image.

### Tracking progress

After running your fuzz testing for a longer time, you might end up in a situation where no new bugs are found, which can indicate one of two situations: (i) your fuzzing is doing a great job and the robustness of your target is increasing, or (ii) your fuzzing is stuck, grinding the same code paths again and again. As stated in the introduction, fuzzing requires maintenance and commitment to stay effective over long periods of time. There are some techniques that you can apply that help you ensure that your fuzzing stays effective and that changes in the target are also tested.

If you are using coverage-guided fuzzers you might be already covered. As long as your code coverage is rising during fuzzing, there is no reason to worry. If your code coverage isn’t rising anymore, you might have a problem that requires more in-depth analysis.

The pure count of code lines covered out of the lines existing doesn’t give you much information. 
For example, the target might have code lines that are not executed without specific configuration, or there might be otherwise unreachable code that makes complete coverage impossible. It is helpful to run test cases with a tool that visualizes which parts of the code are executed and which are not. Comparing those results to the results from previous fuzzing runs, or other test automation, and checking locations that previously have had bugs, will help you ensure that your fuzzing is not regressing and that all relevant code paths are still covered. 

If some new code paths are missing, the next step is to analyze how to make your test case generator generate cases that trigger those paths. Especially with model-based fuzzers you often find out that a fuzzer doesn’t have the required message or field implemented in the model. With fuzzers that mutate sample files, the reason for a lack of code coverage is often in the low coverage of initial sample files, or a strict validation of the input. In the latter case, consider “fuzz-friendly mode”.

Without the code coverage feedback, things get a little tricky. If you have implemented “fuzz-friendly mode”, then you have one simple solution: make bugs. Add some prints, asserts or aborts in locations which your fuzzing should reach and follow from the results that those locations are actually reached. You can also add similar “bugs” to locations that have previously had bugs in them. Just keep in mind that the way you implement those checks shouldn’t affect your fuzzer's ability to stress that area of the code and that those checks shouldn’t end up in production. If your automation is able to use old builds, you can also use a build that has a known bug and verify that your fuzzer finds it. Testing that old bugs are found is also a good technique for finding areas to improve in your system.

# Contributors

- Atte Kettunen(F-Secure)
- Pekka Pietikäinen(OUSPG)
- Marko Laakso(Synopsys,OUSPG)
- Ari Kauppi(Synopsys)
- Joonas Kuorilehto(F-Secure)
- Eero Kurimo(F-Secure)
- Mathias Nyman(F-Secure)
- Risto Kumpulainen(F-Secure)
- Alexey Kirichenko(F-Secure)
- Niklas Saari (OUSPG)

# Translations

- Vancir - zh - review'd by Xiaoli Liu from Oulu University (Translation outdated)

This document was started in the context of DIMECC Cyber Trust Program.
