---
layout: post
title: Standard MIDI File
date:  2015-12-31
author: kuga
---

其实很早之前就想了解 SMF(Standard MIDI File, 标准 MIDI 文件)，
但网上有用的中文资料实在是太少了，而且大多只是形式上介绍标准，跟读天书差不多，
即使是英文资料，大多也是很旧很旧的文章。
幸运的是，当中还是有写得比较有趣的，所以打算翻译一下，也加一些自己的理解。

读完这篇文章之后，你就可以听到自己制作的 MIDI 文件了 ∮ ♪ ♩ ♫ ♬ 。

## MIDI 文件格式(0, 1, 2)
------------------------

MIDI 的文件格式有三种，分别是 0, 1, 2。

* 0: 单音轨格式
* 1: 多音轨格式(最大音轨数为 65535)
* 2: Multiple Song File Format (比较少用)

其中 2 比较少用，这里就不介绍了。
0 和 1 的区别只是文件的存储方式不一样，这两种格式完全可以"无损"转换。
就像 Photoshop 中的概念，一幅图可以由多个或一个层图组成，这里的图层就是我们所说的音轨。
因此，在 Photoshop 中合并多个图层，实际上就相当于在 MIDI 中合并多个音轨。

例如当你想把一首含有 melody 和 base 的音乐转换成 MIDI 格式时：
对于 0 格式，你就需要把 melody 和 base 的信息都记录到一条音轨上。
对于 1 格式，你可以单独把 melody 记录到一条音轨，再把 base 记录到另一条音轨。

#### 块(Chunk)

MIDI 文件是由「块」组成的，一个「Header Chunk」，一个或多个「Track Chunk」

```
SMF = <header_chunk> + <track_chunk> [+ <track_chunk> ...]
```

#### 头部块(Header Chunk)

<table style="margin-top: 30px;">
    <tr>
        <th>&nbsp;</th>
        <th colspan="14">Header Chunk</th>
    </tr>
    <tr>
        <th>字节号</th>
        <td>0</td>
        <td>1</td>
        <td>2</td>
        <td>3</td>
        <td>4</td>
        <td>5</td>
        <td>6</td>
        <td>7</td>
        <td>8</td>
        <td>9</td>
        <td>10</td>
        <td>11</td>
        <td>12</td>
        <td>13</td>
    </tr>
    <tr>
        <th>字节值</th>
        <td>4D</td>
        <td>54</td>
        <td>68</td>
        <td>64</td>

        <td>00</td>
        <td>00</td>
        <td>00</td>
        <td>06</td>

        <td>00</td>
        <td>01</td>
        <td>00</td>
        <td>01</td>

        <td>00</td>
        <td>80</td>
    </tr>
    <tr>
        <th>备注</th>
        <td colspan="4">文件头部标记</td>
        <td colspan="4">头部剩余的字节数</td>
        <td colspan="2">文件格式</td>
        <td colspan="2">音轨数</td>
        <td colspan="2">Tick数</td>
    </tr>
</table>

下面是各个字节段的解释:

* 0~3: MIDI 文件的头部标记，固定值，对应的 ASCII 是 MThd。
* 4~7: MIDI 头部剩余的字节数，固定值，6。
* 8~9: MIDI 文件格式 (0, 1, 2)。
* 10~11: MIDI 的音轨数，对于 0 格式，这个值总是 1，对于 1 格式，最大值是 65535。
* 12~13: MIDI 的速度，一个四分音符包含的 tick 数量，tick 是最小的时间单元。

#### 音轨块(Track Chunk)

