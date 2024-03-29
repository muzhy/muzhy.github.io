+++
isCJKLanguage = true
title = "关于邮件网关的思考——Check Module的设计"
description = "关于邮件网关的思考——Check Module的设计"
keywords = ["email gateway"]
date = 2023-10-29T16:12:39+08:00
authors = ["木章永"]
tags = ["email gateway"]
categories = ["email gateway"]
aliases = [
    "/posts/thinking-about-the-email-gateway-check-module/"
]
+++

在[关于邮件网关的思考](/posts/thinking-about-the-email-gateway/)说明了下关于邮件网关的大体设计框架，但是仅仅是说了下整体的模块划分以及模块将的协作，要能够将实现出程序还需要进行模块内部的设计

<!-- 模块的内部设计不可避免的与选择实现方式产生联系，不能再单纯的从概念上出发，需要考虑整体的架构实现方式。

为了方便理解和扩展，将三个模块放在同一个程序里面运行，程序中包含三个线程，分别运行三个模块，同步调用通过函数调用实现，异步调用采用CSP（Communicating Sequential Processer）模型，使用线程安全队列作为通讯通道。异步调用使用该模型可以在需要扩展时对程序进行简单改造，将通讯通道修改为消息队列即可实现远程调用，程序的基本架构可以保持不变。 -->

本文说明了划分的三个模块中的Check Module的设计

# Check Module
Check Module所要完成的工作简单来说就是要根据所给的邮件内容来决定邮件后续需要进行操作，而判断的依据在这里称为规则。

## 检查结果
Check Modul接收到检查请求后，选择相应的规则进行匹配，若匹配中则执行该规则对应的动作。并且为了应对变化，规则需要是可配置，可扩展的。同时由于会存在多条规则，需要决定这些规则的优先级，当接收到邮件信息后，根据优先级对规则逐一进行匹配，若命中某一条规则后，若规则对应的动作是能中段后续检查的则可以立即返回，不需要再执行后续的检查。

先对应可能的动作，也就是命中规则后的结果，其实前文提到的`check_result`就表示了该值，现在需要更加细化：
```
enum check_result {
    PASS,               // 继续执行后续的检查
    REJECT,             // 拒绝接收该邮件，向SMTP客户端返回错误信息
    NEED_ASYNC,         // 需要执行异步检查获取检查结果
    QUARANTINE,         // 隔离，接收该邮件，但是暂时不投递出去
    DELIVER,            // 将邮件投递出去
    DISARD,             // 丢弃，接收该邮件，但不保存邮件原文
};
```
上面只是列出了可能的结果，可以根据需要再继续扩展。

现在Check Module的处理流程看起来是这样的

![Check Module](/images/CheckModule.png)

## 规则执行时机
目前看起来整个流程是比较简单的，但是这里存在一个问题：

在前面的设计中，Check Module是支持分阶段检查的，那是哪些规则应该在哪个阶段进行检查呢

考虑下具体点的场景

假设有一个规则A需要邮件的所有信息才能进行判断，还有另一条规则B只需要收信人的信息就可以进行检查，并且此时A的优先级高于B的优先级。这时候A,B分别应该在什么阶段进行检查？

加入我们根据规则所需的信息来决定规则应该在哪个阶段进行检查，则B规则在MAIL FROM阶段进行检查，而A规则在DATA阶段进行检查。这将导致优先级的倒置——B的优先级低于A但是B却先于A进行检查。如果让B规则的结果生效，则很可能A规则不会再被检查。这时候可以考虑只记录B的结果，而不立即生效，直到后续所有优先级高于B的规则全部检查完毕后，才执行B的结果。

那要怎么知道B规则后面还有没有优先级高于B的规则呢？显然这需要Check Module获取全部的规则以及知道其所需要的信息在什么阶段能获取到，然后在每个阶段获取能在该阶段进行检查的所有规则进行检查，在返回结果前获取后续阶段所需检查的规则的最高优先级，如果有高于已命中规则的优先级的规则时，则记录下结果及优先级，返回需要继续检查的结果，直到所有阶段检查完成或后续阶段没有更高优先级的规则后，采用之前记录的检查结果。

