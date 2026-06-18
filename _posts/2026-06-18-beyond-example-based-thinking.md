---
layout: post
title: Beyond Example-Based Thinking
---

*How extracting an executable spec paradoxically let us 10x our example-driven tests*

I remember looking at the test suite my team had put together - around 500 integration tests covering our API surface area. Alongside the usual happy-path and negative cases, we even had a few that invoked the APIs concurrently: create two items at once and assert that only one call should succeed, or create and delete an item at the same time and assert that they should happen in _some_ logical order (the item should never be in half created/deleted state). Those concurrent ones were a mind bender, because what you observed depended on the logical order the operations ran in. A developer had to simulate the orderings in their head and write the assertions for each. They were good at catching concurrency bugs, so we wrote a handful of such tests. Overall, we were reasoanbly proud of our test suite.

Yet I still had a nagging feeling. Every time I looked at the suite I could spot gaps: negative cases we hadn't written, interaction patterns we hadn't covered. I could have continued to nag the team to write more tests but every test case is code that has to be written and maintained and there's a natural ceiling on how many you'll add before the cost stops feeling worth it.

Secondly, as I read the tests, I kept seeing the *semantic contract* of the API hiding between the lines. The tests were telling a story:

- You cannot put inventory without setting up the SKU first.
- You cannot take more items out of a warehouse than you put in.
- You cannot delete a SKU with active references.

The contract was **everywhere in the test suite, yet nowhere in particular**. Ideally we'd have written a design doc spelling it out, but we hadn't, so the test suite was the next best thing.

This naturally lead to the question: can we *tease apart* that contract from the tests, and state it directly?

## What the Contract Looks Like

I'll illustrate what the contract looked like with something simpler than the service we were actually testing - a simplified banking service.

The contract's job is to answer one question for each operation: given the current state and this request, what should happen? Here it is for withdrawals:

```csharp
spec.Operation<WithdrawRequest, WithdrawResponse>("Withdraw", (request, state) =>
{
    if (!state.Accounts.TryGetValue(request.AccountId, out var balance))
        return Expect.That(r => r.IsNotFound).SameState();

    if (balance < request.Amount)
        return Expect.That(r => r.IsBadRequest).SameState();

    var newBalance = balance - request.Amount;
    return Expect.That(r => r.IsSuccess && r.Balance == newBalance)
                 .ThenState<BankState>(s => s.Accounts[request.AccountId] = newBalance);
});
```

You can almost read the behavior off it. If the account doesn't exist, you should get a not-found and nothing changes. If the balance is too low, a bad-request, and again nothing changes. Otherwise the withdrawal goes through and the balance drops by the amount. That's all of Withdraw, in one place.

The rules are written against a *state*, and we get to choose what that state is. For a bank account it's nothing more than account IDs and their balances:

```csharp
[State]
public partial class BankState
{
    public Dictionary<string, decimal> Accounts { get; set; } = new();
}
```

No database, no HTTP plumbing, no retry logic - just the minimum you need to say what the expected behavior looks like. The contract describes *what* should happen; the real implementation worries about the *how*. Every operation follows the same shape - check the preconditions, state the expected response, state how the state changes - and the whole contract for the service fits a few pages.

A quick aside: this post isn't a tutorial on Accordant or on how to write these specs. The [docs](https://microsoft.github.io/accordant) cover that. Here I just want to talk about what happened once we had one.

Pulling the contract out of our tests had three consequences. The first I was expecting. The other two caught me off guard.

## It Became Legible

With every rule sitting in one place, we could finally read them next to each other - and a few inconsistencies turned up immediately now that we could _see_ it in one place. Deleting one kind of entity returned a 204; deleting another returned a 404 when it was missing. These kinds of inconsistencies were hard to spot in the tests and implementation but were staring us in the face when the contract was inspectable in one centralized place.

Just having the semantics written down, in one place we could review, was already quite useful.

## It Was Executable

Because the contract is executable, it can check *any* sequence of operations, not just the ones we thought to write. Our first reaction was that of excitement: suddenly we could fuzz inputs and throw arbitrary sequences at the specs and validate that our system worked as expected. This was powerful and continues to find bugs to this day.

But then we also realized that the separation between the scenario and the checks (something that traditional tests _intertwine_) paradoxically allows us to have a _lot more_ example-driven tests.

Examples are good: they're illustrative; a developer skims one and immediately gets how a feature behaves. We'd been a little stingy with them though, because each example was another test to write and maintain. But with the checking living in the contract, an example no longer needed its own bespoke assertions.

Here's what a test used to look like - the scenario and the checking tangled together:

```csharp
var r1 = await client.CreateAccount("alice");
Assert.True(r1.IsSuccess);

var r2 = await client.Deposit("alice", 100);
Assert.True(r2.IsSuccess);
Assert.Equal(100, r2.Balance);

var r3 = await client.Withdraw("alice", 30);
Assert.True(r3.IsSuccess);
Assert.Equal(70, r3.Balance);
```

Every one of those asserts is a little fragment of the contract, copied into the test and free to drift from the others. With the contract doing the checking, each step just hands its response to the spec:

```csharp
var profile = new StateProfile(new BankState());
bool ok; string msg;

var r1 = await client.CreateAccount("alice");
// assertions are encoded in spec.Allows
(ok, msg, profile) = spec.Allows("CreateAccount", new { accountId = "alice" }, r1, profile);
Assert.True(ok, msg);

var r2 = await client.Deposit("alice", 100);
// the same spec.Allows again
(ok, msg, profile) = spec.Allows("Deposit", new { accountId = "alice", amount = 100 }, r2, profile);
Assert.True(ok, msg);

var r3 = await client.Withdraw("alice", 30);
// and here again ...
(ok, msg, profile) = spec.Allows("Withdraw", new { accountId = "alice", amount = 30 }, r3, profile);
Assert.True(ok, msg);
```

