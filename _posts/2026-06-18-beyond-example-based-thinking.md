---
layout: post
title: The Problem With Example-Based Testing Isn't the Examples
---

*What happened when we teased the semantics out of the scenarios.*

I remember looking at the test suite the team had put together - around 500 integration tests covering our API surface area. Alongside the usual happy-path and negative cases, we even had a few that invoked the APIs concurrently: create two items at once and assert that only one call should succeed, or create and delete an item at the same time and assert that they should happen in _some_ logical order (the item should never be in half created/deleted state). Those concurrent ones were a mind bender, because what you observed depended on the logical order the operations ran in. A developer had to simulate the orderings in their head and write the assertions for each. Overall, we were reasonably proud of our test suite.

Yet I still had a nagging feeling. Every time I looked at the suite I could spot gaps: negative cases we hadn't written, interaction patterns we hadn't covered. I could have continued to nag the team to write more tests but every test case is code that has to be written and maintained and there's a natural ceiling on how many you'll add before the cost stops feeling worth it.

Secondly, as I read the tests, I kept seeing the *semantic contract* of the API hiding between the lines. The tests were telling a story:

- You cannot put inventory without setting up the SKU first.
- You cannot take more items out of a warehouse than you put in.
- You cannot delete a SKU with active references.

The contract was **everywhere in the test suite, yet nowhere in particular**. Ideally we'd have written a design doc spelling it out, but we hadn't, so the test suite was the next best thing.

This naturally led to the question: can we *tease apart* that contract from the tests, and state it directly?

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

You can almost read the behavior off it. If the account doesn't exist, you should get a not-found and nothing changes. If the balance is too low, a bad-request, and again nothing changes. Otherwise the withdrawal goes through and the balance drops by the amount. That's all of Withdraw, in one place[^state-machine].

The rules are written against a *state* - the minimal bit of information you need to pin down the semantics. For a bank account it's nothing more than account IDs and their balances:

```csharp
[State]
public partial class BankState
{
    public Dictionary<string, decimal> Accounts { get; set; } = new();
}
```

No database, no HTTP plumbing, no retry logic - none of the machinery the real implementation worries about. The contract describes *what* should happen; the implementation worries about the *how*. Every operation follows the same shape - check the preconditions, state the expected response, state how the state changes - and the whole contract for the service fits a few pages.

<div class="callout" markdown="1">
**Aside** — this post isn't a tutorial on Accordant or on how to write these specs. The [docs](https://microsoft.github.io/accordant) cover that. Here I just want to talk about what happened once we had one.
</div>

Pulling the contract out of our tests had three consequences. The first I was expecting. The other two caught me off guard.

## It Became Legible

With every rule sitting in one place, we could finally read them next to each other - and a few inconsistencies turned up immediately now that we could _see_ it in one place. Deleting one kind of entity returned a 204; deleting another returned a 404 when it was missing. These kinds of inconsistencies were hard to spot in the tests and implementation but were staring us in the face when the contract was inspectable in one centralized place.

Just having the semantics written down, in one place we could review, was already quite useful.

## It Was Executable

Because the contract is executable, it can check *any* sequence of operations, not just the ones we thought to write. Our first reaction was excitement: suddenly we could fuzz inputs and throw arbitrary sequences at the specs and validate that our system worked as expected. This was powerful and continues to find bugs to this day.

This is usually where the story stops. Model-based and property-based testing are pitched as the alternative to the example-based tests most of us write: stop exercising concrete scenarios with specific checks, write a spec or an invariant, and let the machine generate fuzzed sequences instead. I think that framing is a bit of a false dichotomy. Examples are good - they're how a developer skims one case and immediately gets how a feature behaves. **The problem was never the examples. It was that each test fused the example and its checks into a single artifact.**

And once we'd pulled those apart, something paradoxical happened: **we ended up with a lot more examples, not fewer.**

We'd been a little stingy with them, because each example was another test to write and maintain. But with the checking living in the contract, an example no longer needed its own bespoke assertions.

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

It's the same test, only now the checking lives in the spec instead of in hand-written asserts. And once you notice that the only thing distinguishing this example from the next is the _data_ - the sequence of operations and their requests - there's no reason to keep it in C# at all. You can write the same example as JSON:

```json
[
  { "operation": "CreateAccount", "request": { "accountId": "alice" } },
  { "operation": "Deposit", "request": { "accountId": "alice", "amount": 100 } },
  { "operation": "Withdraw", "request": { "accountId": "alice", "amount": 30 } }
]
```

