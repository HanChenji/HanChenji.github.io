# GEM5 前端取指模块

* ```fetch```:

  * 该函数首先根据SMT算法来决定取指的线程，如果是单线程模式而且没有取到合法的TID的话就分析堵塞的原因并更新性能计数器

  * 然后是做取指的地址运算准备

  * 然后是检查```fetchStatus```的状态：如果是```IcacheAccessComplete```状态，则将状态改为```Running```，然后进行后续的译码操作；如果是```Running```状态，则准备发送访问Icache请求；否则的话，说明该tid的取指被堵塞，退出```fetch```函数。

  * 上一步中所说的如果是```Running```状态，则准备发送访问Icache请求，具体来说干了这些事情：首先检查是否需要发送ICache访问请求，这里的注释写的清楚。如果能发送访存请求的话，就调用```fetchCacheLine```函数，然后退出```fetch```函数；否则的话检查该指令是否被标记上了中断并且不能忽略中断去取指，这时候退出```fetch```函数

    * > If buffer is no longer valid or fetchAddr has moved to point to the next cache block, AND we have no remaining ucode from a macro-op, then start fetch from icache.

  * 只有当```fetchStatus```处于```IcacheAccessComplete```状态，也即已经取到了icache的数据的情况下，函数才会继续执行，接下来主要是干一些注册```fetchQueue```的事情

  * ```++fetchStats.cycles```__看来__这个变量记录的是有效的取指周期数

  * 接下来我们来详细描述是怎么从取回的```fetchBuffer```里面解析出指令来并且放入```fetchQueue```里面的:由于兼容ISA的缘故，指令的来源有三：rom, Macroop, deocder。在最外层的while循环的一开始，首先检查是否需要额外的访存请求，并生成```fetchBufferBlockPC```，如果```needMeme```为真，则检查目前的```fetchBuffer```里面的数据是否还有效，以及是否还够用(```blockOffset>=numInsts```)，如果上面两个有一个不满足，则退出while循环；否则的话，说明还有数据在```cacheInsts```里面没有处理，那么```decoder```先后调用```moreBytes```, ```needMoreBytes```函数来取数据

  * 然后我们进入了外层while循环的内层while循环：首先如果指令不是cueMacroop或inRom的话，那么检查```decoder[tid]->instReady()```，如果有效的话则构建```staticInst```，否则的话就退出该while回到上一层while中去，这时候的状态是!curMacroop && !inRom && !instReady，则对应了needMem=1，这时候如果发现```fetchBuffer```还没有用完的话就更新```blockOffset```, ```pcOffset```,否则就退出：这也就是上一段在说的事情

  * 我们注意到，```curMaroop```是在decoder构建了```statisInst```后发现是```isMacroop```属性后赋值的：```curMacroop = staticInst```，也就是说对于指令来自于rom, Macroop的，本质上也是从decoder取来的：这其实就是干了CISC指令集把指令拆成若干微指令的事情。

  * 言归正传，在内层循环开始，如果指令是在curMacroop或inRom的话，则分别调用```fetchRomMicroop```或者是```fetchMicroop```函数来构造```staticInst```。

  * 内层循环执行到此，构造了有效的```staticInst```之后便调用```buildInst```赋值给```instruction```，注意到在上述函数中将此```staticInst```push到了fetchQueue中。接下来就是分支预测器更新```nextPC```。这里注意到```newMacro```变量标志了当前的staticInst是不是‘指令包’的最后一条：其赋值来源有二：一是```staticInst->isLastMicroop```，二是预测器预测了跳转；如果```newMacro```为真，则更新地址相关的变量以及将```curMacroop```置为无效。最后，内层循环检查该指令是不是```isQuiesce```来将fetchStatus更新为```QuiescePending```

  * 内层循环的判断条件为curMacroop不为空（从decoder取出的数据还没有用完），或者是decoder里面还有ready的数据（这样子的话curMacroop就能重新被赋有效值），以及取指带宽未满或fetchQueue为堵塞

  * 于是又来到了外层循环，重新从decoder里面取出有效数据，然后进入内层循环：于是得出结论，对于CISC指令集来说，外层循环是取出inst的动作，内层循环是将inst转化为macroop的操作

  * 退出上述两个循环后，记录下```curMacroop```, ```pcOffset```:如果循环退出原因是带宽不够或fetchQueue满的话，fetchBuffer里面仍有有效数据，```curMacroop```也是有效的，等待下一次取指使用：这里回到了一开始fetchStatus是Running的时候检查是否需要重新发送```fetchCacheLine```的逻辑那里。呜呼，终于逻辑闭环了。

  * ```fetch```函数最后更新```pc[tid```并且标记是否继续发出访存请求的```issuePipelineIfetch```变量