It's the same test, only now the checking lives in the spec instead of in hand-written asserts. And once you notice that the only thing distinguishing this example from the next is the _data_ - the operations, the requests, and the responses we observed - there's no reason to keep it in C# at all. You can write the same example as JSON:

```json
[
  { "operation": "CreateAccount", "request": { "accountId": "alice" }, "response": { "statusCode": 201, "balance": 0 } },
  { "operation": "Deposit", "request": { "accountId": "alice", "amount": 100 }, "response": { "statusCode": 200, "balance": 100 } },
  { "operation": "Withdraw", "request": { "accountId": "alice", "amount": 30 }, "response": { "statusCode": 200, "balance": 70 } }
]
```

and replay it through a single loop:

```csharp
var profile = new StateProfile(new BankState());

foreach (var step in scenario)
{
    bool ok; string message;
    (ok, message, profile) = spec.Allows(step.Op, step.Request, step.Response, profile);
    Assert.True(ok, message);
}
```

That loop never changes. The same handful of lines now drives any number of examples - adding one is just appending to a file.

(There are some subtleties here. Sometimes a request depends on the response of an earlier call - you create an account, get back a server-generated id, and the next request needs to refer to it. A static JSON file can't hardcode an id it doesn't know yet, so you need a way to thread values forward through the sequence. It's a solvable problem with a bit of engineering, and Accordant has mechanisms for these _derived requests_; the [docs](https://microsoft.github.io/accordant) go into the details.)

This flipped the economics. Writing an example went from *author a test* to *jot down a sequence*, and the number of examples in our suite climbed by an order of magnitude. Not generated ones; examples our developers actually chose, as scenarios worth documenting and regressions worth pinning down. And each was now checked harder than before, against the full contract instead of the two or three asserts someone happened to type.

## We Could Reason With It

This last bit - the ability to _mechanically reason_ with the help of the contract was the most surprising, and most gratifying outcome.

Let's revisit those concurrency tests we mentioned in the beginning. When two (or three or four) operations run at the same time, you don't know which one the system processed first, second and so on. Before, a developer had to hold multiple orderings in their head and hand-write the outcomes for each. With the contract, we didn't have to. We could run each operation's spec in possible logical orders and ask whether at least one ordering explained the observed responses. If none of them explained the response, we had found a genuine bug. This was powerful and lead us to write (as well as mechanically generate) tons of concurrency tests, far more than was practical to write by hand. In fact, we often get failures that takes us a while to grok _why_ they are failing. Expecting developers to mentally simulate all these orderings and write tests around them just doesn't work - there's a reason we don't see such tests in our test suites.

This wasn't just limited to just concurrency. Imagine a situation where you perform an operation and you get a socket timeout. Maybe the operation never took effect (as the request never reached the server), or maybe it did (as the request reached the server but the response never got back to you due to a network glitch). These failures are sometimes called [_indefinite failures_](https://antithesis.com/docs/resources/reliability_glossary/#indefinite-error). Now imagine reasoning about valid outcomes involving a _combination_ of concurrency and indefinite failures. It gets complex - fast.

The surprising realization was that the developers did not have to grapple with this complexity when writing the specs. They just wrote the spec when the system was in a _single_ given state. That's easy to read and write. The compositional reasoning above effectively comes for free once you have written down the semantics as executable specs as the framework can do the bookkeeping of the set of states the system could be in and the various logical orders the operations could have run in. It's quite freeing once you experience it.

In addition to reasoning through possible scenarios during the validation phase, the contracts also allowed us to _simulate_ how the (logical) state of the system unfolds over time. If you look back at the code snippet above, you'll see that it is effectively a state machine (potentially an infinite state machine, but a state machine nonetheless). You get a request in _some state_ and you go to one (or more) next states. Apply every operation from every reachable state and a graph falls out - the states the system can be in, with operations as the edges between them. Here's one Accordant generated from the very bank account spec we've been looking at:

![State graph from the bank account spec](/images/bank-account-state-graph.png)

Walking that graph hands you test sequences for free: cover every state, exercise every transition, or steer toward some corner you want to probe. So the same spec works at both ends - it discovers interesting sequences to run, and then it does the complex reasoning to validate the responses.

## Have your cake and eat it too?

It's interesting to realize that all this was possible because of a subtle shift in how we wrote down the semantic contract. We were "half specifying" the contract in any case - just scattered around in lots of test cases. Stating the same contract as an executable contract lead to the following outcomes:

- **Legibility**: the behavior of the API written down in one place you can read and inspect.
- **Executability**: teasing the contract into an executable spec allowed us to validate arbitrary scenarios and paradoxically multiplied the number of example-driven tests in our test suite.
- **Generative**: the spec could be used not just to validate responses but also to generate interetsing sequences.
- **Reasoning**: concurrency and ambiguous failures turn into bookkeeping the spec does for you, freeing the developers from having to write such tests by hand.

It felt like we could have our cake and eat it too.

---

If this resonates, [Accordant](https://github.com/microsoft/accordant) is the framework we built around the idea - open-source, model-based testing for .NET. You write the contract as executable specs, and it takes care of validation, state tracking, the concurrency reasoning, and the test generation.