此时需要有个地方记录该邮件的检查结果，我们可以给每个检查的邮件一个唯一标识ID，ID应该由SMTP Module生成，作为请求参数传递给Check Module，Check Module根据ID存储过程中产生的各种信息；或者将中间信息也一起返回给SMTP Module，让其在下次调用的作为参数返回。 

个人更倾向于使用ID进行标识，将中间信息存放在Check Module，一个原因时SMTP Module显然不需要知道这些中间信息，另一个则是可以减少中间的参数传输，而且显然我们希望每个邮件都有一个唯一标识，方便后续排查问题时能根据该标识还原邮件的处理流程。

还有另一种更加简单的方案：根据规则所需的信息给一个最高优先级。如需要DATA阶段检查的规则的最高优先级不能高于在MAIL FROM阶段检查的规则的最高优先级，但是允许MAIL FROM阶段的规则的优先级比DATA阶段的最高优先级低。

通过在添加规则时做限制，将规则优先级的设置交由使用者负责，程序只负责对规则做检查。这样的设计能简化程序的实现，但是无疑会加大使用者的负担，但是却可以让使用者更好的控制优先级以及规则执行顺序。

但是目前来看将负责的地方留给程序可以更好的避免出错，所以这里按照第一种方式进行设计。

现在Check Module在接收到一个检查请求后的流程变为：

![Check Module](/images/CheckModule-2.png)

新添加的几个处理流程使用红色标识出来，现在的问题转变为要如何确定一个规则是需要对应到哪个阶段执行

要确定规则在哪个阶段执行最简单的方式是将所有规则都放到最后接收到整封邮件执行。但是如果希望规则能够尽可能早执行的话，可以通过确定最晚到什么时候可以顺利执行该规则来决定。

要能够顺利执行完一条规则，必须在执行的时候已经拥有执行该规则所需的所有信息。比如，规则A需要根据邮件内容来判断是否隔离邮件，那么显然需要在接收到完整数据后再执行检查；规则B需要获取发信人和发信IP的信息来判断，则最早可以在MAIL FROM阶段完成后执行。