* ```buildInst```:

  * 

* ```checkSignalsAndUpdate``` :
  * 该函数检查其它流水级送来的信号来决定fetchStatus
  * 该函数首先检查译码阶段的反馈信号来决定```stalls[tid].decode```，也就是fetch是不是要因为decode阶段的堵塞而堵塞；然后检查commit阶段发来的信号，如果commit阶段发来了squash的信号，则fetch阶段也要进去sqush状态，调用```squash```函数来清空流水级，以及清空分支预测器中的信息，然后退出```checkSignalsAndUpdate```函数；如果commit阶段没有squash的话，就正常更新分支预测器
  * 检查完commit阶段的信息后，如果没有退出```checkSignalsAndUpdate```函数，继续检查来自decode阶段是否发来squash信号，如果是的话，同样的，要更新分支预测器，如果fetch阶段本来不处于Squashing状态的话，则调用```squashFromDecode```函数，然后退出```checkSignalsAndUpdate```函数
  * 如果fetch阶段不会被commit, decode发来的信号而squash掉的话，则检查会不会被堵塞，调用```checkStall```函数
  * 如果上述情况都没有发生，也即没有被squash，也没有处于stall状态，而本身处于```Blocked```或```Squashing```状态，说明fetchStatus在本拍可以被转化为```Running```状态
  * 如果上述都没有发生，则fetchStatus不会发生改变，比如一直在等待tlb，等待icache返还等等，则```return false;```
  
* ```squash``` 
  * 该函数类似于```squashFromDecode```函数，首先是调用```doSquash```函数将fetch级清空，PC重置；然后是调用```removeInstsNotInROB```，将处理器中不在ROB中的指令清空
  * __问题__：该函数是在检查commit阶段发来了squash信号之后调用的，那岂不是也要将ROB中的指令清空掉？不在ROB中的指令岂不是比```squashFromDecode```函数清空的多了还没有rename的指令？
  
* ```squashFromDecode``` 
  * 该函数主要干两件事情，一是调用```doSquash```函数，squash掉fetch阶段并重置pc，重置的pc来自于decode级传来的信号；二是调用```removeInstsUntil```函数，清空fetch级到decode级的所有指令
  
* ```doSquash```：
  * 重置```pc[tid]=newPC```，重置```macroop```，将```decoder[tid]```清空
  * 将发送出去的访存请求清空：根据```fetchStatus[tid]```是否为IcacehWaitResponse或ItlbWait来将memReq置为NULL，将因MSHR已满而无法发送ICache访问而建立起的retryPkt清空
  * 修改```fetchStatus[tid]```的状态为```Squashing```，清空```fetchQueue[tid]```
  * __尚不清楚__```delayedCommit```的实际意义
  
* ```checkStall``` 
  * 该函数检查```stalls[tid].drain```来判断fetch流水级是否需要停顿，该停顿有意思的是只来自于commit阶段发来的是否要排空流水线
  * ```stalls[tid].drain```变量的设置比较有趣，在此追踪如下：1.该变量在函数```drainStall```中设置为true,2.函数```drainStall```由commit阶段调用```commitDrained```函数中调用之
  * 该变量以及该函数__应该__是用在处理中断的时候，被标记上中断的指令在commit的时候，清空流水线用

