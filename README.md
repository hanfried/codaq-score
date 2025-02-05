# codaq-score

Code A Quality Score to predict security bugs or maintainance burden in the future

## Idea

Scan a code base and use ML/AI techniques to predict the potential nr of CVEs or maintenance afford needed in the future,
by training it to find and evaluate code smells and compare code histories for historic outcomes regarding CVEs and maintenance.

This tool won't have the exact knowledge where a security bug will be placed in the source nor how to fix.
Both are incredibly hard problems, very likely not decidable in general by computer program.
It won't replace linters, fuzz testing, pen testing, security audits, bug bounties, or security experts.
But it might help to automatically judge the amount of technical debt and future costs for a project.

The outcome will be like a credit score, kununu score, amazon review score and similar.
Nothing you can be totally determined on, something that might be wrong, but anyway,
the world needs a way to judge independently and automatic the quality of software,
beside looking at github stars (what's probably the current "technique" for it) and latest commit,
especially for something tangible like security.

It's intended to be a helpful guide, similar like the other scores, to get an idea how risky and costful a codebase 
might be in the future.
It's not intended to provide direct tooling how to fix code, how to prevent bugs or security issues.
But it can lead to where to review code more intensively, where to use (cost or time consuming) tools and what budget
might be needed in the future.

## Available data

Let's try to focus on hard facts that are relatively easy to get, hard to bias, and automatically processable.

### CVE reports

Contain:
* Date of CVE
* a base score
* a severity
* an exploitable score
* an impact score 
* of how severe it is (high / critical)
* and optionally a category (like overflow) 

See, e.g. [CVE details](https://www.cvedetails.com/)

### Github (and similar) repositories

Contain:
* History of code (hard fact)
* Release dates with corresponding code (soft fact, depending on the maintainer)
* Changes over time (hard fact)
* Pull requests (soft fact, depending on project and maintainer)
* Issues with some categories (very individual) (and maybe a sentiment analysis :-o)
* Security informations (sometimes)

### News articles / Tweets / ...

Contain bugs in combination with public attention.
Probably hard to parse, to judge, to unbias, and relatively seldom

### Additional tooling

* Outcome of linters and specialized security tools
* Code coverage of tests or similar
* Code obfuscator and minimizers
* Documentation
* Content of links in github
* ...

## Training idea

### Input 

Use random code/artefact snippets from a github repo from (main/master) branch for different dates/releases in the past.

Instead of just taking the whole code base,
it's probably better to take just some random parts of it as input, because:

* Training on a whole code base will overfit
  (the ml model will most likely just detect your project and remembers the security score for it) 
  and will be quite often out of a context window for w/e model is choosen for larger models.
* Ordering of input is hard to make reliable for training, so best is to give it up completely
* Allows to train the same model for small projects as for large projects
* Automatic augmentation from the start
* Allows to judge incomplete projects (e.g. your current code base in your IDE)
* Allows to easily add additional artefacts from linting, documentation, following links
* Allows to obfuscate or minimize code samples
  (so model does not just learn some specific variable names or some code style to re-identify the project)
* Avoids having to run the code or tooling for it depending on programming language or specific versions
  as:
  * that's tough to setup for random repos
  * even for programming languages well understood, 
  * very costly, security problematic
* Avoids overfitting on dependencies
  * if they won't show up in most train samples, the ml model has to learn something about codequality without just
    learning how bad some dependencies are (or learn when similar patterns appear)
* Allows parallel processing and train on cheap computers
* Allows to "identify" somehow problematic code parts in test time:
  * they should be in the intersection of code snippets when running different samples and looking to those with worst scores

To experiment:

* Samples of same length or different random lengths
  * gut feeling: random lengths give more augmentation and leak less the project
  * what is a good average length for the code snippets (it probably depends on the size of model)
* Semantic complete (like complete functions, classes, files) or just randomly cut
  * gut feeling: randomly cutting is more augmentation and leak less the project and 
  * does not need to understand the code from our point of view (makes data preparation tooling much easier)
* Use code only or code and documentation and other artefacts (like some jsons or open issues for what ever)
* Include linting output as artefacts to sample from or not 
  * in case, does it need a reference for the same line as code or not
* Is oversampling of code parts that are not covered by unit tests helpful or not
* ...

### Output

#### Targets

* Expected Nr of CVEs existing for the historic project code base when looking into the future (from the historic release date),
  not completely sure how much to look into to future (always up to now = training time, or always 1 year in the future, ...)?
* Expected total sum of CVE score
  * that probably should be our target score for the system, all other outputs are just multitask learning targets
  * also predict total sum of exploitable, impact scores etc
* Expected num of critical / high scores
* Similar by category (overflow / ...)
* Predict which lines will be changed in the future (and at best how often)
  * that's pure multitask learning: it doesn't really say something about security/quality as it does not differentiate
    * feature requests
    * code refactorings
    * program logic bugs
    * security bugs
    * but still: it helps a model to understand code
    * and: could be used as additional learned information what kind of code is very likely to be changed again in the future
    * especially lines that will be very likely changed multiple lines could indicate troublesome code that needs to be reviewed more closely
* Optionally: predict which lines will be mentioned in issues or pull requests
* Optionally: predict linting output (pure multitask learning)

#### Weighing

When only looking to (randomized) samples of a project, of course the total nr of predicted CVEs or scores
will depend on how much a fraction of the total code base we are looking to,
if the sample includes 30% of the code base, we probably won't expect to see 100% of CVEs predicted
but something in relation to the input size.

It's something to learn to (gut feeling: the relation is not linear, maybe more sqrt or logarithmic),
so we probably need a `f_weighting` to learn something like:

    target_specific_score_for_training = f_weighting(fraction_code_covered, fraction_documentation_covered, ...) * target_total


#### Inference

Take multiple samples of the code base,
and predict the target scores.
Then summarize the inferences into one score (probably in the beginning just average score multiplicated by 1 / f_weighting.

Advantages:

* Automated augmentation
* Can construct a learnable method to summarize results beside just averaging scoring
* Can give an idea of uncertainty and how code quality is overall in the project by looking at 
  * histogram of min/max distribution of the score
  * see where worst scores intersect
  * so you can see whether whole project is high/low risk
  * or just some parts of it
* Allows parallelization
* Hopefully runs on cheap hardware
* Harder to play the score

Disadvantages:

* Higher inference costs
* Inference results might be non deterministic 

## Roadmap

### Collect and understand training data

* Get maybe 5k-10k github repos (maybe 5k top github repos, plus 5k github repos most mentioned in CVEs)
* Get CVE data
* Prepare training data

### POC with a cheap model

### Look whether results make sense and how far predictions are

When we look to historic data (we can't know yet CVE data for the future),
it's always possible that the model just cheats, so needs double checking

### Experiment

See above for possible experiments


## Discussion / Problems

As said, none of these techniques will pinpoint inside the code what is exactly a problem.
It won't replace any linters, expert reviews, etc.

But it's doing pretty much the same as a human expert (=experienced developer) would do with an unknown code base:
* Browsing around in the code base and look whether you see code smell or good programming practices
* Probably check some implementations in detail whether they make sense (=what linters will do somehow)
* Look around in issues and links in the code base

The architecture should allow to add additional tooling on top of it, so none of the shortcomings would be a showstopper.