A tiny driver reads each step, makes the actual call against the running system, and hands the real response to the spec:

```csharp
var profile = new StateProfile(new BankState());

foreach (var step in scenario)
{
    var response = await client.Call(step.Operation, step.Request);

    bool ok; string message;
    (ok, message, profile) = spec.Allows(step.Operation, step.Request, response, profile);
    Assert.True(ok, message);
}
```

That loop never changes. The same handful of lines now drives any number of examples - adding one is just appending to a file.[^derived]

This flipped the economics. Writing an example went from *author a test* to *jot down a sequence*, and the number of examples in our suite climbed by an order of magnitude. Not generated ones; examples our developers actually chose, as scenarios worth documenting and regressions worth pinning down. And each was now checked harder than before, against the full contract instead of the two or three asserts someone happened to type.

## We Could Reason With It

This last bit - the ability to _mechanically reason_ with the help of the contract - was the most surprising, and most gratifying, outcome.

Let's revisit those concurrency tests we mentioned in the beginning. When two (or three or four) operations run at the same time, you don't know which one the system processed first, second and so on. Before, a developer had to hold multiple orderings in their head and hand-write the outcomes for each. With the contract, we didn't have to. We could run each operation's spec in possible logical orders and ask whether at least one ordering explained the observed responses. If none of them explained the response, we had found a genuine bug. This was powerful and led us to write (as well as mechanically generate) tons of concurrency tests, far more than was practical to write by hand. In fact, we often get failures that take us a while to grok _why_ they are failing. Expecting developers to mentally simulate all these orderings and write tests around them just doesn't work - there's a reason we don't see such tests in typical test suites.

This wasn't limited to concurrency. Fault injection is another case. Say you do a create and get a socket timeout. Maybe the operation never took effect (the request never reached the server), or maybe it did (the request landed but the response got lost on the way back) - these are sometimes called [_indefinite failures_](https://antithesis.com/docs/resources/reliability_glossary/#indefinite-error). So now you're tracking two possible states. Do another write that also times out, and each of those branches splits again - now you're reasoning about four. And if that weren't enough, layer concurrency on top, where the operations could also have been applied in any order.

Here's the part I find delightful: the developer writing the spec never has to think about any of this. You write each rule against a _single_ given state - that's easy to read and easy to write - and the framework does the bookkeeping, fanning out across the set of states the system could be in and the orders the operations could have run in. The combinatorial reasoning comes for free. It's quite freeing once you experience it.

In addition to reasoning through possible scenarios during the validation phase, the contracts also allowed us to _simulate_ how the (logical) state of the system unfolds over time. If you look back at the contract above, you'll see that it is effectively a state machine. You get a request in _some state_ and you go another state (or loop back to the same state). If you start with a set of operations and transitively apply them in each reachable state (up to some bound), a state graph falls out. Here's one Accordant generated from the bank account spec we've been looking at:

![State graph from the bank account spec](/images/bank-account-state-graph.png)

Walking that graph results in meaningful test sequences that aren't purely random but drive the system to interesting states. You can also begin to ask questions about state and scenario coverage instead of just code coverage.

## It All Came From One Move

Looking back, everything in this post came from one move: stating the contract directly, as executable code, instead of leaving it scattered across hundreds of tests. That one shift led to the following:
    
- **Legibility**: the behavior of the API written down in one place you can read and inspect.
- **Executability**: validating arbitrary scenarios, and paradoxically multiplying the number of example-driven tests in our suite.
- **Generative**: the same spec generating interesting sequences, not just checking the ones we wrote.
- **Reasoning**: concurrency and ambiguous failures turning into bookkeeping the spec does for you.

All I'd wanted was to state the contract directly, in one place. I didn't expect that one move to buy us as much as it did.

---

[Accordant](https://github.com/microsoft/accordant) is the open-source .NET framework we built putting these ideas to work on a large Azure service at Microsoft. It descends from a rich lineage of model-based testing work there - Spec Explorer, Spec#, NModel, and others.

[^state-machine]: The astute reader might immediately notice that we're defining a state machine above which says the next state we should transition to when given a particular request in some state.

[^derived]: It's not always quite that simple. Sometimes a request depends on the response of an earlier call - you create an account, get back a server-generated id, and the next request needs to refer to it. A static JSON file can't hardcode an id it doesn't know yet, so you need a way to thread values forward through the sequence. It's a solvable problem with a bit of engineering, and Accordant has mechanisms for these _derived requests_; the [docs](https://microsoft.github.io/accordant) go into the details.