* ```updateFetchStatus()``` :
  * 如果fetchstatus的状态要发生改变，那么有两种情况，其一：本来是处于Running, Squashing, ICacheAccessComplete状态，那么这时候要将```_status```设置为Active；其二：如果不是上述情况的话，则将```_status``` 将Active设置为Inactivate。注意到，这个```_status```指的是fetch模块的状态，不是每一个线程的状态：只要有一个线程active，那么```_status```就是active
  
* ```pipelineIcacheAccesses``` :
  * 发送icache的访问请求，由```issuePipelinedIfetch``` 拉高的线程发起
  * 这个函数首先检查要取的PC是不是```isRomMicroPC``` ，然后根据pc, offset以及对齐方式生成访问cache的地址，同时，在向Icache发送请求前也会检查数据是否已经放在了fetchBuffer里面，向Icache发送请求调用了```fetchCacheLine``` 这个函数
  
* ```fetchCacheLine``` :
  * 这个函数真正的向gem5的访存系统发出访存请求
  * 首先检查cache的状态：当cache处于堵塞状态，或者有中断正在处理而且```delayedCommit``` 为0（该变量决定了能够忽略有中断正在处理而前端能否继续取指）时退出此函数，放弃取指
  * 建立起访存请求数据结构```RequestPtr mem_req``` ，指定访存的地址信息（提前处理好了对齐），size信息，以及一些id信息等。该访存请求的tlb处理部分为：注册```mem_req``` 中的translation信息，同时将```fetchStatus[tid]``` 置为```ItlbWait``` 。
  * 这个函数应该得向前面提到的```fetchBuffer``` 注册，我猜应该是先放在这个buffer中，然后去处理tlb，然后再回来
  * __问题一__：```mem_req```会不会以及如何放在fetchBuffer中？
    * ```fetchBufferPC``` 会在```finishTranslation``` 函数中赋值为```mem_req->getVaddr()``` ，在tlb翻译成功的情况下
    * 但是后来注意到，```fetchBufferValid```这个变量是在```processCacheCompletion```函数中拉高，而上述函数是在```IcachePort::recvTimingResp```函数中调用的：到此变量```fetchBuffer```相关的变量就清楚了，存储发往icache的访存请求，当完成TLB地址翻译，在```finishTranslation```函数中向ICache正式发送访存请求时置为```false```，当访存请求发回且成功时，置为```true```
  * __问题二__：处理完tlb之后，如何继续这个访问Icache请求？
    * 同样是在问题一中的```finishTranslation```函数中，翻译成功之后利用```mem_req```构造起```data_pkt```，然后调用函数```icachePort.sendingTimingReq```来真正的发起对于icache的访问
  * 该函数真正处理访存请求是在```translateTiming``` 函数中，这是一个ISA相关的函数
  
* ```processCacheCompletion```函数：
  * 
  
