今天了解了下Cpu、指令集等概念。



***

所有编程语言都要经过转换，才能被Cpu识别，Cpu所能识别的语言，有专门的规范约束，这种规范就是指令集(ISA，Instruction Set Architecture)

常见的指令集：x86、ARM v8、MIPS

---
<p>
实现了指令集的语言就是微架构（microarchitecture)

常见的微架构：Intel的Haswell微架构，Apple的Cortex架构

</p>

> 业界认为只有具备独立的微架构研发能力的企业才算具备了CPU研发能力，而是否使用自行研发的指令集无关紧要。微架构的研发也是IT产业技术含量最高的领域之一。

---
RISC、CISC只是两种不同的指令集方式。
***

因为最近在看Tomcat实现，所以就拿Tomcat做类比。

我的理解：就是指令集是Servlet规范，cpu是web容器，微架构就是具体的Tomcat容器
