---
published: true
category: AI
title: 从Vibe Coding到Agentic Engineering
author: smona
date: '2026-10-19 10::00'
layout: post
---

用 claude code 搓的项目： https://github.com/silbertmonaphia/nyx  
plan+review我用的是glm的coding plan，impl我用的是deepseek的api,coding plan总觉得有点坑，还老是得跟着它5小时窗口走（动不动就没token了），个人感觉还是deepseek这种只提供接口没有coding plan的良心一点，而且ds的api便宜是真便宜

Gemini cli到Claude Code用了好几个Coding Agent，从一开始vibe coding到现在进入agentic engineering研发深水区，本质上对于agent coding就3个问题：  

# １、AI写的代码怎么敢上线？

答案是：Harness，感觉业界都说烂了，但是对，就是它  

信心 = 双AI对抗 + SDD + TDD + 人肉兜底  

A、开发 和 test_review 的agent是不同AI模型，做AI对抗，比如开发的是deepseek-flash的api，test_review的是glm的模型，当AI成吨成吨产出代码，现在业内共识就是：人类肉眼根本不可能review得过来，只能用AI去review AI。否则软件开发速度的瓶颈就卡在了人类review这里  

B、SDD: Spec驱动开发，引入openspec(opsx)，需求先提交 opsx，越小越原子越好AI最擅长小而具体的场景。然后人类针对 /opsx:propasal 的方案做质询，其实我很多时候都不会直接去看opsx原文了，都是让AI告诉我它这opsx怎么设计的，然后把一些边界条件质询的办法去质询AI为什么这么设计如果xxx时候怎么样，其实有点类似原本在小团队当Lead的时候和2个组员讨论设计的时候，类似苏格拉底提问的方式进行的，设计这一步挺重要的，一个完整详尽的opsx可以让后面开发agent实现后不至于返工太多，返工都是token烧钱啊，所以一开始做plan设计的时候反而相对后面返工来说代价更低，对需求的设计要多花功夫，而后面agent去loop自己开发写代码测试的时候就可以不用怎么盯着，等它开发完再通知我。  

一旦opsx这份spec完备了，接下来就是 /opsx:apply 让 开发agent从opsx领取任务开发，开发的时候 test_review agent也是根据这个 opsx 去写测试，开发agent 开发完成后交给 test_review agent去 review，这个让其他agent review code的话，其实claude code最近2026年9月上的agent之间通信的功能就挺好，感觉是从还在实验阶段的agent team里面剥离出来的，比agent team更可控一点，开发agent完成了通过这个mailbox发送给 test_review agent去 review,test_review agent就切subagent哼哧哼哧地去review了  

所以日常开发在opsx这层就变成了: /opsx:propose => /opsx:apply => /opsx:sync => /opsx:archive  

C、TDD：测试驱动开发(unitetest, lint, e2e test,perf test,security test) 这些原本AI编程之前这套软件的质量保障体系，现在还是很重要。lint这块想提一点，最近看了pstack的作者  Lauren Tan 分享的她的lint也有一个：就是不让agent写代码的时候加代码注释，我也深有体会，最近开发过程中就遇到 无论是 开发agent 还是 test_review agent 都因为读到代码的注释导致对那块代码产生了误读，代码即注释，注释原本是给人类看的，反而对AI来说造成不必要的负担。  

而对agent开发当然也有它自己评估手段，比如golden set eval  

D、人肉：对，最后其实还是得人肉验证一次产品行为你才放心，即便有了前面SDD+TDD，可以是 产品PM 或者 测试QA 去亲自人肉对着 基于需求理解去验一次，这步还是必不可少但是可以AI自动化一大部分比如测试用例，前面SDD描述再详细，AI最后落地逻辑总会有些许偏差。很多时候都不需要产品或者测试，开发者自己点开功能，点两下就发现有地方实现和spec期望不太一样。但是不得不说AI对很具体的问题，比如带有报错信息全面的bug，修起来又快又准是真的。  

# 2、怎么避免不必要的token浪费？

