# GEM5提交模块

* ```tick```函数

  * 提交模块的主函数

* ```commit```函数

  * 处理来自于之前流水级的标记在指令上的例外，将新的指令加入到ROQ中，并且从ROQ头上取出指令提交

  * 具体来说：首先上来检查是否有需要处理squash的情况，```trapSquash```这个结构体存放的信息是一个线程是否要因为陷入例外而处于squash状态，其分别在以下几个地方被赋值：

    * 首先，该数据结构仅会在```processTrapEvent```函数中被拉高，调用这个函数是用来说明处理器正在处理陷入，而这个函数由函数```generateTrapEvent```构建数据结构```EventFunctionWrapper *trap```，然后调用```cpu -> schedule()```加入到CPU的执行序列中去，顺便一提，函数```generateTrapEvent```会将```trapPending```状态拉高，表明处理器有陷入未处理，而当加入到执行序列里的这个```trap```经过```latency```拍被执行的时候就会将```trapSquash```真正拉高
    * 该状态会在多个情况下被清零，比如说在初始化以及reset的时候，调用```takeOverFrom```函数的时候，我们在此重点看一下其在函数```squashFromTrap```函数中的调用：根据我们的猜测，上一段中所说的加入到执行序列中的陷入被执行完毕的时候，应该调用该函数。查看代码得知，```squashFromTrap```函数是在```commit```函数检查```trapSquash```为高后调用

  * 然后我们来看```tcSquash```这个数据结构，__TODO__

  * 然后函数检查当前状态是否为```SquashAfterPending```：这个状态被带代码注释解释的很好：__TODO__

    * > A squash from the previous cycle of the commit stage (i.e. ```commitInsts()``` called ```squashAfter```) is pending. Squash the thread now.

  * ```commit```函数在检查完会将commitStatus状态转化为```ROBSquashing```的上述三种情况之后 ，开始处理来自IEW的信息来决定是否要将ROB中的某些指令排出，判断条件是```fromIEW->squash()```为真且CommitStatus不处于trapPending状态（因为之后trapSquash会拉高，流水线中的指令迟早都会被清空，因此如果commit阶段正处在trapPending的状态的话，就忽略来自IEW的sqush信息）且发出squash请求的指令要比ROB中最年轻的指令要老（不然就见鬼了）；IEW阶段发起的squash请求分为两类：一类是发生了分支指令误预测，一类是发生了order violation（比如相关的load越过了store），commit函数会在此处打印相关信息。有趣的是，进入这个if分支之后，commitStatus会被设为```ROBSquashing```，而在Commit函数一开始检查squash信息为真的时候也会置成```ROBSquashing```:一个是来自IEW的外因，一个是Commit阶段的内因。由外因置起```ROBSquashing```后便处理反馈给IEW的信息```toIEW```：

    * __TODO__

  * 当检查完线程是否进入```ROBSquashing```状态后，若有任一线程未处于该状态，则调用```getInsts()```函数，```commitInsts()```函数，分别从Rename阶段获取指令并注册到ROB中，以及真正的提交指令

    * __Q__:再次有一个困惑，如果陷入ROBSquashing状态不是由内因引起（由内因引起会清空ROB）而是有比如IEW阶段发现了分支误预测而引起的话，难道就不能让旧指令从ROB中推出吗？

  * ```commit```函数接下来进入了收尾工作，主要干的事情就是生成反馈给IEW的信息```toIEW```：具体来说，就是生成了```usedROB```, ```freeROBEntries```变量，然后判断ROB是否为空来生成```emptyROB```变量。注意到，进入if分支的判断条件是```changedROBNumEntries```，其在调用```squashAll```函数，```getInsts```函数，```commitInsts```函数中会被拉高

