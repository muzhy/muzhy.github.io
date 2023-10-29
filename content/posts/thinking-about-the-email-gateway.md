+++
draft = true
isCJKLanguage = true
title = "Thinking About the Email Gateway"
description = ""
keywords = []
date = 2023-10-21T21:16:58+08:00
authors = ["木章永"]
tags = []
categories = []
+++

# 邮件网关
随着IM的发展，电子邮件作为通讯方式的重要性有所降低，但是仍然是现代社会不可或缺的一部分，特别是对于较为正式的场合都需要通过电子邮件来进行沟通。
尤其是在国外，由于电子邮件普及的时间更久，使用的范围更广，当需要与外国人交流时，邮件仍然时一个不可或缺的手段。

但是伴随着邮件而来的，还有大量的垃圾邮件存在，邮件网关的一个重要的功能就是部署在邮箱系统之前，通过一定的规则检查邮件，对于检查出来的垃圾邮件进行隔离，不投递到邮箱系统。相当于邮箱系统的一个保安。

# 需要邮件网关的原因
为什么不在现有的邮箱系统内部添加邮件拦截的功能呢？个人认为有以下几个原因：
1. 已经部署并运行了一段时间的邮箱系统，除非原本就具有邮件拦截功能外，若需要增加额外的拦截功能可能需要大范围的改动，且各个公司部署的邮箱系统的服务可能各不相同，相当于需要根据每个现有的邮箱系统进行改造
2. 保持功能的单一性。邮箱系统应该只负责接收邮件并将邮件发送到对应邮箱账号的收件箱中，增加邮件检测拦截功能超出了原本的设计功能。额外增加一个邮件网关只负责检查和拦截邮件，对于通过拦截的邮件直接发送邮箱系统即可，不需要处理繁琐的用户数据
3. 灵活快速变更。由于垃圾邮件层出不穷，且经常会变换花样，经常需要根据最新的情报更新安全策略。邮箱网关可以专注于安全方面的问题，设计为可灵活配置更新安全策略

通过在原有的邮箱系统前增加一个代理层负责检测拦截邮件，不需要改动到原有的系统，且可以让两个系统各司其职

# 邮件网关的整体架构
前面已经说了，网关是要部署在邮箱系统前面，因此需要对外提供SMTP服务来接收邮件；接收到邮件之后，需要对邮件进行检查，判断是否应该拦截邮件；对于通过检查的邮件，通过SMTP将邮件投递到邮箱系统
![邮件网关](/images/email-gateway.png)

据此，将网关分为三个模块（Model）：
1. SMTP模块（SMTP Module）：用于接收外部发送来的邮件
2. 检查模块（Check Module）：用于检查邮件是否需要被拦截。
3. 发信模块（Deliver Module）：用于将通过检查的邮件投递到邮箱系统

此处所说的模块是从功能概念上进行的划分，不同的实现可以选择不同的方式。比如只是用于流量很小的场景时，这三个模块甚至可以采用最简单的用三个函数来实现，通过函数调用来实现模块间的通讯；对于需要应对可能的大流量场景时，可能需要将三个模块实现为不同的服务以便部署到不同的机器上，再通过消息队列或其他远程调用方式进行通讯。

在明确了功能划分，将整体分为三个模块后，需要确定这三个模块是如何配合工作的。
最简单的情况下，SMTP Module接收完邮件后，将邮件发送给Check Module进行检查就完成了；类似的，邮件通过Check Module检查后，将通过检查的邮件发送给Deliver Module进行发信就可以了。

但实际情况总是要复杂很多。首先，SMTP协议本身是分为多个阶段的——CONNECT、EHLO/HELO、MAIL FROM、RCPT TO、DATA、DATA DOT、QUIT——SMTP Module要能在每个阶段都有能力调用Check Module进行检查，以便能够在命中了某些规则后直接拒绝（比如，想要拒绝某个IP的发信，可以在CONNECT阶段就拒绝）；其次，Check Module除了要将检查通过的邮件发送给Deliver Module外，还需要考虑如何处理那些没有通过检查的邮件，这些邮件可能是前面说的直接拒绝掉了不需要保存，也可能是希望暂时将邮件接收下来，希望等待后续管理员判断是否要投递再进行投递；另外Check Module可能需要在接收到整个邮件后进行一些耗时的检查操作，如病毒检查或使用AI判断邮件是否是垃圾邮件；

## SMTP Module 和 Check Module的协同工作
SMPT Module需要调用Check Module进行检查的地方有以下地方：
1. CONNECT 
2. MAIL FROM
3. RCPT TO
4. DATA FINISHED

简单来说，根据SMTP协议的各个阶段划分，每个阶段完成后都要能将当前的接收到邮件信息发送给Check Module进行检查，SMTP Module在根据检查的结果执行后续的动作。

