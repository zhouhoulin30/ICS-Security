# IFFSET：In-Field Fuzzing of Industrial Control Systems using System Emulation

# IFFSET：使用系统仿真的工业控制系统现场模糊

## 摘要

工业控制系统（ICS）在过去十年中不断发展，从专有软件/硬件转变为配备开源操作系统的当代嵌入式架构。IT世界不同，后者预期会持续更新和打补丁，对始终在线的ICS进行安全评估可能会给其所有者带来过高的成本。所以，必须有一种解决方案，用于定期评估各种ICS的网络安全状况，而不影响其运营。因此，在本文中，我们介绍IFFSET，一个平台，利用基于Linux的ICS固件的全系统仿真，并使用模糊测试进行安全评估。我们的平台从实时ICS设备中提取文件系统和内核信息，构建一个通过QEMU在桌面系统上仿真的映像。我们采用模糊测试作为安全评估工具，分析ICS特定的库并找出潜在的安全威胁条件。我们使用商用PLC测试了我们的平台，展示了潜在威胁，而不中断控制过程。

**索引术语—工业控制、仿真、模糊测试**

## Ⅰ. 简介

工业控制系统（ICS）促进了在工业环境中对物理过程的控制、监控和指导，实现了强大的自动化同时最小化了人的依赖。ICS在工业环境中的普遍存在也已扩展到关键基础设施，因为包括电厂、海水淡化厂和炼油厂在内的设施都配备了此类设备。因此，ICS被认为是功能性关键设备，因为它们的操作中断可能导致收入损失、环境灾难或甚至人命损失[1]。

鉴于它们在所服务环境中的关键作用，ICS的生产主要遵循了稳健性和耐用性的设计原则，以确保功能不间断。然而，近期事件和研究工作已经突显出来自网络领域的潜在威胁，这些威胁可能通过这些设备危及整个设施[2]。Stuxnet是最著名的一个案例，这是一种在伊朗核设施中发现的恶意软件，针对铀浓缩离心机[3]。

这些威胁对关键基础设施ICS的吸引力突显了考虑这些设备的网络安全态势的需求。然而，ICS从未以网络安全威胁为设计考虑，因此在这方面提供的保护最小。此外，当前连接互联网的部分PLC采用了基于通用操作系统的固件进行部署，如Linux（表Ⅰ），为信息技术（IT）威胁进入操作技术（OT）领域搭建了桥梁。有鉴于此，评估和评价ICS的安全能力变得至关重要。然而，这些设备不能随意停用，因为它们的功能通常是连续的，应该无间断进行。此外，ICS是昂贵且多样化的硬件。这些威胁对关键基础设施ICS的吸引力突显了考虑这些设备的网络安全态势的需求。然而，ICS从未以网络安全威胁为设计考虑，因此在这方面提供的保护最小。此外，当前连接互联网的部分PLC采用了基于通用操作系统的固件进行部署，如Linux（表I），为信息技术（IT）威胁进入操作技术（OT）领域搭建了桥梁。有鉴于此，评估和评价ICS的安全能力变得至关重要。然而，这些设备不能随意停用，因为它们的功能通常是连续的，应该无间断进行。此外，ICS是昂贵且多样化的硬件。这些因素创造了一种非侵入性和跨平台的安全评估ICS现场的必要性。

