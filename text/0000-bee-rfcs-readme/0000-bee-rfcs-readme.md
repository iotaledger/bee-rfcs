+ Feature name: `BEE RFCs readme motivation change`
+ Start date: 2019-09-26
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC proposes a change to the RFC process itself. Particularly the `Motivation` section of the RFC template to put more emphasis on the need to write clear requirements and how to write them.

# Motivation

+ A more consistent and better means of capturing the motivations driving the changes proposed in RFCs across all projects.
+ Ensuring the changes proposed in RFCs are thought through from the user and business perspective.
+ Stakeholders should be involved in the RFC process as early as a new RFC is created. A well defined motivation ensures that ideas can be easily validated before the design is scrutinized.

# Detailed design

Replace the current motivation in the template:
```
Why are we doing this? What use cases does it support? What is the expected outcome?
```

With the following:
```
Why are we doing this? What use cases does it support? What is the expected outcome?

1. Write a summary of the motivation.
2. List all the specific use cases that your proposal is trying to address. Write them from the perspective of the person who will be using the software. We recommend using the "Job story" format, which provides context, motivation, and the desired outcome.

When _____ , I want to _____ , so I can _____ .

Example 1:
When I query a node for a list of transactions, I want to be able to sort them by date, so I can work with the most relevant ones.

Example 2:
When I configure a node, I want to be able to control how much transaction history the node stores, so I can make sure I only store the data I need without incurring additional operational costs.

```

The suggested format is an evolution on a user story format. It may not apply to all RFCs, but I'd encourage everyone to try and think of any functionality with this framing.

# Drawbacks

- The motivation part of RFCs will take longer to write up.
- Larger RFCs may tackle large amounts of 'jobs'. In which case it might be beneficial to either break down the RFC, or list only higher level motivations.
- For the current implementation of 'known components', this may seem redundant. If the RFCs motivation is really clear, however, it should not be an issue to write such a motivation.
- If it's an internal component, it may be difficult to think from the user perspective. However, even internal components need to tie into a specific scenario that they allow.

# Rationale and alternatives

- The job story approach forces thinkink from the perspective of the person who will be using the product.
- Standard requirement format was considered, but it allows for too much vagueness.
- Acceptance criteria format was considered, but that is mainly suitable for products with UI or very UX heavy products. 
- What is the impact of not doing this?: Not doing this I'm afraid will lead to incosistent motivation specifications and disallow us from quickly validating individual RFCs from the motivation perspective.

# Unresolved questions

- Do we want to enforce this in a format like this? Or should we use a more loose format?