因此，SMTP Module和Check Module之间的调用似乎可以采用同步调用的方式，所需要的信息包括：邮件信息、当前检查所处的阶段。

这种同步的调用在DATA FINISHED之前的都是可行的，但是在完整接收到整个邮件信息后，可能需要执行耗时的检查操作，而此时SMTP Module希望能尽快断开连接以处理后续的请求，对方的邮件服务也可能希望在发送完全部数据后尽快断开连接。此时SMTP Module需要提供异步检查的功能，让SMTP Module可以先返回250，直到Check Module检查完成后再根据结果进行操作。

但是，也有些情况，希望在接收到邮件数据后，经过一些检查后（如对于邮件结构不符合规范的可以直接拒绝）确定没有问题后再由SMTP Module返回成功，而不是在检查之前就返回成功。

因此，SMTP Module需要支持同步和异步的方式，但是哪些检查要采用同步的方式，哪些检查要采用异步的方式呢？

将采用同步还是异步方式的调用方式的决定权交给SMTP Module似乎不太合理，因为做出决定的一大重要信息是某些检查操作是否耗时，而操作是否耗时只有负责执行Check Module才知道，同时，SMTP Module不需要也不应该知道需要进行什么检查，这些信息Check Module知道就好了，SMTP Module只需要知道什么时候需要进行检查并根据检查结果进行处理。

所以，可以采用以下方式：
1. SMTP Module 在需要进行检查的位置先通过同步的方式调用Check Module
2. Check Module 在接收到检查请求后，先执行那些需要同步的检查操作，如果这些同步检查的操作处理完成后该阶段还需要执行异步检查操作的话，返回一个结果表示需要进行异步检查
3. SMTP Module 接收到检查结果后，如果发现需要进行异步检查再通过异步调用方式调用Check Module，处理邮件信息和当前阶段外，还需要有回调函数（对于远程调用可以时回调的URI）
4. Check Module 在接收到异步检查请求后，启动异步检查并返回调用成功。直到该阶段所有的异步检查处理完成后，将结果通过回调函数返回给SMTP Module
5. SMTP Module 接收到异步检查的结果后，根据结果执行操作。

虽然从业务上看似乎只有在DATA FINISHED后才可能需要异步检查，从设计上支持每个阶段都能根据结果来决定是否需要进行异步检查会更加灵活，且可以保持接口统一，每个阶段的检查都可以使用相同的接口：
```
enum check_result {
    PASS, 
    REJECT, 
    NEED_ASYNC,
    ...
};

check_module::check(mail_info, current_state) -> check_result ;
check_module::async_check(mail_info, current_state, callback) -> bool;

smtp_module::check_callback(check_result)

smtp_module::on_xxx(){
    result = check_module::check(mail_info, curr_state)
    if result == NEED_ASYNC then
        check_module::async_check(mail_info, current_state, check_callback);
        return 
    end
}
```

检查结果（check_result）可以时间简单的枚举类型来表示，在其中使用一个值来表示需要进行异步检查

Check Module提供两个接口，一个同步调用接口，一个异步调用接口，所有阶段的检查都通过这两个接口来完成。异步调用接口只需要返回发起异步调用是否成功，异步调用的检查结果通过call_back返回

SMTP Module在需要检查的地方，都先调用check_module::check进行同步检查，拿到结果后判断是否需要异步检查，需要异步检查再通过check_module::async_check进行异步检查

SMTP Module和Check Module之间的通讯所需要的接口就是以上几个，根据不同的实现可能需要在进行适当的调整，上述伪代码是基于在单个应用内通过回调函数的，如果是设计到远程调用，需要考虑不同的回调方式以及mail_info的传递方式（mail_info 表示当前接收到的邮件信息，可能很大）

## SMTP Module 和 Deliver Module的协同
SMTP Module接收完邮件并检查通过后，需要将邮件通过Deliver Module发送到位于网关后端的邮箱系统。同样的，调用方式也需要考虑选择同步还是异步。

同步的好处一方面是实现简单，另一方面是，当邮件被邮箱系统拒收时，网关可以直接返回给连接到网关的SMTP客户端，比如当收信人不存在时，邮箱系统可能会直接拒收，但是由于网关不知道邮箱系统有什么用户只能接收该邮件。

但是同步也存在很多问题，首先，由于检查邮件时需要异步，所以同步邮箱系统返回的错误信息给SMTP客户端实际上是无法实现的；其次，当邮箱检查后需要进行隔离（先由网关收下邮件，但暂时不投递，等待后续通过其他方式才投递）时，同步也是无法实现的；

