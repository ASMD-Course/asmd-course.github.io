# Laboratory 01

Code available [here](https://github.com/ASMD-Course/asmd23-lab01-atdd)

## Objectives
 - be operative with Cucumber/JUnit/Java/IntelliJ
 - exercise writing requirements with Gherkin
 - exercise ATDD
 - pre-check ability of ChatGPT to deal with Gherkin


## Tasks

1. **CALCULATOR**: The repo has a calculator example (with Gherkin specification, step implementation, and production code). Play with it. Add examples
and operations, use all Gherkin features. Strictly use ATDD. Does everything work as expected?

2. **TOOLING**: Experiment with installing/using Cucumber with Scala and/or in VSCode. Is VSCode better at all here? Does Cucumber play well with
Scala 3?

3. **REENGINEER**: Take an existing implemented small app with GUI, e.g. an OOP exam. Write Gherkin specifications explaining what the system is
expected to do, and make acceptance tests pass. Does the system need a refactor of implementation? What does it tell us about how
an application has to be designed to be easily acceptance tested?
Search here: https://bitbucket.org/mviroli/oop2023-esami (2023, 2022, ...)

4. **REQUIRE**: Write Gherkin specifications that completely capture specification of requirements (functional, non-functional) of a real application.
Might use a previous project of yours, with requirements already written down, or use any existing/hypothetical (small) application.
What are good/bad aspects of Gherkin?

5. **ATDD-LLM**: LLMs/ChatGPT can arguably help in write/improve/complete/implement/reverse-engineer a Gherkin specification. Experiment with
this, based on the above tasks or in other cases. Is ChatGPT useful for all that?



### 1. Calculator
I'll start the exercise by analyzing The Gherkin specifications already defined in the calculator example. 

```gherkin
Feature:  Adding numbers with a Calculator
  In order to not learn math
  As someone who is bad at math
  I want to be able to add numbers using a Calculator

  Background: Start with a Calculator
    Given I have a Calculator

  Scenario Outline: Add two numbers
    When I add <arg0> and <arg1>
    Then the sum should be <res>
    Examples:
      | arg0 | arg1 | res |
      | 1    | 1    | 2   |
      | 1    | -1   | 0   |
      | -5   | -6   | -11 |
```

Following the ATDD cycle, I'll first check if tests are passing, then I proceed to refactor the code and ask the customer
acceptance.

![Calculator](atdd-cycle.png)

For the refactoring part I'm going to create an interface where the public methods will be defined, then I'll apply it 
to the implementation class. I'll proceed to add also some unit tests with JUnit.

#### Customer Acceptance
For the customer acceptance, I have simulated the customer dialog by asking question to an LLM, in this case ChatGPT.
I've provided as input the Gherkin specification and asked to assume the role of a customer without a technical background.

First we have reviewed all the scenario specified in the Gherkin file:

**Customer** : Thank you for showing me the Calculator software. Let’s start by checking one of the basic requirements. 
If I want to add two positive numbers, like 1 and 1, what result does the Calculator give me?

**Me**: Using the calculator (showing the scenario and the tests) it return 2 as result.

**Customer**: Perfect, the result is 2, just as I expected. It seems to work for adding two positive numbers.

_..etc.._

Then client have proceeded to asking me about the limitations of the software and how it can be improved:

**Customer**: I have a couple of follow-up questions. What happens if I try to add more than two numbers, like 1, 2, and 3? 
Can the Calculator handle that?

**Me**: For now the calculator can handle only operations with two operands at time, but this behaviour can be extended. 

**Customer**: I see, so the Calculator is currently limited to two numbers at a time. That’s fine for now since it meets 
my needs. But it would be great to have the ability to add more than two numbers in the future. Another question can I 
verify the numbers I’ve entered before performing the addition? For example, if I accidentally input the wrong number, 
can I review or reset the inputs?

**Me**: No, actually the calculator can't review numbers when these are inserted.

**Customer**: Oh, I understand. So once the numbers are entered, there’s no way to review or correct them. 
That’s okay for now since the operations are straightforward, but being able to review or reset the inputs would be 
a useful feature for future versions. Overall, the Calculator seems to fulfill the basic requirements I outlined. Is 
there anything else I should know about its current functionality or limitations?

**Me**: For now the calculator is limited only for the addition and doesn't cover any other operations.

**Customer**: That’s good to know. So, the Calculator is currently limited to addition only, with no support for other 
operations like subtraction, multiplication, or division. 
While addition is all I need for now, having those other operations might be useful in the future

#### New features
From the feedback received from the customer, it can be deduced that the customer needs to extend the functionality 
of the calculator by adding additional operations that it can perform (such as subtraction or multiplication) and modify 
existing ones (input control and handling more numbers per operation).

I will proceed to implement these starting by adding more scenarios to the Gherkin file and then implement
the new features.

```gherkin
Feature: Performing operations with a calculator
  In order to not learn math
  As someone who is bad at math
  I want to be able to perform operation on numbers using a Calculator

  Background: Start with a Calculator
    Given I have a Calculator

  Scenario Outline: Add two numbers
    When I add <arg0> and <arg1>
    Then the sum should be <res>
    Examples:
      | arg0 | arg1 | res |
      | 1    | 1    | 2   |
      | 1    | -1   | 0   |
      | -5   | -6   | -11 |

  Scenario Outline: Subtract two numbers
    When I add <arg0> and <arg1>
    Then the subtraction should be <res>
    Examples:
      | arg0 | arg1 | res |
      | 1    | -1   | 2   |
      | 1    |  1   | 0   |
      | -5   | -6   | 1   |
```

Before actually write code I'll complete the Step file adding the new steps for the subtraction operation and also, 
adopting a TDD approach, I'll write also the new unit test for the new operation. After that I've
added the new operation first on the Calculator Interface and then in the Implementation class.

I've checked if all the test passed and performed a small refactor to the code.
After this I've repeated the step of asking the customer acceptance (as done before).

I've repeated these step for also for the multiplication operation.

```gherkin
Feature: Performing operations with a calculator
  In order to not learn math
  As someone who is bad at math
  I want to be able to perform operation on numbers using a Calculator

  Background: Start with a Calculator
    Given I have a Calculator

  Scenario Outline: Add two numbers
    When I add <arg0> and <arg1>
    Then the sum should be <res>
    Examples:
      | arg0 | arg1 | res |
      | 1    | 1    | 2   |
      | 1    | -1   | 0   |
      | -5   | -6   | -11 |

  Scenario Outline: Subtract two numbers
    When I add <arg0> and <arg1>
    Then the subtraction should be <res>
    Examples:
      | arg0 | arg1 | res |
      | 1    | -1   | 2   |
      | 1    |  1   | 0   |
      | -5   | -6   | 1   |
      
  Scenario Outline: Multiply two numbers
    When I add <arg0> and <arg1>
    Then the subtraction should be <res>
    Examples:
      | arg0 | arg1 | res |
      | 1    | -1   | 2   |
      | 1    |  1   | 0   |
      | -5   | -6   | 1   |
```

### 2. Tooling

In this exercise will experiment with using Cucumber with Scala using VSCode.

For visual studio code I've installed the official [Cucumber extension](https://marketplace.visualstudio.com/items?itemName=CucumberOpen.cucumber-official)
Compared to the alternatives, this is the most maintained one providing also several feature like syntax highlighting,
autocompletion and debugging.

These are the steps that I've followed for setting up the extension:
- Disabled or uninstalled other Cucumber extensions
- In the `.vscode` folder I edited the `settings.json` file with the following content:

```json
{
  "cucumber.features": [
    "src/test/**/*.feature",
    "features/**/*.feature",
    "tests/**/*.feature",
    "*specs*/**/*.feature"
  ]
}
```
This ensures that the extension will look for the feature files in the specified directories.
To test its usage, I created a sample feature file and ran it using the extension, comparing it to the IntelliJ plugin.
```gherkin
Feature: I want to test visual studio code extension for cucumber

    Scenario: Giving away a pencil
        Given I have 3 pencils in my case
        When I give 1 pencil to Jacob
        Then I've now 2 pencils
    

    Scenario: I receive a pencil
        Given I have 3 pencils in my case
        When I receive 1 pencil from Jacob
        Then I've now 4 pencils
```

Functionally, the two are quite similar, both provide decent syntax highlighting and support user shortcuts for creating 
new steps. However, VSCode offers a slightly more powerful autocomplete feature, while IntelliJ has better integration 
with the IDE for running tests.

This integration is the main reason I prefer IntelliJ for working with Cucumber. 
However, for a quick setup and a more lightweight environment, VSCode is a solid alternative.

### 3. Reengineer

For this exercise I've taken the implemented version of GUI exercise from the fist session 
of the OOP exam Course of the 2023 academic year.

The first step was to analyze the requested features, looking for the main functionalities that the system should have.
In the Test file the first comment enlists the expected features of the exercise:
- the user clicks on any cell in the grid, and it numbers incrementally,
- user can continue to select multiple cells, as long as it does not select one adjacent to an already numbered cell,
- the first press on a cell *adjacent to a numbered cell* (horizontal/vertical/diagonal), then all numbered cells move up-right by one position,
- with each subsequent press of any cell, all numbered cells move further up-right by one position,
- as soon as a press would cause a numbered cell to leave the grid, at which point the application closes.

We can translate these features into Gherkin scenarios:

```gherkin
Scenario: Click on cells
    When I click on empty cells that haven't any neighbor
    Then the cells will be marked incrementally
    
Scenario: Click on a cell adjacent to another that is currently marked
    When I click on a casual cell
    And then I click on another adjacent to the one clicked previously
    Then the mark will move upper-right

 Scenario: When the cells starts moving by a previous click on a neighbor then they continue moving on any other click
    When A cell starts to move
    And I click on any other cell
    Then the first cell will continue to move

 Scenario: Complete the game when marked cells
    When I click on a cell near the border
    And I click on another adjacent the one near the border
    Then the game will end
```

Note that I've merged the first two features into a single scenario, as they are closely related.
The application use a MVC pattern, so the testing part will be focused on the Controller class, where the main logic is implemented.

The interface of the controller (Logic.java) exposes few methods for the interaction with the view. While this is a good 
practice, as it exposes the minimum necessary to interact with the application, it is inflexible to use within tests.

```java
public interface Logic {

    Optional<Integer> hit(Position position);

    Optional<Integer> getMark(Position position);

    boolean isOver();
}
```

For this reason the interface has been enriched with additional methods:
    
```java

List<Position> getCells();

int getSize();

```

The first one return all the cells that have been marked, while the second one return the size of the grid.

The second step was to implement a wrapper class that extends the Logic interface, adding utility methods for the tests
letting the original interface untouched.


```java
public interface TestableLogic extends Logic {
    /**
     * Creates a neighbor from a given position
     *
     * @param position The position to find or create a neighbor for
     */
    void createRandomNeighbor(Position position);

    /**
     * Hit random cell when I click on a position
     */
    void hitRandomCell();

}
```
### 5. ATDD-LLM
For the following exercise I will use ChatGPT to analyze and improve Gherkin specifications of an existing application:
[Scooby](https://github.com/PPS-22-Scooby/PPS-22-Scooby). The tests are structured in multiple .feature files, each one
describing a specific aspect of the application. First I'll ask ChatGPT to solve a reverse engineer task.
Input feature files are available [here](https://github.com/PPS-22-Scooby/PPS-22-Scooby/tree/main/src/test/resources/features),
I'll upload them directly on the chat asking with the following prompt:

> The following files are cucumber features for a library. Can you reverse engineer it? What kind of application is ?

After analyzing the feature files, ChatGPT was able to deduce correctly that the application is a library for scrapping
websites.
>  A small framework for crawling websites and extracting data. It wraps URL handling and HTTP, coordinates which pages 
> have been seen, filters page content with selector policies, and aggregates the extracted results.

Also it tried to infer a minimal API for it.

```python
url = Url.parse("https://example.com/foo")  # depth awareness, parent(), join()
resp = Http.get(url)                         # .status, .headers["content-type"], .text

crawler = Crawler(http=Http, validators=[TextOnly()])
doc = crawler.fetch(url)                     # raises/skips on invalid/offline/non-text

policy = SelectorPolicy(by="class", params=["item", "price"])
scraper = Scraper(policy)
items = scraper.apply(doc)                   # [] if no matches

coord = Coordinator()
if not coord.seen(url):
    coord.add([url, *discovered_links])      # validates, dedupes http/https
    result.update(items)                     # batch/merge semantics
```
Once the context is clear, ChatGPT was able both to provide new features and to improve the existing ones.
It created new feature files, I report as an example the one for the Error Handling feature:

```gherkin

Feature: Error handling, retries, and backoff
  To be resilient
  The crawler should retry transient errors and fail fast on permanent ones

  Background:
    Given a crawler with max retries 3
    And base backoff is 500 milliseconds with exponential factor 2
    And the request timeout is 3 seconds

  Scenario: Retries on 5xx responses with exponential backoff
    Given the server at https://flaky.example.com/page returns:
      | attempt | status |
      | 1       | 500    |
      | 2       | 502    |
      | 3       | 200    |
    When I fetch "https://flaky.example.com/page"
    Then the crawler attempts up to 3 times
    And the final status is 200
    And the delays were approximately 0.5s then 1.0s between attempts

  Scenario: Honors Retry-After on 429
    Given https://rate.example.com/data responds 429 with Retry-After 2
    When I fetch "https://rate.example.com/data"
    Then the crawler waits ~2 seconds and retries once
    And if the next response is 200 the fetch is considered successful

  Scenario: Does not retry on non-retriable 4xx (except 408/429)
    Given https://bad.example.com/missing responds 404
    When I fetch "https://bad.example.com/missing"
    Then the crawler does not retry
    And the fetch result is "permanent_failure"

  Scenario: Times out long-running requests
    Given https://slow.example.com/hang never responds
    When I fetch "https://slow.example.com/hang"
    Then the request times out after ~3 seconds
    And is retried up to 3 total attempts

  Scenario: Aborts chains of redirects beyond the limit
    Given max redirects is 5
    And https://loop.example.com/ redirects 10 times
    When I fetch "https://loop.example.com/"
    Then the fetch result is "too_many_redirects"
```
The original set of feature files already contains some scenarios for handling errors; this comprehends new cases and its
more specific.
I've also asked to improve the existing ones by adding more examples and improving the language, for example the following
code snippet represents new added scenarios for Results files:

```gherkin
Scenario: Append items and preserve order
  When I add items:
    | name   | url                   |
    | A      | https://example.com/a |
    | B      | https://example.com/b |
  Then the size is 2
  And item[0].name = "A" and item[1].name = "B"

Scenario: Merge (upsert) by key
  Given the unique key is "url"
  When I add an item { "name": "A2", "url": "https://example.com/a" }
  Then the item at url "https://example.com/a" has name "A2"

Scenario: Reject invalid item with errors
  When I add an item { "name": "", "url": "not a url" }
  Then it is rejected with validation errors on "name" and "url"

Scenario: Stable serialization
  Given I export to JSON twice without changes
  Then the serialized output is byte-for-byte identical

Scenario: Pagination of large result sets
  Given page size is 2
  When the result contains 5 items
  Then page 1 has 2 items, page 2 has 2 items, page 3 has 1 item

Scenario: Attach and query tags
  When I tag items where name starts with "A" as "alpha"
  Then those items include the tag "alpha"
```

Generally speaking ChatGPT was able both to understand the context of the application and to provide useful improvements
to the existing Gherkin specifications, as well as to create new ones that were missing. Human feedback is needed to
ensure that the improvements are valid and that the new features are useful.