* ```getInsts```函数

  * 本函数做的事情就是从rename阶段取出已经做完重命名的指令，把他们注册到ROB中
  * 该函数比较简单，在此不赘述
  * 感兴趣的是if分支的判断条件中的```inst->isSquashed()```，我们来梳理一下这个变量什么时候拉高的：主要是由```BaseDynInst```类中的```setSquashed()```函数拉高，比如commit函数调用的那些squash函数里面都会调用```squashAll```函数，接着就会调用```rob->squash()```，而里面会调用```doSquash()```函数，而里面会调用 ```(*squashIt[tid])->setSquashed()```函数

* ```commitInsts```函数

  * 主要干活的函数，该函数一开始的注释表达了两个意思：
    * 首先，只会试图commit当拍中已经处于ready状态的指令，而不是当拍准备变成canCommit状态的来自于```fromIEW```的指令，这一点是通过先调用```commitInsts```，后调用```markCompletedInsts```函数来实现的
    * 然后，当线程处于```ROBSquashing```状态时，不会提交指令，这一点是通过调用函数```getCommittingThread```，发现线程不处于某几个状态时就返回无效的线程号来实现的:__还是之前那个问题__：当commit阶段由于受到IEW发过来的分支悟预测而处于```ROBSquashing```状态时，为什么不能在ROB头上提交指令？__不懂啊__
  * 具体来看，```commitInsts```函数的主体是一个长度为提交带宽的while循环，循环内部，首先调用函数```getCommittingThread```来根据SMT算法选出哪个线程要提交指令，然后检查是否有中断，对于中断的处理流程尙不熟悉，在此梳理一下：
    * 首先来看变量```interrupt```什么情况会被赋值：在函数```propagateInterrupt```中，其在```commit```函数一开始被调用，如果系统处在FullSystem情况下的话。而我们并不感兴趣GEM5的Full System模式，Syscall Emulation对我们来说就够了，因此不必关心```interrupt```这个变量
  * 综上所述，我们并不关心interrupt，继续往下看。如果准备提交指令的线程号是无效的，或者该线程的ROB头上的指令不是canCommit状态的，代码退出循环(break)，__为什么不是继续下一个循环（continue)???__：仔细看了一下代码，发现确实应该是break，因为函数```getCommittingThread```如果返回的线程号是无效的话，说明所有的线程都不能够提交指令了。if分支的两个判断条件```commit_thread == -1```和```!rob->isHeadReady(commit_thread```似乎是重复的，因为在函数```getCommittingThread```里面只有满足了后者才有可能返回有效的线程号
  * 继续，根据要提交的线程号从ROB头上拿出要提交的指令，然后根据是否是```isSquashed```来进行相应的处理，如果头指令是squash状态，则直接调用```retireHead```函数将其从ROB中移除：这个函数主要干了三件事情，一是将头指令从该线程的指令列表中删除，而是更新ROB中全局的最老的指令，三是从全局的指令列表中将此指令删除
  * 如果头指令不是squash状态的话，就调用```commitHead```函数来试图将头指令提交，如果提交成功的话：
    * 首先是要处理一些与```hardware transactional memory```相关的东西，主要是更新```htmStarts```, ```htmStops```这两个变量，**TODO**：到底什么是HTM呢？不懂啊
    * 然后有一段比较**诡异**的代码，根据tid是不是0来决定是不是要更新```canHandleInterrupts```变量，**TODO**：这个tid=0有什么特殊的地方吗？
    * 接下来检查store conditional是不是被正确处理，这里的```readPredicate()```是什么意思？**TODO**
    * 然后是更新misc regs **TODO**
    * 注意到，如果被成功提交的头指令标记上了```squashAfter```，调用```squashAfter```函数来处理；如果处理器的提交阶段处于```drainPending```状态的话，检查当前是不是’指令边界‘（microPC为空，没有中断，线程不处在`drainPending`状态），来决定是否清空流水线：调用```squashAfter```函数来清空ROB中的某些指令，调用`cpu->commitDrained(tid)`函数来调用前端取指模块的`drainStall`函数，将前端对应的线程处于堵塞状态
    * 接下来是一段维护`oldPC`的代码，利用PC来检查提交是否出错