A、(每日)养成经常管理上下文的习惯 /context /clear /compact ：/context 看当前上下文占用谁最多，/clear 一般是一个任务做完了干净了直接/clear清掉拿到一个新的上下文， /compact一般是不想清理前面上下文的时候会压缩一下，一般claude code会有自己的自动上下文压缩，但是如果到了它自己预设的 85%, 95%的压缩阈值，其实前面context已经有很多无关信息了，不仅token消耗大，还会让agent focus偏移也就是所谓的drift，一般现在大模型的上下文窗口1M，但是到50%左右上下文占用就开始drift了，所以你会看到各家厂商虽然说自己模型支持1百万的上下文窗口，但是API文档里面CLAUDE_CODE_AUTO_COMPACT_WINDOW一般只设置到50万到70万  

B、(每月)记忆的精简：比如claude code的记忆是3层，个人CLAUDE.md + 项目的CLAUDE.md+它自己的MEMORY索引： 个人和项目CLAUDE.md会膨胀，所以定期把它精简到200行以内，然后MEMORY也会膨胀，而且claude code截止202510月的最新版还是没有自动整理MEMORY的功能，我自己是写了个每个月claude code自己的cron做一次MEMORY的整理把过时的错误的记忆纠正以及精简记忆  

C、(每月) /doctor + /insight：/doctor 能让claude code自己分析自己skill,setting,CLAUDE.md等是不是太长了或者太久了做优化，而 /insight 会基于最近的200次会话分析出一份html报告让你知道token消耗在哪里  

# 3、提高并行度？
1主1副=>到1主2副=>到1主4副，一步步拉高，而不是一步到位，每一步感受单个任务大概到多大多复杂的程度AI实现比较好，哪里AI会犯错然后再让它自己总结更新到它自己记忆外加更新lint，一点点不断更新lint和它自己记忆过程中把并行拉上去  

提高并行度这件事得在，前面第二点之后做，或者和第2交叉做，并行度意味着token消耗量会开始直线上升，所以我建议是先做好token消耗的管理工作，也就是熟悉好上下文的管理后才开始提高并行度。  

可以先从1主1副=>到1主2副=>到1主4副，现在业界的观点就是甜点区间在 4-6个 agent 并行，超过7个边际效用递减了，多agent的peer之间沟通摩擦会显著上升token消耗量显著上升，而产出效果却没有相应的提升，而且并行开发的代码合并发生冲突概率也显著增加问题  

我的路径一开始 只有一个claude code开发，然后它开发loop的时候其实我是闲着的，这时候我就切1个新的claude coded去做新的设计落地opsx的propasal，然后基本我这边设计完了，它那边也做完了，然后它又可以去做我这个和AI一起设计好的新的opsx，这就是1主1副手。  

然后我觉得1主然后对2副也可以，每个副手自己领取一个opsx的slic切开worktree去loop开发，然后1主做它们之间的协调，外加 review 2个副手的实现，这个也还可以  

现在我已经开到了1主3副，还在试验中，但是感觉明显比1主2副要吃力，因为agent彼此之间会同步和回复，所以我和1主的plan做设计的对话的时候会被插入打断，现在还在努力适应中，可能不能每个副手都是并行去实现opsx的slice，我现在就是3个副手agent每个各自领取一个opsx的slice并行开worktree去做，而是比如 2个agent做并行实现，1个test开发和review，把写test和review的职责从的从1主身上剥离出去，1主就只做协调其他3个agent，以及和我一起做下一个opsx slice的设计plan就没了  

也就是说并行度可以从 业务线、职能线 这两个维度去切分agent，业务线角度就是每一个agent单独负责一个opsx slice所有前端和后端全部开发，职能线就是每一个agent都有自己专门的工作比如前端、后端、测试、review。这种划分其实和公司里面业务-职能划分是类似的，多个业务线对应多个opsx slice，每个业务线里面有多个职能团队对应不同 前端、后端、测试、review agent等  

提高并行度避免冲突的办法现在有2个：  
- 一个是 subagent + worktree  
- 另一种是Claude Code Agent Team的做法: 也就是切分只读的agent，写的agent的话就是做到不重叠改代码，也就是把冲突解决前置到前面切分工作，让agent开工的时候改的那部分文件本身就不会和别的agent  
