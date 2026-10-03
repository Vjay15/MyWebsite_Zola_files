+++
title="Ordering Items for the Orderly"
date="2026-10-03"
template="blog.html"
authors=["Vjaylakshman K",]
+++

Hey my epic and cool friends on the internet, I hope you have been doing well! it's your junior engineer from last time who, ahem, made designing API keys sound like rocket science. I have finally become a full-timer (MTS @ [SparrowCRM](https://sparrowcrm.com)) and I am learning lots of cool things and also bashing my head against the wall sometimes.

This time I am going to explain some cool things I learnt about ordering items :D ! 

## The task
We use React Flow at work for all graph based systems in the product and I was assigned the task of supporting adding in between nodes in one of the features of SparrowCRM, Smart Router, which allows you to assign leads based on a preset of rules. Here is how it basically used to arrange the nodes in the graph before.

Each node was assigned a two-digit value such as 11, 12, 21, 22 etc so to roughly put, it looks like this 

![Nodes arranged in a grid with two digit values like 11, 12, 21, 22](/images/initial_nodes.png)


Very easy solution at that point, the ones place dictates the horizontal position whereas the tens position dictates the vertical position. This is the plain simple integer based ordering where you just order items as 1, 2, 3 and call it a day, but it falls short when you bring in adding or deleting in between nodes, when all positions are assumed to be integers, adding a node in between requires rebalancing the entire subtree from that node which is not optimal. This triggered the itch in me to look into how graphs are handled and by extension ordering.

## Let the React flow, flow.... 

Ideally, in React Flow based graph systems the nodes and the edges are free flowing, they are not constrained by the "This node should always be in this position" nor "This edge should always be a straight line". Nodes exist separately and you control which node is connected to what and where nodes should be, just like n8n. You just record the X and Y positions of each node and keep note of the edges and their source and target nodes. The position, the path taken is entirely dictated by the frontend, and the backend stores them. If in case you want to show them in order, you can add a handy dandy "Re-organize" button and use an existing re-organizing algorithm to calculate the position of the nodes such as ELK.

The unique case in my situation is that the system was based on deterministic node placements as points in a grid-like system. In this case letting React flow, flow was not really a choice for me since that meant refactoring the entire front end to support the new system and also how the backend handled the routing. One thing I learnt in engineering is that, make it simple, make it work then make it better. The current system works, but it's an ordered grid placement where order mattered. So instead of trying to force it into a new system, I tried looking at how we can use the existing system and provide the feature soon. Therefore I pivoted my research into how ordering is handled.

The first thing I did was decouple the position into separate horizontal and vertical elements since keeping them in a single integer would not help at all. Now we shall look into how we can approach this.

## What if 1 becomes 1000?

The decoupling alone wont help since, essentially they were just single digit numbers and since it's integer based ordering I would've to update every node downstream if I have to just add a node in between.
```
keys = [1, 2, 3, 4]

append:
    return last(keys) + 1                    // 4 -> 5

insert_at(i):
    // no integer exists between 1 and 2,
    // so shift every key from i onwards by 1
    for each key at position >= i:
        key = key + 1
    return keys[i]                           // take the freed slot
```

Instead what if I just made 1 as 1000 and 2 as 2000, now I have 1000 nodes to add in between!

![Integer ordering with a wide gap, nodes at 1000, 2000, 3000](/images/gap_integer_order.png)

One of the easiest approaches that can be taken but it comes with its shortcomings even if you can just take the midpoint of each of these as an integer you will eventually hit a point where you will lose precision, what if you want to add between 1000 and 1001 the rebalancing has to happen and this can be hit in literally 9 inserts, which is enough but not a lot and it would trigger frequent rebalancing which is not ideal. Let's improve on this 

The code version of that idea:

```
GAP = 1000
keys = [0, 1000, 2000]

append:
    return last(keys) + GAP                  // 2000 -> 3000

insert_at(i):
    left  = keys[i - 1]
    right = keys[i]

    if right - left <= 1:                    // like 1000 and 1001, no room
        keys = respread all keys GAP apart   // the rebalance
        return keys[i]

    return left + (right - left) / 2         // 1000, 2000 -> 1500
```

## Here comes Fractional Indexing

```
keys = [1.0, 2.0]

insert_at(i):
    mid = (keys[i - 1] + keys[i]) / 2        // 1.0 and 2.0 -> 1.5

    if mid did not move:                     // double ran out of precision
        keys = spread all keys evenly        // the rebalance
        return keys[i]

    return mid
```