* ```commitHead```函数

  * 主要干活的函数，主要干三件事情，一是处理```isNonSpeculative```的情况，二是检查是不是要陷入例外，三是真正的提交指令

  * 首先来看第一种情况，也就是头指令并没有执行(```!head_inst->isExecuted()```)，但状态却被标记为了```canCommit```，然后进入了```commitHead```函数来试图提交该指令的情况。我们认为进入这种情况的，有两类合法的指令类型，一类是为了保证读写原子性与访存序的ld, st指令，一类是```head_inst->isNonSpeculative()```类的指令：第一类指令需保证流水线中没有比他更老的指令的情况下才会执行（比如LA架构中的```dbar```, ```ibar```类的栅栏指令__?__）；第二类指令我们目前尙不熟悉，梳理如下：

    * ```head_inst->isNonSpeculative()```函数返回的是```isNonSpeculative```这个flag，这个东西的定义在```src/cpu/StaticInstFlags.py```里面：

    * > ```'IsNonSpeculative', # Should not be executed speculatively```

    * 因此我们知道了这类指令说的是不能推测执行，只能够在commit后执行，因此我们这里看到了该指令成为了```head_inst```之后才会进行处理，而这些指令是在什么时候被设置为```canCommit```状态的呢？在IEW阶段的```dispatchInsts()```函数中通过判断指令是不是我们上述说的两类来调用```inst->setCanCommit()```函数来将其置为可提交状态

  * 做完了触发```!head_inst->isExecuted()```的指令合法性检查之后，判断当拍是否已经有提交的指令或者st queue仍有为完成的st指令，这时候退出```commitHead```函数，返回```false```，于是在```commitInsts```函数中，通过break来退出大的while循环，从而实现真正的非猜测执行(regmap等的更新要在下一拍才能看到)。否则的话，就注册```toIEW```的信息，然后下一拍让这些非猜测执行类的指令去放心的执行

  * 接下来```commitHead```函数来检查是否有例外需要处理，在进入例外处理的if分支前还干了两件事情：如果例外发生的指令是在hardware transactional memory的话：

    * 这段HTM相关的代码没看懂，**TODO**

  * 第二件事情就是如果指令没有标记上例外并且不是st指令的话，将该指令标记为completed

  * 然后进入了例外处理的分支，首先为了实现精准例外，检查st queue里面是否有未完成的指令或者当拍已经有指令提交(等待regmap等结构更新)；然后排除掉模拟器运行错误而引发的例外，通过调用```cpu->checker->verify()```

    * 这个函数扫了一眼，挺有趣的，**TODO**

  * 在执行例外处理函数之前，首先将线程的```noSquashFromTC```拉高，这里**似乎**是为了保证例外的执行不会产生额外的squash；然后调用函数```cpu->trap()```来处理例外，具体来说是调用```inst_fault->invoke()```函数来实现的，这里的代码写的很巧妙，各种例外继承了同一个基类```FaultBase```，然后这里的```inst_fault```的数据类型是```shared_ptr<FaultBase>```，各个fault类的revoke函数的定义在```src/sim/faults.cc```文件中

  * 调用完```cpu->trap()```函数之后，```noSquashFromTC```拉低，同时```commitStatus```改为```TrapPending```，这里的将```TrapPending```状态拉高实际上是冗余代码，因为接下来会调用```generateTrapEvent```函数(里面会将```TrapPending```置起)，再然后```commitHead```就```return false```，结束了指令例外的处理，退出了```commitHead```函数

  * 如果头指令不是非猜测执行指令，也没有标记上例外的话，就开始真正的提交。首先是调用```updateComInstStats```函数来更新一些计数器，以及打印一些信息之类的；然后是更新重命名表；对于HTM类指令的话更新一些信息；然后是调用```retireHead```函数将头指令清空

  * 然后退出```commitHead```函数

* ```updateHead```函数

  * 目的清楚，略 

* ```markCompletedInsts```函数

  * 从IEW阶段的queue里面找到完成的指令，然后标记上ready commit的状态
