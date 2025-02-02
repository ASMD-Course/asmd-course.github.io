# Lab01

In the following lab we will experiment the various frameworks and techniques for creating BDD/ATDD tests in a Java
application. 

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

I will proceed to implement the new features. I'll start by adding more scenarios to the Gherkin file and then implement
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