<table style="margin-top: 30px;">
    <tr>
        <th> </th>
        <th colspan="13">Track Chunk</th>
    </tr>
    <tr>
        <th> </th>
        <th colspan="8">Header</th>
        <th colspan="1">Data</th>
        <th colspan="4">End</th>
    </tr>
    <tr>
        <th>字节号</th>
        <td>14</td>
        <td>15</td>
        <td>16</td>
        <td>17</td>

        <td>18</td>
        <td>19</td>
        <td>20</td>
        <td>21</td>

        <td>22 ... X</td>

        <td>X+1</td>
        <td>X+2</td>
        <td>X+3</td>
        <td>X+4</td>
    </tr>
    <tr>
        <th>字节值</th>
        <td>4D</td>
        <td>54</td>
        <td>72</td>
        <td>6B</td>

        <td>?</td>
        <td>?</td>
        <td>?</td>
        <td>?</td>

        <td>...</td>

        <td>00</td>
        <td>FF</td>
        <td>2F</td>
        <td>00</td>
    </tr>
    <tr>
        <th>备注</th>
        <td colspan="4">音轨头部标记</td>
        <td colspan="4">音轨剩余的字节数</td>
        <td colspan="1">音乐数据</td>
        <td colspan="4">音轨结束标记</td>
    </tr>
</table>

下面是各个字节段的解释:

* 14~17: 音轨头部标记，固定值，对应的 ASCII 是 MTrk。
* 18~21: 音轨剩余的字节数。
* 22~X: 真正的音乐数据(哎哟不错哦๑乛v乛๑)。
* X+1~X+4: 音轨结束标记。

上面就是一个 MIDI 文件的基本结构了，下面通过一个实际的例子讲解一下「音乐数据」(22~X)这一部分。

## 音乐数据(Music Data)
----------------------

下面是一个「音乐数据」的实际例子，依次播放三个音符 C-E-G，并最终停止。

<table>
    <tr>
        <th>&nbsp;</th>
        <th colspan="16">Music Data</th>
    </tr>
    <tr>
        <th>字节号</th>
        <td>22</td>
        <td>23</td>
        <td>24</td>
        <td>25</td>

        <td>26</td>
        <td>27</td>
        <td>28</td>

        <td>29</td>
        <td>30</td>
        <td>31</td>

        <td>32</td>
        <td>33</td>
        <td>34</td>
        <td>35</td>
    </tr>
    <tr>
        <th>字节值</th>
        <td>00</td>
        <td>90</td>
        <td>3C</td>
        <td>60</td>

        <td>7F</td>
        <td>40</td>
        <td>60</td>

        <td>7F</td>
        <td>43</td>
        <td>60</td>

        <td>7F</td>
        <td>B0</td>
        <td>7B</td>
        <td>00</td>
    </tr>
    <tr>
        <th>备注</th>
        <td colspan="4">Event 1</td>
        <td colspan="3">Event 2</td>
        <td colspan="3">Event 3</td>
        <td colspan="4">Event 4</td>
    </tr>
</table>

例子中包含了 4 个事件，分别是播放音符 C(中央C)，符放音符 E，播放音符 G，停止播放。

#### 事件延迟时间(Event Delay)

所有的事件头部都有一个「延迟时间」，表示该事延迟多长时间后发生。
例子中分别是 00, 7F, 7F, 7F，这里的数值就是之前所说的 tick 数量。
时间是**相对**的，不是**绝对**的，因此上面的事件是这样执行的：

* 00: 立即播放音符 C
* 7F: 延迟 7F 个 ticks 后播放音符 E
* 7F: 延迟 7F 个 ticks 后播放音符 G
* 7F: 延迟 7F 个 ticks 后停止播放

在这里有些同学可能会问(其实根本没人问๑乛v乛๑)，只用一个字节表示时间会不会太少了？
这位同学问得好！当然不够了！所以事件中的头部时间实际上是采用动态字节表示的，要多少有多少！

但这里有一个规则: **最后一个字节的值是必须小于等于 7F，其它字节必须大于 7F**。

所以如果要表示 128 个 ticks，是不可以直接写 80 的，需要用两个字节 8100，
129 个 ticks 就是 8101，256 个 ticks 就是 8200，如此类推。
由于只有最后一个字节是小于等于 7F 的，所以读取时间的时候很容易判断何时停止。

需要注意的地方是当表示 2^14 个 ticks 时，必须写成 818000，不能写成 810000。

