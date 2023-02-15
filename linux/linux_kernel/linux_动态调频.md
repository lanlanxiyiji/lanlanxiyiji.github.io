## 1. 提供的功能
主要解决两个问题：什么时候调频调压，怎么调频调压。
功能抽象出几个模块：
cpufreq driver提供调频调压的机制，这一层可以为上层规避掉硬件信息，提供统一的向上接口。
cpufreq governor提供不同的策略
cpufreq core对通用的调频逻辑做抽象，为上层提供功能、接口封装，对下层调用抽象封装的硬件控制接口

## 2. 框架
![Octocat](https://nclovestars.github.io/images/20230215095441.png)

-   cpufreq core：是cpufreq framework的核心模块，和kernel其它framework类似，主要实现三类功能：
	- 对上，以sysfs的形式向用户空间提供统一的接口，以notifier的形式向其它driver提供频率变化的通知；
	- 对下，提供CPU频率和电压控制的驱动框架，方便底层driver的开发；同时，提供governor框架，用于实现不同的频率调整机制；
	- 内部，封装各种逻辑，实现所需功能。这些逻辑主要围绕struct cpufreq_policy、struct cpufreq_driver和struct cpufreq_governor三个数据结构进行。


-   cpufreq governor：负责调频调压的各种策略，每种governor计算频率的方式不同，根据提供的频率范围和参数(阈值等)，计算合适的频率。
	-   userspace：用户通过操作scaling_setspeed文件节点操作频率及电压的调整。
	-   ondemand：根据CPU当前的使用率，动态调整cpu的频率及电压。Sched通过调用ondemand注册进来的回调函数来触发负载的估算，它以一定时间间隔对系统负载进行采样，按需调整cpu的频率及电压，若当前cpu的利用率超过设定的阈值，就会立即调整到最大的频率。调频速度快，但是不够精确。
	-   conservative：类似ondemand，在调频调节时会平滑一下，以防最大、最小频率之间来回跳变。调整的时候会以一定步长调整，而不是直接调整到目标值。同时会周期的计算系统负载，用以决定调到什么频率。
	-   schedutil：通过将自己的调频策略注册到hook，在负载发生变化的时候，会调用该hook，此时就可以进行调频决策或执行调频动作。前面的调频策略都是周期采样计算cpu负载有滞后性，精度也有限，而schedutil可以使用PELT(per entity load tracking)或者WALT(window assist load tracking)准确的计算task的负载。如果支持fast_switch的功能，可以在中断上下文直接进行调频。

-   cpufreq driver：负责平台相关的调频调压机制的实现，基于cpu subsystem driver、OPP、clock driver、regulator driver等模块，提供对CPU频率和电压的控制。是平台驱动开发工程师需要关注的结构。

-   cpufreq stats：负责调频信息和各频点运行时间等统计，提供每个cpu的cpufreq有关的统计信息。