* ```finishTranslation``` ：
  * 首先是梳理一下这个函数的调用链：1.```translateTiming``` ，这是一个ISA相关的函数，编译器会链到比如与RISCV对应的函数，进行TLB翻译，翻译完成后调用 2.```Translation::finish()``` 函数，这是一个虚函数，实际调用的是O3模型中的```FetchTranslation::finish()``` 函数，然后这个函数中调用了3.```finishTranslation``` 函数
  * 该函数的主要功能是在完成了TLB翻译之后，继续向Icache发送访存请求：1.首先尝试唤醒CPU，然后检查fetchStatus，如果其不处于ItlbWait状态或者mem_req与最新一次```fetchCacheLine``` 函数中注册的```memReq[tid]``` 不一致，说明此tlb完成的访存请求已经被取消掉了，那么更新```tlbSquashes```计数器，然后退出该函数；2.根据变量```fault```检查tlb翻译是否成功，然后分别处理。
  * 如果地址翻译成功，则试图发送icache访问请求：首先检查地址是否是一个合法的物理地址，如果不是的话，说明执行到这里的指令会触发例外？我们需要等待commit阶段squash掉一些东西，然后将fetch阶段重回正轨？接下来构建起用来发送访存请求的数据结构，同时更新上个函数中提到的```fetchBufferPC[tid]```中，然后调用```icachePort.sendTimingReq```函数，如果该函数返回```NULL```说明MSHR已满，无法再接受访存请求 （gem5中的cache是经典的non-blocking cache），此刻更新fetchstatus为IcacheWaitRetry状态，然后构建起retryPkt等待之后发射，cacehBlocked置起；如果访存请求被成功发送，则进入```IcacheWaitResponse```状态。
  * 如果地址翻译失败，则会触发tlb refill例外，具体操作如下：首先检查取指带宽是否已满，这时候暂时不处理，留到下一拍再处理，将该访存请求放到```finishTranslationEvent```这个结构体中，然后调用```schedule```函数将其放到下一拍中；如果该地址翻译失败当拍就能处理的话，首先将```memReq[tid]```清空，然后挂起例外，用一条```noop```指令代替当前指令，让其流水到commit阶段以便触发例外来处理tlb refill，设置该```noop```指令的下一条指令为该原指令，并且置fetchStatus为TrapPending
  * 该函数最后调用```updateFetchStatus()```来修改fetch状态
  
* 有了以上的梳理之后我们便可以总结```tick```函数：
  * tick函数是fetch流水级的main函数，干的事情就是取指
  * 首先，tick函数为每个线程检查后续流水级反馈过来的信号，决定是否停顿以及决定fetchStatus，然后调用```fetch```函数为每个线程取指，之后更新一下fetchStatus，然后再发送一次访问ICache的请求，这实际上是模拟了两个流水级：一个流水级处理ICache返回的数据，一个流水级负责发送ICache请求而
  * 之后```tick```函数干的事情就是注册流水线寄存器```toDecode```：将```fetch```函数注册到```fetchQueue```结构中的指令放到里面去。注意到这里采用了轮流来的算法，首先随机从某个线程开始，然后每个线程向```toDecode```里面注册一条指令，直到```fetchQueue```空或者是译码带宽满
  
* fetch流水级的状态机：
  * ```ThreadStatus```的状态列举如下：
  * 1. Running: 顾名思义，其将会在```fetch```函数中检查curMarcroop或者是fetchBuffer是否还有效来决定是否发送访存请求（这样子就有可能进入ItlbWait状态）
    2. Idle ***冗余代码***
    3. Squashing 当在```checkAndUpdate```函数中检测到sqush信号后将转为该状态，ing意味着是个进行式，会在下一次调用```checkAndUpdate```函数的时候如果不进入其它状态的话重新回到Running状态
    4. Blocked 当commit阶段决定清空流水线之后，将```stalls[tid].drain```拉高，```checkAndUpdateSignals```函数调用```checkStall```函数将进程引入```Blocked```状态，注意到```stalls[tid].drain```变量只会在```takeOverFrom```函数中重新拉低，也就是说操作系统重新将该进程装载到CPU中的时候（当然了，```stalls[tid].drain```当然也会在一些reset函数中被调用
    5. Fetching：***冗余代码***
    6. TrapPending 该信号在```finishTranslation```函数中发现TLB地址翻译出现错误后置起；顺便说一下这里的处理：指令被标记上fault，并且视为一条nop指令一直流到commit阶段，然后向fetch阶段发来squashing信号，然后由```checkAndUpdateStatus```函数检查该信号并将fetchStatus改为Squashing
    7. QuiescePending
    8. ItlbWait IcacehWaitResponse  IcacheWaitRetry IcacheAccessComplete  noGoodAddr：访存相关，之前介绍较为详细，略。
  
* Q&A:

  * 