What if we throw away the approach of using integers and instead head the path of floats, this is one of the approaches taken by Figma as well ([Realtime Editing of Ordered Sequences](https://www.figma.com/blog/realtime-editing-of-ordered-sequences/)). Given two integers such as 1 and 2, we keep dividing them to find the midpoint, easy and they give floats that sit well in between the two nodes.

![Float based fractional indexing, finding the midpoint between two nodes](/images/float_order.png)

But there is a problem where the floats can eventually reach the precision wall; currently Postgres doubles and JavaScript floats support 53 bits of precision. This could approximate it to around 16 digit precision combining both the whole numbers and the decimals. Therefore we need to keep a check whether the newly generated index is in between the previous and the next node's index. 

In case it reaches that scenario, we need to just rebalance aka take the already existing indexes of all elements and redistribute the indices to 1, 2, 3, 4 etc. I also combined the approach of having integers spaced a bit wider like 1000, 2000, 3000 etc. With this approach I got approximately 53 successive inserts in an interval before it hit rebalance. This solved most of my problems. But I delved deeper.


## Fractional Indexing but make it strings??

For a system which has frequent inserts and deletes happening at all times, Fractional Indexing based on floats is not scalable: they will hit the 53 insert mark so quickly and would have to rebalance often, and this will result in the system becoming slower overall.

Behold the holy grail of fractional indexing articles, [David Greenspan's notebook](https://observablehq.com/%40dgreensp/implementing-fractional-indexing). There is no article on ordering without a mention of David Greenspan's notebooks or Rocicorp's fractional-indexing package. Both follow a similar approach which I will try to put it as simply as possible.

Instead of relying on the ordering of integers and floats, we move to a string based ordering system, where 0 is "0" instead. Seems quite simple but without proper guardrails, this could become complex very fast. So let's go over the basic rules of Greenspan's algorithm

1. The strings start with a, b and c, usually where a-prefixed keys will have a single integer value after it and b will have two integer values after it and so on. ex: a0, b00, c000. The reason is that let's say we reach az and we want to append another element, then if we try to do a roll over and get a00, this key won't satisfy the constraint since "a00" is not greater than "az". 
2. When inserting between two keys, factor out the common prefix and find the midpoint of the differing part. This ensures the keys don't collide and keep growing.
3. If the midpoint has to be found between adjacent neighbors, let's say a and b, we instead roll it over to the next character aka midpoint("a","b") = aV, the reason being V is the midpoint of the base62 characters.
4. The trailing zeroes are banned, the reason being that given we have to find a midpoint between "2" and "20" we then try to find the midpoint of (null, 0) since 2 is a common prefix but in base62 0 is the first element, and even if we try to roll over, 2z is not less than 20.
5. The add before a0 is also handled specially since it would be a midpoint of the (null, a0) case, here we will then go for Zz, Zy etc since Z is before a. The append and prepend are called the sentinel cases.

```
keys are strings: "a0", "a1", "a0V" ...

append(a0)   -> "a1"
prepend(a0)  -> "Zz"
between(a0, a1) -> "a0V"      // there is ALWAYS a longer string that fits

// which is why there is no rebalance here, ever
```

![Lexicographic fractional indexing with base62 string keys](/images/string_order.png)

This is such a beautiful and wonderfully scalable approach. The rebalance trigger is no more losing precision, but the guardrails we set, we are free to choose the rebalance criteria and if it has to be rebalanced, it just reassigns new indices for each node.

I tried to reason this out with Greenspan's notebook but was left with some confusion and looked into Rustam's article which cleared most of my doubts, it's a great read: [Fractional Indexing: Ordering Items in Collaborative Lists](https://n69.in/blog/fractional-indexing/)

Atlassian's LexoRank is built on top of this core machinery, but they add extra features to it. A rank of an item in Jira looks like this 

![An Atlassian LexoRank key with a bucket number, separator and rank string](/images/lexorank.png)

The last part is the string which we are familiar with and there is a separator and a number. This number denotes the bucket to which the current items belong. When a rebalance occurs, the values in the 0th bucket will be moved to the 1st bucket and so on. There are three buckets so far: 0, 1 and 2. This can be very useful in case we want to recover from a failed rebalancing. 

The specifics of how exactly LexoRank works seems quite close to the lexicographic indexing albeit with it's own features. Although I am not 100% sure if they follow the same approaches.

## Why not Linked Lists?

Linked List would be another natural candidate for this but I didn't include it in this article, the reason being that to order something properly we have to traverse the whole linked list, and the implementation would be complex. Let's take for example I need to validate an addition of an item based on a parent of this branch of items. I need to traverse the list backwards to find it out. It could be easily tackled with our fractional indexing approaches by just order based on the index and find the first element.

## Benchmarks:

Finally I wanted to see how all of them compare against each other. I ran everything against a real SQLite file instead of in-memory arrays, and measured each insert as TAT, the time from generating the key to the row being saved in the db. There were two workloads: a mixed one that seeds a 6x6 grid (my 11/12/21/22 setup split into separate x and y) and does 300 random inserts on both axes, and a squeeze one where every insert goes into the same gap between two nodes. Each strategy gets squeezed from keys it would naturally hold: integer starts dense at 1 and 2, gap and floats start at 1000 and 2000, and strings start at a0 and a1. For the string approach I used Rocicorp's [fractional-indexing](https://www.npmjs.com/package/fractional-indexing) package instead of implementing Greenspan's algorithm myself.

In simple words: mixed is the normal case, a graph like a real router where inserts happen all over the place. Squeeze is the worst case, just two nodes with every insert going into the same gap between them.

> Disclaimer: Since this was just a disposable test, I used an LLM to generate the tests, ran the experiment and I manually verified them.


| Strategy | Avg TAT (mixed) | Max TAT (mixed) | Rebalances (mixed / 300) | Inserts survived before 1st rebalance (squeeze) | Rows updated in 1st rebalance | Rebalance latency | Longest key |
|---|---:|---:|---:|---:|---:|---:|---:|
| **integer** | 11.3 ms | 25.0 ms | **290 / 300** | 0 (dense 1,2) | 1 | 4–8 ms | 4 |
| **gap-1000** | 4.9 ms | 17.3 ms | 2 / 300 | 9 | 11 | 6–9 ms | 6 |
| **fractional-float** | 4.7 ms | 11.3 ms | 0 / 300 | **53** | 55 | 4–12 ms | 24 |
| **string-fractional (unbounded)** | **4.6 ms** | 14.0 ms | 0 / 300 | 3000+ (never) | — | — | 502 |
| **string-fractional (32 char capped)** | 4.2 ms | 12.9 ms | 0 / 300 | 180 | 181 | 6–27 ms | 5 |

Some observations:

- integer rebalanced on 290 out of 300 inserts. Since integer keys are always dense (1,2,3...), there is no gap to begin with, and the squeeze probe could not do even one insert before a rebalance.
- gap-1000 gives you around 9 inserts per gap before a respread. To see what that means in practice: all strategies ran the same 300 inserts, but integer ended up writing ~23,000 rows to the db to do it (renumbering the tail on almost every insert), while gap-1000 wrote only 407. Same work, 60x fewer writes.
- fractional floats survived 53 inserts between 1000 and 2000 before the midpoint collapsed onto a neighbour, which lines up with the 52-bit mantissa of doubles. After the respread every gap gets fresh room, so the next failure took over a thousand more inserts, in a different gap this time.
- string fractional (unbounded) did not rebalance even once. 3000 inserts into the same slot and the keys just keep growing (up to 502 chars).
- string fractional with a 32 char cap behaves like the column is VARCHAR(32). If a new key does not fit in 32 chars, we rebalance. In the squeeze this happened every ~181 inserts, and in the mixed workload it never happened at all, keys stayed at 5 chars.

Why 32 chars? Shorter keys mean a smaller db index and a smaller json payload for the frontend. With 500 nodes, going from 3 char keys to 256 char keys takes the db table from 28 KB to 300 KB and the payload from 137 KB to 381 KB. So there is a real cost to letting keys grow. But the rebalance speed does not really change with key length (it is ~20 to 30 ms either way, it depends on the number of rows, not how long they are). 32 chars is just a comfortable middle ground, keys stay small and normal inserts never come anywhere close to the limit.

The timings may vary between runs due to db write latency, but they capture most of the real life scenario. I also checked the read path, how long it takes to build the graph back from the stored keys. Even with 10,000 nodes all four strategies finish within a couple ms of each other, so the ordering strategy mostly matters on the write path.

Eventually I settled on the float based fractional indexing approach, reason being that I just wanted to use what seems to be easy to understand, yes lexicographic approach is understandable and I could implement it but one thing I learnt from my earlier approaches is to keep things simple. Along with that, for my use case, a user is going to build a router and then is probably not going to use it for quite a long time so frequent inserts are not a concern of mine. Use the right tool for the right job.

## Conclusion:
I never thought that ordering items could be this thoughtful haha, but I learned so much that I felt I had to write an article on it. Thanks for reading through this. Hope you all have a great day :)

> AI USAGE DISCLAIMER: I used AI in order to find articles apart from whatever I could do via a search engine, help me visualise and explain and understand tradeoffs. For writing I wrote the initial draft fully by hand and then used AI to fact check my claims and fix my silly spelling mistakes. And the test bench is also fully AI generated with human review.
