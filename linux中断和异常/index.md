# Linux中断和异常


中断(interrupt)通常被定义为一个事件，该事件改变处理器执行的指令顺序。这样的事件与CPU芯片内外部硬件电路产生的点信号相对应。

中断通常分为同步 (synchronous) 中断和异步 (asynchornous) 中断:
- 同步中断是当指令执行时由 CPU 控制单元产生的，之所以称为同步，是因为只有在一条指令终止执行后 CPU 才会发出中断。
- 异步中断是由其他硬件设备依照 CPU 时钟信号随机产生的。

在 Intel 微处理器中，把同步和异步中断分别称为异常 (exception) 和中断 (interrupt)。

中断是由间隔定时器和I/O设备产生的，异常是由程序的错误产生的，或者是由内核必须处理的异常条件产生的。

## 中断信号的作用
中断信号提供了一种特殊的方式，使处理器转而去运行正常控制流之外的代码。当一个中断信号达到是，CPU 必须停止它当前正在做的事情，并且切换到一个新的活动。所以要在内核态堆栈保存程序计数器的当前值(即eip和cs寄存器的内容)，并把与中断类型相关的一个地址放进程序计数器。

## 中断和异常
中断:
- 可屏蔽中断 (maskable interrupt) : I/O 设备发出的所有中断请求(IRQ)都产生可屏蔽中断。可屏蔽中断可以处于两种状态: 屏蔽的 (masked) 或非屏蔽的 (unmasked)；一个屏蔽的中断只要还是屏蔽的，控制单元就忽略它。
- 非屏蔽中断 (nonmaskable interrupt) : 只有几个危急事件(如硬件故障)才引起非屏蔽中断。非屏蔽中断总是由CPU辨认。

异常:
- 处理器探测异常(processor-detected exception) : 当 CPU 执行指令时探测到一个反常条件所产生的异常。可以进一步分为三组，这取决于CPU控制单元产生异常是保存在内核态堆栈eip寄存器中的值。它们分别是故障(fault)、陷阱(trap) 和异常中止(abort)。
- 编程异常(programmed exception) : 在编程者发出请求时发生。是由 int 或 int3 指令触发的。当 into(检查溢出)和 bound(检查地址出界)指令查询的条件不为真时，也引起编程异常。控制单元把编程异常作为陷阱来处理。编程异常通常也叫做软中断(software interrupt)。这样的异常有两种常用的用途: 执行系统调用和给调试程序通报一个特定的事件。

处理器探测的异常:
- 故障: 通常可以纠正。一旦纠正，程序就可以在不失连贯性的情况下重新开始。保存在 eip 中的值是引起故障的指令地址。因此，当异常处理程序终止时，那条指令会被重新执行。
- 陷阱: 在陷阱指令执行后立即报告。内核把控制权返回给程度后就可以继续它的执行而不失连贯性。保存在 eip 中的值是一个随后要执行的指令地址。只有当没有必须要重新执行以终止的指令时，才触发陷阱。陷阱的主要用途是为了调试程序。
- 异常中止: 发送一个严重的错误，控制单元出了问题，不能在 eip 寄存器中保存引起异常的指令所在的确切位置。异常中止用于报告严重的错误，如硬件故障或系统表中无效的值或不一致的值。由控制单元发送的这个中断信号时紧急信号，用来把控制权切换到相应的异常中止处理程序，这个异常中止处理程序除了强制受影响的进程中止外，没有别的选择。

每个中断和异常是由0~255之间的一个数来标识。Intel 把这个8位的无符号整数叫做一个向量(vector)。非屏蔽中断的向量和异常的向量是固定的，而可屏蔽中断的向量可以通过对中断控制器的编程来改变。