#### 音符播放/停止(Node On/Off)

接下来我们着重分析 22~25(字节) 的内容。

<table>
    <tr>
        <th>&nbsp;</th>
        <th colspan="16">Music Data</th>
    </tr>
    <tr>
        <th>字节号</th>
        <td>22</td>
        <td>23</td>
        <td>24</td>
        <td>25</td>

        <td>26</td>
        <td>27</td>
        <td>28</td>

        <td>29</td>
        <td>30</td>
        <td>31</td>

        <td>32</td>
        <td>33</td>
        <td>34</td>
        <td>35</td>
    </tr>
    <tr>
        <th>字节值</th>
        <td>00</td>
        <td>90</td>
        <td>3C</td>
        <td>60</td>

        <td>7F</td>
        <td>40</td>
        <td>60</td>

        <td>7F</td>
        <td>43</td>
        <td>60</td>

        <td>7F</td>
        <td>B0</td>
        <td>7B</td>
        <td>00</td>
    </tr>
    <tr>
        <th>备注</th>
        <td colspan="4">Event 1</td>
        <td colspan="3">Event 2</td>
        <td colspan="3">Event 3</td>
        <td colspan="4">Event 4</td>
    </tr>
</table>

* 22号字节: 表示延迟 00 个 ticks。
* 23号字节: 「9」表示事件 Note On，「0」表示 Channel，
* 24号字节: 表示音符的绝对高度，3C 代表中央C (Middle C)
* 25号字节: 表示音量

连起来读就是: 延迟 0 个 ticks，然后在 Channel 0 中播放中央 C 音符，音量是 60。

另外，24，25号字节相当于事件的参数，不同的事件有不同的参数，个数也可能不一样。

再者，Channel 只有 4 个二进制位，所以最多支持 **16 个 Channel**。
关于 **Track** 和 **Channel** 的区别，我们后面再说，先不管，先继续分析其它字节。

* 26号字节: 表示延迟 7F 个 ticks
* 27号节字: 表示音符绝对高度，40 代表 E，等价于 3C + 4 个半音。
* 28号节字: 表示音量

连起来读就是: 延迟 7F 个 ticks，然后在 Channel 0 中播放音符 E，音量是 60。

又有同学问了，怎么少了一个字节，楼主眼瞎吗 Σ( ° △ °|||)︴？？？
放心，没瞎，**我还得看周杰伦和五月天的演唱会！**
这里是因为后面的事件和前面的事件一样，可以省略掉，因此 90 就可以不写了。

接下来看最后一个部分

* 32号字节: 表示延迟 7F 个 ticks
* 33号字节:「B」表示事件 Note Off，「0」表示 Channel，
* 34号字节: 7B 表示 All Notes Off Controller
* 35号字节: 忽略

最后一个事件不是 Node On 而是 Node Off，因此不能省略事件参数。
好了，终于到了要把这个 MIDI 文件做出来的时候了，大家准备好**可乐**！

## C-E-G Demo
-------------

下面是一个完整的 MIDI 文件：
(**在真实的文件中，字节间是没有空格的，也没有换行，下面只是为了方便观看**)

```markdown
4D 54 68 64 00 00 00 06 00 01 00 01 00 80
4D 54 72 6B 00 00 00 15
00 90 3C 60
81 00 40 60
81 00 43 60
81 00 B0 7B 00
00 FF 2F 00
```

上面第一行标记了 MIDI 的文件格式是 1，音轨数为 1，速度为 80 个 ticks。
第二行是 Track 头，3～6 行是「音乐数据」，第 7 行是 Track 尾。
另外在编写完「音乐数据」后，要记得更新 Track 头的「音轨剩余字节数」的值。

把上面的值写到文件中，名为 **hello.mid**。
(这里有一个<a href="https://github.com/xkuga/hello-midi" target="_blank">小工具</a>，可以在线生成 mid 文件)

Windows 默认是可以直接播放 mid 文件的。
如果你使用 Ubuntu，可以安装 timidity 来播放 MIDI 文件，命令如下:

