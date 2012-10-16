---
title: '触发器实现分析'
date: '2012-10-16'
categories: mmo
tags: [trigger]
analytics: false
comments: false
---

# 触发器的目前实现

这里只提一些关键问题的实现，主要是围绕触发器的触发以及触发器对象的生命周期管理问题。诸如其他类似触发器触发检测的实现，因为较为独立，无需改进。

## 触发器列表管理

触发器相关的管理列表分为两个：

* hash_map<trigger_guid, Trigger*>
* hash_map<owner_guid, TriggerList*>

其中，TriggerList是一个更细分的列表管理。它将触发器按类型存放在不同的列表，其内部结构大概为：

    hash_map<trigger_guid, Trigger*> *triggers[eTriggerSize]

对触发器而言之所以使用了两个表来管理，一部分原因是脚本需要。在脚本中，往往需要单独通过触发器ID来获取该触发器的信息，例如获取其参数、删除触发器。另一方面，在程序中当某个游戏对象发生了某个事件时，又往往需要通过`owner guid`来检查附加在该对象身上的触发器。

TriggerList的存在仅仅是为了加快在检查某个事件发生时会触发哪些触发器时的速度，因为避免了遍历其他类型触发器的冗余过程。

## 触发器触发

### 触发器触发实例

在描述触发器触发的细则前，需要对触发器触发实例做了解。

触发器触发实例的出现，是为了实现触发器下一游戏帧而设计。每一个触发器都有两个参数表：`in-arg`和`out-arg`。`in-arg`主要是在注册触发器时，由脚本填入的参数表，用于描述一个触发器在哪种情况下会被触发；`out-arg`主要是当触发器触发时，需要保存下来的触发环境，例如当属性改变触发器触发时，`out-arg`就需要填入改变量，这些值用于脚本内的逻辑。

一个触发器对象在不同的时间点触发时，其`out-arg`都是不一样的。每一次该触发器触发时，要将这一次触发环境保存下来，以放到下一帧执行，就需要将`out-arg`保存起来。因此，触发器的触发实例结构就为：

    struct TriggerInst {
        Trigger *trigger; 
        ParamTable *outarg;
    };

一个触发器在某一帧可能会被多次触发，因此需要一个列表来保存这些触发实例：

    map<trigger-guid, list<TriggerInst*>>

### 触发过程

触发器的触发过程大致如下：

    -> 发生某个事件 `Trigger::TraverseOwner`
        -> 通过`owner guid`查找到`TriggerList`
            -> 通过触发类型得到小列表 `map<CGUID, Trigger*>`
                -> 遍历，检查满足触发条件的触发器
                    -> 缓存起来准备下一帧执行

在游戏主循环中，每一帧里会把缓存起来的触发器依次执行：

    -> 复制缓存列表
        -> 遍历该列表，依次执行
            -> 清除列表  

### 生命周期管理

目前针对触发器的生命周期管理，是通过往触发器身上添加一个删除标志，以及一个删除列表实现的。

删除一个触发器的情况是非常多的（以下会描述），当删除一个触发器时，会将该触发器关联的一些资源释放掉，设删除标志为真。但这里并不真正删除触发器对象，即不回收内存，而是将其加入一个删除队列。除此之外，这里也会立即将这个触发器从各个管理列表中移除。

# 触发器主要问题

以下罗列一些能想到的、曾经遇到过的一些触发器问题，以及一些目前的解决方案。

## 删除问题

触发器的删除（这里特指暂放于删除列表）有很多情况，这是导致问题的关键所在，包括但不限于以下情况：

### 一般情况

一般情况下，当触发器执行(dispatch)完一次后，会检测自己是否达到了指定的触发次数而删除自身。在编写这样的代码时，很容易做到触发器删除后不再编写依赖该触发器的代码，不会出现问题。
 
### 脚本删除

脚本可以随意删除某个触发器，以达到中断该触发器的目的。如果在一般的脚本中删除触发器，不会有问题。但是当在触发器执行的脚本中删除这个触发器，则会出现问题。因为在执行完这个脚本后，代码上还会使用这个已被删除的触发器指针做逻辑。例如判断该触发器执行次数等。

即使在目前的版本中，通过往触发器上添加一个删除标记，用于标示其是否合法，虽然可简单地避免野指针问题，但也容易出现错误的逻辑。除非在很多地方加上触发器合法性判定的代码。

### 宿主删除

