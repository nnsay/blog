---
title: Make effective communication
date: 2019-12-04 11:35:00
tags:
- 工作
- 效率
excerpt: how to do an effective communication in the office?
---

### 1. What is the best result
- the best publish time is yesterday.
- the best code is that has not been written yet.

### 2. About design/solution
the factors for a better design:
- layered
    - basic targets
    - optional targets
- benefits
- drawbacks

when talk about the `benefits` and `drawbacks`, we can evaluate from the following aspects:
- compliance
- performance
- extensionable
- understandable
- devops
- implement time

### 3. How to discuss about the design
Sometime we need to discuss your design or solution with other people. In order to save time, a good steps you can follow:

3.1 Does other people have known the background about the design?
- no, stop
- yes, go on

3.2 Do your have same design `basic targets`?
- no, try to make a common target
- yes, go on

3.3 Design clarification
- you give your desgin, basic targets/optional targets/code  ...
- other people give their design, idea/sugguestion/code ...
- keep the same part, find out the different part, list the the different part
- foreach the different part list
    - what is the other pepole design benefit?
    - do you agree?
        - yes, adopt
            - do the new suggestion bring out new probleme?
        - no, discuss
- summary the sugguests what the both sides agree with
- reimplement
- checking

### 4. The skill for communication
- make some samll and core questions about you design to confirm the other people whether understands your design.
- ask for the drawback of your design  or the benifit of other people's design
- ask question to make the other people think instead of always explaining the yourself design
- invite a third part to join the discussion
- summary the common thoughts


### 5. Summary the common thoughts
- batch processing should a fault-tolerant, eg: batch deleting
- replace the trick with a common way, eg: async load config before user module
- keep the code as simple as possible but be not a magic for everything