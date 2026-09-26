+++
title="Ordering items for the orderly"
date="2026-09-26"
template="blog.html"
authors=["Vjaylakshman K",]
+++

Hey my epic and cool friends in the internet, it's your junior engineer from last time who ahem made designing API keys sound like rocket science. I have finally become a full timer (MTS @ [SparrowCRM](https://sparrowcrm.com)) and I am learning lots of cool things and also bashing my head against the wall sometimes. I hope you are doing well :)

This time I am going to explain some cool things I learn about ordering items! 

## The task
We use React Flow at work for all graph based systems in the product and I was assigned the task of supporting adding in between nodes in one of the features of SparrowCRM, Smart Router which allows you to assign leads based on a preset of rules. Here is how it basically works.

Each node was assigned a two digit value such as 11, 12, 21,22 etc so to roughly put, it looks like this 

ADD AN IMAGE HERE

Very easy solution at that point, the ones place dictates the horizontal position whereas the tens position dictates the vertical position. This is the plain simple integer based ordering where you just order items as 1,2,3 and call it a day, but it falls short when you bring in adding or deleting in between nodes, when all position is assumed to be integers, adding a node in between requires rebalancing the entire subtree from that node which is not optimal. This triggered the itch in me to look into how graphs are handled and by extension ordering.

## Let the React flow, flow.... 

Ideally, in a React flow based graph systems the nodes and the edges are free flowing, they are not constrained by the "This node should always be in this position" nor "This edge should always be a straight line" Nodes exist separately and you control which node is connected to what and where nodes should be, just like n8n. You just record the X and Y positions of each node and keep note of the edges and their source and target nodes. The position, the path taken is entirely dictate by the frontend, and the backend stores them. If in case you want to show them in order, you can add a handy dandy "Re-organize" button and use an existing re-organizing algorithm to calculate the position of the nodes such as ELK.

The unique case in my situation is that the system was based on deterministic node placements as points in a grid-like system. In this case letting React flow, flow was not really a choice for me since that meant refactoring the entire front end to support the new system and also how backend handled the routing. One thing I learnt in engineering is that, make it simple, make it work then make it better. The current system works, but its an ordered grid placement where order mattered. So instead of trying to force it into a new system, I tried looking at how we can use the existing system and provide the feature soon. Therefore I pivoted my research into how is ordering handled.

The first thing I did is decouple the position into separate horizontal and vertical elements since keeping them in a single integer would not help at all. Now we shall look into how we can approach this

## What if 1 becomes 1000?

Now that our X and Y indices are separate the issue at hand is simple, we have 1 and 2 in between we cant add integers, let's make it 1000 and 2000, now we have a staggering 1000 nodes to add in between! One of the easiest approaches that can be taken but it comes with its shortcomings even if you can just take the midpoint of each of this as an integer you will eventually hit a point where you will lose precision, what if you want to add between 1000 and 1001 the rebalancing has to happen and this can be hit in literally 9 inserts, which is enough but not a lot and it would trigger frequent rebalancing which is not ideal. Let's improvise on this 

## Here comes Fractional Indexing

What if we throw away the approach of use integer and instead head the path of floats, this is one of the approaches taken by Figma as well, since now we can work with floats or in my case doubles, we have around 16 digit precision which when I repeatedly insert a node in between 1000 and 2000, it allows me to go till 53 inserts approx, it may vary based on the position of the top and bottom but its much more room than initially we had, and very much better too. The rebalancing would be less frequent and adding and deleting much much trivial. Yippee! This was the approach I took and it proved to solve all my issues I had with integer based positioning. Problem solved yes? true but what if we could go even deeper.

## Lexorank!