```bash
$ sudo apt-get install timidity
```

安装完后，播放的命令如下:

```bash
$ timidity hello.mid
```

其实上面的例子还有一个问题，就是当 D 音播放的时候，C 音是不会停的，
当 G 音播放的时候，也是会夹带 C 音和 D 音的，如果我们想单独播放每个音，要怎么办？
有一种的做法是在播放下一音前，先把上一个音消掉，实现时可把音量调为 0，如下:

```markdown
4D 54 68 64 00 00 00 06 00 01 00 01 00 80
4D 54 72 6B 00 00 00 1A
00 90 3C 60
81 00 3C 00
00 40 60
81 00 40 00
00 43 60
81 00 43 00
00 FF 2F 00
```

由于 MIDI 采用**相对时间**，所以在停止上一个音符后，delay 填写 00 可马上播放下一个音符

到这里，你已经对 MIDI 有一个很基本的认识了，干杯( >_< )つロ！！

## Canon
--------

最后是一个「卡农」前奏的例子，旋律是 321765671，和弦走向的级数是经典的 156341451。

<table>
    <tr>
        <th>Melody</th>
        <td>E</td>
        <td>D</td>
        <td>C</td>
        <td>B</td>
        <td>A</td>
        <td>G</td>
        <td>A</td>
        <td>B</td>
        <td>C</td>
    </tr>
    <tr>
        <th>Chord</th>
        <td>CEG</td>
        <td>GBD</td>
        <td>ACE</td>
        <td>EGB</td>
        <td>FAC</td>
        <td>CEG</td>
        <td>FAC</td>
        <td>GBD</td>
        <td>CEG</td>
    </tr>
</table>

Track 0 是消音，Track 1 是旋律，Track 2 是分解和弦。

```markdown
4D 54 68 64 00 00 00 06 00 01 00 03 00 80

4D 54 72 6B 00 00 00 29

84 00 B0 7B 00
84 00 7B 00
84 00 7B 00
84 00 7B 00
84 00 7B 00
84 00 7B 00
84 00 7B 00
84 00 7B 00
85 00 7B 00

00 FF 2F 00

4D 54 72 6B 00 00 00 28

00 90 40 60
84 00 3E 60
84 00 3C 60
84 00 3B 60
84 00 39 60
84 00 37 60
84 00 39 60
84 00 3B 60
84 00 3C 60

00 FF 2F 00

4D 54 72 6B 00 00 00 70

00 90 30 40
81 00 34 40
81 00 37 40

82 00 2B 40
81 00 2F 40
81 00 32 40

82 00 2D 40
81 00 30 40
81 00 34 40

82 00 28 40
81 00 2B 40
81 00 2F 40

82 00 29 40
81 00 2D 40
81 00 30 40

82 00 24 40
81 00 28 40
81 00 2B 40

82 00 29 40
81 00 2D 40
81 00 30 40

82 00 2B 40
81 00 2F 40
81 00 32 40

82 00 30 40
80 00 34 40
80 00 37 40

00 FF 2F 00
```

细心的同学一定会问，为什么消音音轨要放在最前面，放在最后可以吗？

在这里不可以！！！

**因为同一时刻，事件的执行是从第一个音轨开始的**。
如果把消音放到最后，那么在这个例子中就会出现在某一时刻，
先播放旋律音，再播放分解和弦音，然后再消音，这样，前面的两次播放就会被消掉。

## Homework
-----------

1. 写一小段自己喜欢的音乐。
2. 上面的 Canon 是 1 格式的，尝试把他变成 0 格式。
3. 关于 Track 和 Channel 的区别，哈哈，大家自己 Google XD。

## Reference
------------

1. <a href="http://www.skytopia.com/project/articles/midi.html" target="_blank">http://www.skytopia.com/project/articles/midi.html</a>
2. <a href="http://www.music-software-development.com/midi-tutorial.html" target="_blank">http://www.music-software-development.com/midi-tutorial.html</a>