看起来异步投递的选择是比较合理的，但是异步的时候存在其他问题：当邮件投递到网关成功，但是投递到邮箱系统失败的时候应该如何通知发信人？

当然并不是所有投递失败的邮件都需要同时发信人，但是总是存在需要通知发信人的场景。此时通知发信人也需要是异步的方式，一种常见的方法是，使用固定的发信人，想投递失败的邮件的发信人发送一封邮件通知该邮件投递失败。

因此，SMTP Module在需要投递邮件时，将邮件信息发送给Deliver Module，由Deliver Module负责发信，SMTP Module只要将邮件发送到Deliver Module就可以认为工作已经结束了，剩下的操作由Deliver Module负责。

Deliver Module接收到投递邮件的请求后，创建一个异步的投递任务后，立即返回成功。异步的投递任务负责使用SMTP协议将邮件投递到后方的邮箱系统，当SMTP返回失败并且根据策略决定不再尝试投递后，生成一个退信的邮件，通过公网投递到发信人的邮箱。

Deliver Module 并不需要对邮件做检查，于Check Module没有交互。

# 模块内部设计
模块的内部设计不可避免的与选择实现方式产生联系，不能再单纯的从概念上出发，需要考虑整体的架构实现方式。

为了方便理解和扩展，将三个模块放在同一个程序里面运行，程序中包含三个线程，分别运行三个模块，同步调用通过函数调用实现，异步调用采用CSP（Communicating Sequential Processer）模型，使用线程安全队列作为通讯通道。异步调用使用该模型可以在需要扩展时对程序进行简单改造，将通讯通道修改为消息队列即可实现远程调用，程序的基本架构可以保持不变。

## Check Module
Check Module所要完成的工作简单来说就是要根据所给的邮件内容来决定邮件后续需要进行操作，而判断的依据在这里称为规则。

### 检查结果
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

### 规则执行时机
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

所以，只要知道规则需要哪些信息就可以确定规则的最早执行时间。而要知道规则需要哪些信息，则需要从构成规则的结构下手

### 规则的组成结构
一条规则应该包含多个检查项，每个检查项可以看作是一个函数，输入参数是邮件一部分信息，而规则就是这些函数组合起来的表达式，表达式的结果就是规则的结果：

$$ z = f(x) \cup g(y) $$

#### 规则的检查项
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

#### 检查项简单的运算

规则除了检查项外，还需要包含这些检查项的组合关系。

比如规则A有两个检查项a, b， 那么规则的结果是a和b都检查通过才返回通过还是a和b之间一个检查通过就返回通过？

对于一个检查项，通过该检查项的邮件可以看作是一个集合，检查项的结果之间的运算可以看作是集合运算，按照集合运算的规则来定义规则能支持的运算即可，可以根据实际的业务常见和实现复杂度做取舍，在这里无需赘述。

回到之前的规则的表述: $$ z = f(x) \cup g(y) $$

#### 集合运算结果到检查结果的映射

检查项和运算规则定义了公式的右边，还需要确定左边结果的定义。

对于一封邮件而言，集合运算的结果只有两种——在集合中、不在集合中。但是之前定义的规则的结果`check_result`有多个枚举值，也就是说我们还需要在规则中增加一个映射函数，将布尔值映射到`check_result`中，只有两个值的输入集需要映射到多个值的输出集，显然靠一个映射函数是无法做到的。

此时应该提供多个映射函数，由规则的创建者决定要采用哪个映射函数将集合运算的结果映射到检查结果——在集合中时应该对应`check_result`中的哪个枚举值，不在集合中又应该对应哪个枚举值。

或者换另一种表述，提供一个映射函数，输出参数除了集合运算结果外，还需要又另外参数来决定在集合中对应哪个枚举、不在集合中对应哪个枚举。

$$ result = mapping( f(x) \cup g(y) , z) $$


#### 规则间的优先级
规则本身的定义基本确定了，但是有多条规则时，还需要对每一条规则赋予一个优先级来确定最终的结果是以那一条规则的结果为准

优先级的确定比较简单，给一个整数，数值越大表示优先级越高即可。

但是这里存在一个业务上的取舍：是要求在创建规则的时候要求所有规则的优先级都唯一，还是允许优先级重复然后依据规则的创建/修改时间来确定相同优先级的规则的优先级。

从设计的角度要求所有规则的优先级都唯一会更好一点，但是可能会给规则创建者带来负担（可以通过适当的提醒来降低）；允许重复的优先级在创建规则时或许会很爽，但是实际执行时由于需要根据时间来进一步确定优先级，可能会出现不符合规则创建者希望的结果。

个人认为要求所有规则的优先级唯一是更好的选择，因为规则创建者可以看作是专业人士，专业人士有责任维护这些规则之间的优先级。
