---
layout: post
title: Play with SVG Paths in Canvas with AndroidFillingLoaders
---

We usually don't like the internal drawing logic from the Android SDK too much. When we read about it 
we use to feel weird about it, as it seems to be a little bit tedious. But it is not that hard if you 
read it carefully, and if you are capable of understanding it properly, you will end up creating really 
interesting figures and animations like the following one:

<div style="text-align:center">
![Small gif][1]
</div>

The previous animation has been extracted from the [AndroidFillableLoaders library](https://github.com/JorgeCastilloPrz/AndroidFillableLoaders) 
which was published by me like two days ago. The lib wants to create an interesting filling effect for a custom silhouette 
created by a given SVG Path, and it is totally open to be extended by the Android community.

[1]: https://raw.githubusercontent.com/JorgeCastilloPrz/AndroidFillableLoaders/master/art/demoSmall.gif