[![pF4EQat.png](https://s21.ax1x.com/2024/03/24/pF4EQat.png)](https://imgse.com/i/pF4EQat)

为此，仿真在桌面或嵌入式系统的生产和测试中已被常规使用，作为一种便利的评估手段。虽然仿真可以提供适应性强、非侵入式的平台，但安全评估本身是一个不同的组成部分。模糊测试是一种用于发现可能导致妥协状况的边界情况的方法。在当代文献中，模糊测试被探索为评估嵌入式系统的有力手段，展示了在揭露威胁方面的强大结果。

在本文中，我们介绍了IFFSET（InField Fuzzing through System Emulation Tool），这是一个与硬件无关的工具，用于安全评估基于Linux的工业控制系统（ICS），通过仿真和模糊测试来全面测试目标平台的安全性。我们的平台通过常规网络连接连接到现场ICS设备，以非侵入式方式提取固件，并利用QEMU框架建立仿真实例。通过结合目标设备的文件系统和内核，我们的平台实现了完整的系统仿真，然后利用这一仿真环境对设备固件的组件进行模糊测试。我们针对兼容Codesys框架的设备，模糊测试处理控制逻辑加载和执行的平台特定库。我们的平台在发现产品二进制文件和平台特定库中的系统威胁条件方面取得了成功，同时在模糊测试性能上比实际设备提高了一个数量级。

## Ⅱ. 问题描述

假设的场景如下：具有基于Linux的固件的潜在易受攻击的设备在工业环境中控制一个关键功能。设备在访问其局域网之外是隔离的。设备不能被拔下插头、重启或以任何方式抑制其系统功能。由于设备与外部网络访问隔离，因此除了通过本地网络（SSH或FTP）之外，没有直接与设备通信的方式。必须从安全性方面对设备进行评估，以识别潜在的现有漏洞，从而决定是否可以保持原样、进行更新或删除和更换。

## Ⅲ. IFFSET

本节介绍IFFSET背后的主要概念，IFFSET是我们针对基于Linux的PLC固件的仿真框架。本文中的固件是作为系统软件运行的通用操作系统和处理执行控制逻辑的二进制文件/应用程序的运行时环境的组合。运行时作为单一应用程序托管在操作系统中，并包含在一个简单的$\texttt{ELF}$二进制文件中，连同所需的动态加载库。

### A. IFFSET开发

**固件获取：**在这第一步中，IFFSET从目标设备获取文件系统以及内核信息。在基于Linux的实时机器上，现有的文件和文件夹是启用系统引导所需文件和文件夹的超集。IFFSET需要$\texttt{proc}$文件夹中的两个文件用于提取和准备阶段，即$\texttt{Kconfing}$和$\texttt{proc\_version}$。这些文件提供了必要的信息，内核配置和当前版本，指导正确的源代码选择。临时或启动生成的文件夹被省略，因为它们包含的静态信息将被处理过程中的脚本视为启动生成的，导致启动失败。

```python
在一个运行Linux的活动（正在使用的）计算机系统中，存储在硬盘上的所有文件和文件夹数量和类型，比那些仅仅用于启动和运行操作系统所必需的文件和文件夹要多。换句话说，操作系统启动和正常运行所必需的文件和文件夹构成了一套基础集合，而在实际运行的Linux机器上，这套基础集合被更多的文件和文件夹所扩展，这些额外的文件和文件夹用于各种其他功能和应用，但不是启动系统所必须的。这里提到的“超集”是数学和集合论中的概念，指包含另一集合所有元素并可能还有额外元素的集合。

至于临时或启动生成的文件夹被省略的原因，这是因为这些文件夹包含的信息在系统启动过程中会被重新生成，因此它们不是固件获取或仿真过程中必需的。如果这些临时或启动生成的信息被包括进去，处理过程的脚本可能会误认为这些信息是启动时必须的静态信息，并尝试在仿真环境中重现它们，这可能导致启动流程失败，因为仿真环境无法准确地重现实际硬件在启动时生成的临时数据。简而言之，为了避免仿真时出现启动失败的问题，这些由启动过程生成的临时文件夹和信息被省略不考虑。
```

**提取与准备：**对于固件的提取，我们利用了Chen等人在2016年介绍的基于定制$\texttt{binwalk}$的实现方法[7]。为了给仿真阶段准备必要的文件，IFFSET遵循两个主要程序：a）使用在获取和提取步骤中收集的信息构建内核，以及b）修改初始化脚本以成功启动仿真系统。提取的配置文件包括构建设备本身使用的内核所需的所有必要信息，包括生成的映像类型和目标文件系统类型。用于存储、网络、GPIO等的控制器必须使用内核中包含的现有驱动程序连接到设备。根据评估的系统组件，一些设备被排除在配置文件之外，但有些设备对于评估阶段是必不可少的，例如看门狗定时器。

```python
看门狗定时器（Watchdog Timer）是一种硬件或软件定时器，它在系统或程序运行过程中发生故障时提供一种自动恢复机制。如果系统或应用程序正常运行，它会定期重置或“喂狗”（即更新定时器），以防止定时器超时。如果因为软件错误、硬件故障或其他问题导致系统挂起或无响应，看门狗定时器将在未被及时重置的情况下超时。一旦超时，看门狗定时器可以配置为执行一系列恢复动作，比如重启系统、记录错误信息或触发其他恢复流程。这种机制可以提高系统的可靠性和稳定性，尤其是在那些需要长时间自主运行且无人值守的嵌入式系统或工业控制系统中。
```

最后，对系统启动$\texttt{init}$阶段部署的脚本中进行了一些修改。在挂载根文件系统后立即错误地调用$\texttt{rcS}$脚本会导致内核死机和系统崩溃。对于这种情况，一个预初始化脚本确保了必要的文件夹被创建。根据目标设备的评估配置，从外围设备的角度来看，$\texttt{rcS}$文件中的一些行也被删除，以避免给过程的其余部分带来复杂性，此外还从$\texttt{rc.d}$文件夹中移除了与配置无关的脚本文件。最后添加的组件是将在评估中使用的模糊测试二进制文件，这些文件根据通过$\texttt{binwalk}$提取的信息为正确的架构编译，并且可以在仿真实例上本地执行。

**仿真：**设置QEMU使用主机系统的单个CPU核心，以便允许多个实例同时运行。内存和存储配置遵循典型嵌入式系统硬件，具有256 MB的主内存和1 GB的QEMU映像作为存储。

**模糊测试：**在我们工具的模糊测试阶段，我们考虑了固件的两个组成部分。我们针对操作系统$\texttt{bin}$文件夹中的通用二进制文件和实现运行时的二进制文件以及相关的动态库进行了目标设置。对于典型的二进制文件模糊测试过程，我们通过文件元数据确定了二进制构建工具是OSELAS工具链，这是一个源自GNU的用于嵌入式Linux的开发工具链。OSELAS是一个便利的二进制构建工具，但它不是典型的、更加开放透明的GNU开发过程的一部分，且每年只更新一次。这给固件二进制文件引入了一个不可预测因素，因为构建工具中未发现的问题有可能威胁到构建的二进制文件，这增加了彻底测试的必要性。这给固件二进制文件引入了一个不可预测因素，因为构建工具中未发现的问题有可能威胁到构建的二进制文件，这增加了彻底测试的必要性。

除了常规的二进制文件，我们还考虑了特定于平台的二进制文件，在本例中是实例化Codesys 2.3运行时环境的二进制文件。这个运行时环境是在部署的工业控制系统（ICS）中应用控制逻辑的主要工具。我们基于其在工业领域的普及程度选择了支持Codesys的设备，超过250家PLC制造商支持他们产品使用Codesys框架。这个二进制文件被构建为一个$\texttt{ELF}$文件，并且是专有平台的一部分，因此自然是闭源的，有关其功能的信息很少。我们尝试直接使用我们选择的模糊测试工具——American Fuzzy Lop（AFL），通过可用的动态插桩技术对这个二进制文件进行模糊测试，但我们的尝试没有产生有用的结果。因此，我们转向逆向工程以更好地理解这个二进制文件的执行流程和功能。我们使用$\texttt{ghidra}$提取了二进制文件的大致反编译视图，并且，由于它是一个定义良好的$\texttt{ELF}$文件，我们从$\texttt{main}$函数开始并跟踪了每一个被调用的函数。虽然它是一个独立的二进制文件，但其主要目标是创建任务，将这些任务实例化为线程，这些线程是负责执行服务的实体，如网络接口和输入/输出调解。创建线程后，二进制文件进入睡眠周期，大部分运行时间都被挂起。由于AFL不会将自己附加到被模糊测试的二进制文件克隆或分叉线程，我们简单模糊测试结果的缺乏得到了解释。==因此，我们只剩下一种模糊运行时二进制文件的方法，将函数从动态链接库中隔离出来，并从自开发的二进制文件中调用它们，即测试工具。从一系列链接的共享对象中，我们选择了与网络和I/O相关的对象，因为它们与环境的直接接触使它们成为更有趣的利用目标。其中一个被测试的功能是在[8]报告中提到的网络引入的利用的一部分。==

```python
输入输出调解（input/output mediation）是一个计算机系统管理和控制数据流向和从输入设备（如键盘、鼠标、传感器等）到输出设备（如显示器、打印机、扬声器等）的过程。这个过程涉及到数据的接收、处理、转换和发送，确保数据正确无误地从源头移动到目的地。在工业控制系统（ICS）或任何复杂系统中，输入输出调解是关键的，因为它涉及到从外部环境接收指令和反馈，以及控制系统对这些指令的响应和动作。

输入输出调解的目的是：
确保数据在正确的时间以正确的格式被发送和接收。
为系统内部处理和外部通信提供一个清晰、可靠的数据流通道。
处理并转换不同类型的数据和信号，以便它们可以被系统内的不同组件使用。
例如，在一个自动化制造过程中，传感器收集的数据（输入）需要被处理和解释，然后系统可能会做出决定来控制机械臂的动作（输出）。输入输出调解确保这些步骤按照预定的逻辑和时间顺序发生，保证了过程的准确性和效率。


在软件测试中，"harness"（测试工具）指的是一套用来自动执行、监测和管理测试用例的工具和代码。它通常包括测试脚本、测试数据、以及执行测试并报告结果的软件。在模糊测试（fuzzing）的上下文中，一个测试工具（harness）特别设计来触发和监视被测试软件（通常是一个程序或库）的行为，尤其是在面对异常或随机生成的输入数据时的行为。这种测试工具的目的是自动化测试过程，发现潜在的错误或漏洞，而不需要人工干预。
```

对于模糊测试的结果，分析考虑了由各种测试输入引起的崩溃和挂起。崩溃是两种情况中更直接的一种，因为它们是已解决的事件。当一个二进制文件崩溃时，系统软件会产生一个错误信号以及相关数据，这些数据可以指出崩溃的原因，例如一个未识别的bug。挂起则是另一种情况，探索起来很有趣，但除了给定的输入外，它几乎没有提供太多信息来修复其原因。然而，挂起对于可靠性以及可用性评估来说很有趣，它提供了目标程序执行挂起的原因。

**IFFSET的正确性：**由于我们在仿真实例上而不是设备本身进行安全评估，自然会产生安全评估是否具有代表性的问题。为了证明IFFSET结果的适当性，我们进行了实验，测试了仿真系统和原始设备之间威胁检测的相关性。在现有文献中[9]，正确性通常通过开发一个带有漏洞的程序，并在开发的平台以及设备本身上运行模糊测试来确认。因此，在本文中，我们开发了一个C程序，其中插入了12个将导致崩溃的漏洞，这些漏洞的灵感来自于[10]中的漏洞插入。这些漏洞包括基于栈和堆的缓冲区溢出、空指针异常和越界读取。我们使用二进制插桩工具和手动选择的输入种子来加速漏洞发现过程。图1展示了这次正确性检查的结果。图表显示IFFSET和直接在PLC上执行模糊测试都能检测到所有12个漏洞。此外，IFFSET检测到崩溃的速度要快得多（这将在进一步的实验结果中详细阐述），因为运行IFFSET的计算机与嵌入式PLC相比执行速度要快得多。

![image-20240324214034245](C:\Users\15647\AppData\Roaming\Typora\typora-user-images\image-20240324214034245.png)

**IFFSET的适用性：**我们开发了IFFSET，使其具有通用性，可扩展性和硬件无关性，因此可以用于评估任何基于Linux的PLC，无论CPU架构如何。通过利用设备树、$\texttt{Kconfig}$文件和系统文件信息，IFFSET可以适应QEMU仿真所支持的各种CPU架构的多种外设配置。固件获取过程针对基于Linux系统中的关键文件和文件夹，在准备过程中，IFFSET根据提取的系统信息而不是固定模型调整文件。为了测试IFFSET的适用性，我们已经成功地将其部署在了Wago PLC（ARM）、西门子PLC（x86）和BeagleBone开发板（ARM）上，这些都是具有基于Linux的操作系统的设备。

## Ⅳ. IFFSET评估

为了对IFFSET进行实验评估，我们选定了Wago PFC-100和Siemens Simatic IOT2020进行测试。托管IFFSET的计算机配备了i7处理器和16GB的RAM。作为一个示例过程，我们已经将PLC集成在一个多阶段闪蒸海水淡化厂的硬件在环（HITL）仿真中。我们选择了20个最常用的Linux实用程序，除了Codesys运行时二进制文件/库之外，它们还接收定义的输入。如第三节所述，我们选择了AFL，一种开源的模糊测试工具，它采用输入变异功能并包含仪器化能力。我们的实验关注平台的多个方面：证明固件提取过程中对目标设备没有影响，关于性能的模糊测试结果，以及发现的挂起/崩溃实例。我们对系统二进制文件和提取的Codesys运行时库进行了2小时的模糊测试流程。

### A. 非侵入性

我们首先考虑了固件提取过程可能对目标工业控制系统（ICS）及其托管的控制逻辑过程产生的潜在影响。

我们在执行脱盐过程的实时设备上进行了固件提取，同时监控由实现的控制逻辑直接控制的数量。图2展示了这些值随时间的波动，当设备处于空闲状态以及IFFSET部署时的情况，假设IFFSET在时间=$0^{3}$时部署。该图清楚地表明，IFFSET不具侵入性，不以任何方式影响过程。

[![pF4l5B8.png](https://s21.ax1x.com/2024/03/24/pF4l5B8.png)](https://imgse.com/i/pF4l5B8)

### B. 模糊测试结果

固件二进制文件：在这一部分，我们针对20个最常用的Linux二进制文件。表Ⅱ展示了IFFSET分析的结果，展示了在2小时测试中发生的挂起和崩溃实例数量以及执行的操作。

[![pF4l4nf.png](https://s21.ax1x.com/2024/03/24/pF4l4nf.png)](https://imgse.com/i/pF4l4nf)

挂起的数量自然大于崩溃的数量，因为一个挂起的二进制文件的原因也可能与系统有关，而且模糊测试工具不会无限期地等待。另一方面，二进制文件的崩溃仅归因于二进制文件本身，作为基于格式错误的输入的自身执行的产物。关于崩溃，我们发现了已知且已修补的漏洞，如[11]，这些漏洞在当前的Linux版本中已被纠正，但在我们测试的设备中仍然可以被利用。bash展示了最多的未发现崩溃，这是其复杂性相比其他测试二进制文件的一个副产品。这种情况也在报告的CVEs中见证，其中bash的报告漏洞数量是其他测试二进制文件的五倍多。

**Codesys运行时：**表Ⅱ列出了对利用的动态库中提取的Codesys运行时函数的模糊测试结果。在我们测试的18个函数中，14个产生了崩溃实例，其中5个列在表Ⅱ中。我们针对这次实验的是各种实用函数。$\texttt{SysSockRecv}$是运行时TCP软件堆栈的数据接收部分，是上述利用[8]的目标。两个$\texttt{kbus}$函数是控制逻辑应用与GPIO之间输入传输的辅助器。结果说明了对于固件安全，这样的平台特定关键部分作为一个整体进行安全评估的必要性。这些函数是“面向前端”的，与系统外部的实体（例如网络、GPIO等）有直接接触，使它们更有可能被利用。对这些函数的模糊测试方案并不完美：我们执行了运行时加载的库函数，但基于逆向工程模拟了它们的执行上下文信息。然而，结果确实值得注意，通过相对简单的方法如模糊测试，展示了容易发现的潜在漏洞。

### Ⅲ. 模糊测试性能

图3展示了使用AFL进行模糊测试时IFFSET的性能结果。为了进行比较，我们还使用了Wago和Siemens PLC进行了模糊测试过程；换句话说，我们在PLC上加载了模糊测试工具并直接对二进制文件进行了模糊测试。此外，考虑到我们的使用场景假设一个桌面级计算机执行模糊测试，我们还展示了使用所有CPU核心（4核/8线程）时的模糊测试性能。如预期的那样，仿真平台的性能远远超过了原始系统。我们的结果显示，在与原始系统相比，1核心和4核心仿真执行在不同基准测试中平均分别提速了440%和1520%。这个结果并不令人惊讶，因为仿真实例利用了桌面或服务器级硬件，其性能远远超过了像ICS这样的低功耗嵌入式设备。尽管如此，同一个宿主中可以并存多个相同系统的实例，如同使用了4个核心的4个并发实例所展示的那样，提供了极好的可扩展性。

[![pF4lfjP.png](https://s21.ax1x.com/2024/03/24/pF4lfjP.png)](https://imgse.com/i/pF4lfjP)

## Ⅴ. 总结

在这篇论文中，我们介绍了一个用于现场评估工业控制系统（ICS）网络安全状况的新颖框架。我们的框架通过QEMU实现系统仿真，并使用成熟的模糊测试工具对常用的系统二进制文件以及特定平台的库进行测试。我们已经在商用PLC上测试了我们的平台，展示了导致二进制文件挂起或崩溃的实例，以及高达15倍的性能提升。

## ACKNOWLEDGMENT &RESOURCES

这项工作得到了纽约大学阿布扎比全球博士奖学金项目的支持。我们的平台可以在https://github.com/momalab/IFFSET找到。