---
title: zero knowledge proof
date: 2017-11-23 14:52:05
tags:
---
## 零知识证明

读一读这篇论文[零知识证明及其应用](http://www.doc88.com/p-1327381288501.html)。虽然这篇论文不过是一些资料的复制粘贴，但好在是中文的。

>零知识证明是一种认证协议，A向B证明自己拥有“某种东西”，而不会给B有使用自己特性的机会，也就是让B拥有向别人证明他是A的能力。

零知识证明涉及到的数学过程看不懂，只能理解下概念（在论文的第17,18页）。

另外[wiki](https://en.wikipedia.org/wiki/Zero-knowledge_proof)也做了很好的说明：
> A zero-knowledge proof must satisfy three properties:
>1. Completeness: if the statement is true, the honest verifier (that is, one following the protocol properly) will be convinced of this fact by an honest prover.
>2. Soundness: if the statement is false, no cheating prover can convince the honest verifier that it is true, except with some small probability.
>3. Zero-knowledge: if the statement is true, no cheating verifier learns anything other than the fact that the statement is true.

提供了两个经典的例子：
* The Ali Baba cave
* Two balls and the colour-blind friend

以及实际的应用：
* Discrete log of a given value
* Hamiltonian cycle for a large graph

我的理解是：Alice掌握了一个秘密，可能是一个秘钥，可能是一个方程解，可能是某种证明过程，可能是某种配方，她可以选择直接给你看秘密，也可以通过更聪明的方式，间接地让你相信她真的知道秘密。 Alice选择一些随机数，并且代入随机数和秘密将现有问题转换成另一个问题，然后提供另一个问题的解做为承诺，Bob可以选择发送另一些随机信息给Alice，Alice对信息处理后回传，Bob对回传信息和承诺加以比对，如果能在数学证明上一致性，就加1分。重复这个过程多次，如果积累的分数足够高，Bob就可以相信Alice是诚实的。在这个过程中，问题的选择很重要，一是要使得Alice作弊的可能性非常低，二是Bob不能假装自己就是Alice转而欺骗Dave。  但零知识证明也不是无懈可击的，比如如何规避中间人陷阱。