将优先级低的规则提前执行然后保存结果虽然可以提前执行部分规则，但是很可能造成资源的浪费，可以采取另一种思路，先将所有的规则按照优先级排序，高优先级的规则排在前面，然后从前向后遍历，获取每个规则的最早执行时间，规则的执行时间确定为
$$ max(最早执行时间, 前面的规则的执行时间）$$
初始的前面规则的执行时间为CONNECT，这里假设执行的阶段越靠后，其所代表的数值越大。

所以，只要知道规则需要哪些信息就可以确定规则的最早执行时间，确定最早执行时间就可以确定规则执行时间。而要知道规则需要哪些信息，则需要从构成规则的结构下手

## 规则的组成结构
一条规则应该包含多个检查项，每个检查项可以看作是一个函数，输入参数是邮件一部分信息，而规则就是这些函数组合起来的表达式，表达式的结果就是规则的结果：

$$ z = f(x) \cup g(y) $$

### 规则的检查项
输入的参数决定了检查项最早可以在哪个阶段进行检查，规则中的最晚检查项决定了规则最早可以执行的时机。

函数的实现方式决定了如何处理输入的参数，通常而言，可以大致分为以下三类：
1. 文本匹配。根据邮件的部分信息进行文本匹配，比如发信人是XXX的邮件拒收，包含某些关键字或者符合了正则表达式的匹配的隔离这种针对邮件内容进行简单的文本匹配。
2. 行为统计拦截。比如某个IP或发信人在一段时间内大量发信，则后续的一段时间不接收该IP或发信人发出的邮件。
3. 调用外部服务根据结果检查。如SPF检查需要将发信IP和发信人域名通过DNS获取结果来判断；病毒检查通常需要调用第三方杀毒引擎；或者通过AI服务根据邮件内容获取邮件是否是垃圾/钓鱼邮件的概率。

函数的实现需要依赖代码实现，对于第一类函数，所需要的邮件信息可以通过参数指定，通常检查含义可以表达为：对X根据模式串Y以F方式的匹配，若从匹配中则返回结果Z。

比如对发信人（X）根据模式串`myzhy@github.com`（Y）以全匹配（F）的方式进行匹配，若匹配到则投递该邮件（Z）

这时候，检查项所需的参数需要在哪个阶段获取到也需要根据参数推断，在加载该检查项时就需要获取到所需参数中最晚能获取到的时机。在以上例子中，发信人信息可以在MAIL FROM阶段到。

对于第二类函数，所需要的邮件信息也可以通过参数指定，比如某个IP在一个小时连发信数量超过100封的邮件进行拦截，对应的IP信息可以在CONNECT阶段获取到。

第三类函数所需的参数其实时固定的，比如SPF检查需要发信人IP和发信域名，病毒检查需要在DATA阶段后的邮件内容，这些参数通常在实现时就已经确定了，即该检查项本身就确定了其能最早执行的时机。

仅对于怎么获取检查项而言，其实可以分为两类：
1. 检查项本身就决定了该检查项的最早检查时机（第三类）
2. 检查项本身并不能决定其最早检查时机，还需要根据其参数才能明确该检查项的最早检查时机（第一，二类）

### 检查项简单的运算

规则除了检查项外，还需要包含这些检查项的组合关系。

比如规则A有两个检查项a, b， 那么规则的结果是a和b都检查通过才返回通过还是a和b之间一个检查通过就返回通过？

对于一个检查项，通过该检查项的邮件可以看作是一个集合，检查项的结果之间的运算可以看作是集合运算，按照集合运算的规则来定义规则能支持的运算即可，可以根据实际的业务常见和实现复杂度做取舍，在这里无需赘述。

回到之前的规则的表述: $$ z = f(x) \cup g(y) $$

### 集合运算结果到检查结果的映射

检查项和运算规则定义了公式的右边，还需要确定左边结果的定义。

对于一封邮件而言，集合运算的结果只有两种——在集合中、不在集合中。但是之前定义的规则的结果`check_result`有多个枚举值，也就是说我们还需要在规则中增加一个映射函数，将布尔值映射到`check_result`中，只有两个值的输入集需要映射到多个值的输出集，显然靠一个映射函数是无法做到的。

此时应该提供多个映射函数，由规则的创建者决定要采用哪个映射函数将集合运算的结果映射到检查结果——在集合中时应该对应`check_result`中的哪个枚举值，不在集合中又应该对应哪个枚举值。

或者换另一种表述，提供一个映射函数，输出参数除了集合运算结果外，还需要又另外参数来决定在集合中对应哪个枚举、不在集合中对应哪个枚举。

$$ result = mapping( f(x) \cup g(y) , z) $$


### 规则间的优先级
规则本身的定义基本确定了，但是有多条规则时，还需要对每一条规则赋予一个优先级来确定最终的结果是以那一条规则的结果为准

优先级的确定比较简单，给一个整数，数值越大表示优先级越高即可。

但是这里存在一个业务上的取舍：是要求在创建规则的时候要求所有规则的优先级都唯一，还是允许优先级重复然后依据规则的创建/修改时间来确定相同优先级的规则的优先级。

从设计的角度要求所有规则的优先级都唯一会更好一点，但是可能会给规则创建者带来负担（可以通过适当的提醒来降低）；允许重复的优先级在创建规则时或许会很爽，但是实际执行时由于需要根据时间来进一步确定优先级，可能会出现不符合规则创建者希望的结果。

个人认为要求所有规则的优先级唯一是更好的选择，因为规则创建者可以看作是专业人士，专业人士有责任维护这些规则之间的优先级。

# 总结
Check Module主要有规则构成，规则包含检查项、检查项间的运算、运算结果到检查结果的映射函数以及规则本身的优先级。

Check Module启动时获取所有的规则，根据规则中的检查项及参数获取到每个阶段能执行的规则，在对应的结果按照优先级执行能在该阶段执行的规则，获取到规则的结果后，判断后续是否有更高优先级的规则需要执行来决定返回的检查结果。