每个触发器都有关联的游戏对象（玩家、怪物），简称宿主。当某个宿主被删除时，需要删除该宿主对应的所有触发器。一般情况下删除没有问题，但是，如果是在触发器执行的脚本中所做逻辑导致的宿主删除，则会出现问题。由于宿主删除而导致删除的触发器中，有可能包含当前正在被执行的触发器，于是出现非法触发器。

### 定时器

触发器有定时器类型，用于向脚本提供定时器功能。定时器类触发器在触发和生命周期管理问题上与普通触发器存在很大差别。定时器分为三种类型：延时（只触发一次）、周期（触发多次）、生命时长控制。其中生命时长控制指的是控制一个触发器存在的时间。

无论以上哪种定时器，都会涉及到在定时器执行流程中删除触发器。但删除触发器时，也会涉及到注销触发器注册的定时器。在加入触发器删除列表之前，定时器类触发器在管理上也发生过很多混乱的BUG。在目前的版本中，统一将触发器的删除放在`OnTimeOut`中，而`OnTimerCancel`几乎不做任何事（以前的版本中不是这样做的）。经过实际一段时间的使用，暂没有得到错误反馈。

## 其他细节问题

基于上一节描述的几种删除情况，触发器在实际应用场合下还暴露出了以下问题。

### 执行未完成的触发实例

一个触发器在某一帧里可能会产生多个触发实例。在这些触发实例被执行前，这个触发器就可能被删除，这个时候就需要依次将这些触发实例全部执行。

这种情况的发生，除了在脚本中主动删除某个触发器外，也有可能是由于某个对象被删除导致。因为对象被删除前需要将其附加的所有触发器也删除。

### 触发实例执行时丢失触发信息

触发实例在执行的时候，当初该触发器触发时的一些数据在这个时候可能已经发生改变，在脚本中取这些信息做逻辑时，则会因为数据和当初发生时不一样（甚至出现相反的情况）而产生逻辑错误。这个问题的解决是通过在触发时将需要的数据保存下来，在执行时将这些数据传入脚本。

### 触发器间相互删除

在执行触发实例时，有类似以下代码：

    for (TriggerTable::iterator it = triggers.begin(); it != triggers.end(); ++it) {
        TriggerInsts *insts = it->second; // 某个触发器的触发实例列表
        for (TriggerInsts::Triggers::iterator iit = insts->triggers.begin();
                insts->valid && iit != insts->triggers.end(); ++iit) {
            RefTrigger ref = *iit; // 某个触发实例
            bool ret = Trigger::DispatchTrigger(ref.trigger, ref.outarg);
            if (!ret || ref.trigger->IsEndTrigger()) {
                Trigger::Del(ref.trigger); // 删除该触发器
                break; // 后面的触发实例没必要执行
            }
        }
    }

以上代码存在问题。

首先，某个触发器可能在`Trigger::DispatchTrigger`中删除自身，这导致后面未执行的触发实例都变为无效，修改为：

    bool ret = Trigger::DispatchTrigger(ref.trigger, ref.outarg);
    if (ref.trigger->Destructed()) break; // trigger->Destructed()标示触发器是否已经无效

另一方面，在某一个触发实例执行的过程中，它可能会删除另一个触发器，那么这个被删除的触发器实例对应的所有触发器实例都不需要执行。进一步修改为：

    RefTrigger ref = *iit; // 某个触发实例
    if (ref.trigger->Destructed()) break; // 被其他触发器删除
    bool ret = Trigger::DispatchTrigger(ref.trigger, ref.outarg);
    if (ref.trigger->Destructed()) break; // 被自身删除

### 校验触发实例有效性

在执行触发实例的时候，也会产生新的触发器实例，但这些实例都会被放在下一帧执行。这些实例也极有可能在产生后，还未到达下一帧就被删除。意思是，在执行本帧所有的触发实例时，可能会先产生了一个触发实例，但紧接着又将其删除。

解决这个问题目前的简单方法是在执行完所有的触发实例后，立即对新加入的触发实例做检查：

    for (TriggerTable::iterator it = m_triggers.begin(); it != m_triggers.end(); ) {
        TriggerInsts *insts = it->second;
        if (insts->triggers.size() > 0 && Trigger::IsInDel(insts->triggers.begin()->trigger)) {
            DeleteInsts(insts);
            it = m_triggers.erase(it);
        } else {
            ++ it;
        }
    }

*似乎可以在执行触发实例的那个循环中加入一些判断，而回避这里的遍历。*

