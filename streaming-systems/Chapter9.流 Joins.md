# Chapter9.流 Joins

当我第一次开始学习连接时，这是一个令人生畏的话题。 `LEFT`、`OUTER`、`SEMI`、`INNER`、`CROSS`：连接的语言具有表现力和扩展性。再加上流媒体带来的时间维度，剩下的似乎是一个具有挑战性的复杂话题。好消息是，join 确实不像它们最初看起来那样长着令人讨厌的尖牙的可怕野兽。与许多其他复杂主题的情况一样，在您理解了联接的中心思想和主题之后，建立在这些基础知识之上的更广阔的前景突然变得更加容易理解。所以请现在加入我，我们将探讨……好吧，加入这个引人入胜的话题。


### 你所有的加入都属于流

连接两个数据集是什么意思？ 我们直观地理解连接只是一种特定类型的分组操作：通过将共享某些属性（即键）的数据连接在一起，我们将一些以前不相关的单个数据元素收集在一起，形成一组相关元素。 正如我们在 [第 6 章](Chapter6.流和表.md#streams_and_tables) 中了解到的那样，分组操作总是消耗一个流并生成一个表。 知道了这两件事，就可以得出构成整章基础的结论：*在他们心中，所有连接都是流式连接*。

这个事实的伟大之处在于它实际上使流式连接的主题变得更容易处理。 我们在流式分组操作（窗口化、水印、触发器等）的上下文中学习的用于时间推理的所有工具继续适用于流式连接的情况。 也许令人生畏的是，将流媒体添加到组合中似乎只会使事情复杂化。 但是正如您将在后面的示例中看到的那样，将所有连接建模为流式连接具有一定的简洁性和一致性。 很明显，几乎所有类型的连接实际上都归结为同一模式的微小变化，而不是感觉存在大量不同的连接方法。 最后，这种清晰的洞察力有助于使连接（流式或其他方式）变得不那么令人生畏。

为了给我们一些具体的推理，让我们考虑许多不同类型的连接，因为它们应用于以下数据集，方便地命名为“Left”和“Right”以匹配通用命名法：

```
12:10> SELECT TABLE * FROM Left;        12:10> SELECT TABLE * FROM Right;
--------------------                    --------------------
| Num | Id | Time  |                    | Num | Id | Time  |
--------------------                    --------------------
| 1   | L1 | 12:02 |                    | 2   | R2 | 12:01 |
| 2   | L2 | 12:06 |                    | 3   | R3 | 12:04 |
| 3   | L3 | 12:03 |                    | 4   | R4 | 12:05 |
--------------------                    --------------------
```


`每个包含三列：

- Num

	一个单独的数字。

- Id

	相应表名称中第一个字母（“`L`”或“`R`”）和`Num`的组合，从而提供了一种在连接结果中唯一标识给定单元格来源的方法。

- Time

	给定记录在系统中的到达时间，这在考虑流连接时变得很重要。
	
为简单起见，请注意我们的初始数据集将具有严格唯一的连接键。 当我们谈到 `SEMI` 连接时，我们将引入一些更复杂的数据集来突出显示存在重复键时的连接行为。

我们首先深入研究 *unwindowed joins*，因为开窗通常只以很小的方式影响连接语义。 在我们用完了对非窗口连接的胃口之后，我们将触及一些更有趣的连接在窗口上下文中的点。


### 无窗口连接

流式连接无限数据总是需要开窗，这是一个流行的神话。 但是通过应用我们在 [第 6 章](Chapter6.流和表.md#流和表) 中学到的概念，我们可以看出这根本不是真的。 连接（窗口化和非窗口化）只是另一种类型的分组操作，分组操作产生表。 因此，如果我们想将由非窗口连接（或等效地，覆盖所有时间的单个全局窗口内的连接）创建的表作为流使用，我们只需要应用一个非分组（或触发器）操作 “等到我们看到所有输入”的变化。 将连接窗口化为非全局窗口并使用水印触发器（即“等到我们看到流的有限时间块中的所有输入”触发器）确实是一种选择，但在每条记录上触发也是如此（ 即，物化视图语义）或随着处理时间的推进而周期性地进行，而不管连接是否是窗口化的。 因为它使示例易于理解，所以我们假设在以下所有非窗口连接示例中使用隐式默认的每条记录触发器，这些示例将连接结果观察为流。

现在，加入他们自己。 ANSI SQL 定义了五种类型的连接：`FULL OUTER`、`LEFT OUTER`、`RIGHT OUTER`、`INNER` 和 `CROSS`。 我们深入研究前四个，并在下一段中简要讨论最后一个。 我们还谈到了另外两个有趣但不太常见（并且不太支持，至少使用标准语法）的变体：`ANTI` 和 `SEMI` 连接。

从表面上看，这听起来有很多变化。 但正如您将看到的，核心连接实际上只有一种类型：`FULL OUTER`连接。 `CROSS` 连接只是一个带有空洞的 true 连接谓词的 `FULL OUTER` 连接； 也就是说，它返回左表中的一行与右表中的行的所有可能配对。 所有其他连接变体都简单地简化为`FULL OUTER`连接的某个逻辑子集。[1](ch09.html#idm139957174228496)因此，在您了解所有不同连接类型之间的共性后，它就变成了 将它们全部记在脑海中要容易得多。 它还使在流式传输的上下文中对它们的推理变得更加简单。

开始之前的最后一点注意事项：我们将主要考虑最多 1:1 基数的 equi 连接，我的意思是连接谓词是一个相等语句并且每一侧最多有一个匹配行 的加入。 这使示例简单明了。 当我们到达`SEMI`连接时，我们将扩展我们的示例以考虑具有任意 N:M 基数的连接，这将使我们能够观察更多任意谓词连接的行为。

### FULL OUTER

因为它们构成了每个其他变体的概念基础，所以我们首先看一下`FULL OUTER`连接。 外部连接体现了对“连接”一词的相当自由和乐观的解释：`FULL OUTER` 的结果——连接两个数据集本质上是两个数据集中的完整行列表，[2](ch09.html#idm139957174216640) with rows 在共享相同连接键的两个数据集中组合在一起，但任何一方的不匹配行都包括未连接。

例如，如果我们`FULL OUTER`——将我们的两个示例数据集连接到一个仅包含连接 ID 的新关系中，结果将如下所示：

```
12:10> SELECT TABLE 
         Left.Id as L, 
         Right.Id as R,
       FROM Left FULL OUTER JOIN Right
       ON L.Num = R.Num;
---------------
| L    | R    |
---------------
| L1   | null | 
| L2   | R2   |
| L3   | R3   |
| null | R4   |
---------------
```

我们可以看到，`FULL OUTER` 连接包括满足连接谓词的两行（例如，“L2，R2”和“L3，R3”），但它也包括不符合谓词的部分行（例如 ，“`L1，null`”和“`null，R4`”，其中 null 表示数据的未连接部分）。

当然，这只是这个`FULL OUTER`-join 关系的一个时间点快照，是在所有数据都到达系统之后拍摄的。 我们来这里是为了了解流式连接，根据定义，流式连接涉及到额外的时间维度。 正如我们从 [第 8 章](ch08.html#streaming_sql) 中了解到的，如果我们想要了解给定的数据集/关系如何随时间变化，我们想用时变关系 (TVR) 来说话。 因此，为了最好地了解联接如何随时间演变，现在让我们看一下此联接的完整 TVR（每个快照关系之间的变化以黄色突出显示）：

```sql
12:10> SELECT TVR
         Left.Id as L,
         Right.Id as R,
       FROM Left FULL OUTER JOIN Right
       ON L.Num = R.Num;
-------------------------------------------------------------------------
|  [-inf, 12:01)  |  [12:01, 12:02) |  [12:02, 12:03) |  [12:03, 12:04) |
| --------------- | --------------- | --------------- | --------------- |
| | L    | R    | | | L    | R    | | | L    | R    | | | L    | R    | |
| --------------- | --------------- | --------------- | --------------- |
| --------------- | | null | R2   | | | L1   | null | | | L1   | null | | 
|                 | --------------- | | null | R2   | | | null | R2   | |
|                 |                 | --------------- | | L3   | null | |
|                 |                 |                 | --------------- |
-------------------------------------------------------------------------
|  [12:04, 12:05) |  [12:05, 12:06) |  [12:06, 12:07) |
| --------------- | --------------- | --------------- |
| | L    | R    | | | L    | R    | | | L    | R    | |
| --------------- | --------------- | --------------- |
| | L1   | null | | | L1   | null | | | L1   | null | |
| | null | L2   | | | null | L2   | | | L2   | L2   | |
| | L3   | L3   | | | L3   | L3   | | | L3   | L3   | |
| --------------- | | null | L4   | | | null | L4   | |
|                 | --------------- | --------------- |
-------------------------------------------------------
```


而且，正如您随后可能期望的那样，此 TVR 的流渲染将捕获每个快照之间的特定增量：

```

12:00> SELECT STREAM 
         Left.Id as L,
         Right.Id as R, 
         CURRENT_TIMESTAMP as Time,
         Sys.Undo as Undo
       FROM Left FULL OUTER JOIN Right
       ON L.Num = R.Num;
------------------------------
| L    | R    | Time  | Undo |
------------------------------
| null | R2   | 12:01 |      |
| L1   | null | 12:02 |      |
| L3   | null | 12:03 |      |
| L3   | null | 12:04 | undo |
| L3   | R3   | 12:04 |      |
| null | R4   | 12:05 |      |
| null | R2   | 12:06 | undo |
| L2   | R2   | 12:06 |      |
....... [12:00, 12:10] .......
```



请注意包含 Time 和 Undo 列，以突出显示给定行在流中具体化的时间，并在对给定行的更新首先导致撤回该行的先前版本时调用实例。 如果此流要随时间捕获 TVR 的全保真视图，则撤消/撤回行至关重要。

因此，尽管连接的这三种渲染（表、TVR、流）中的每一种都彼此不同，但很明显它们只是对同一数据的不同视图：表快照向我们展示了整个数据集 因为它存在于所有数据到达之后，并且 TVR 和流版本捕获（以它们自己的方式）整个关系在其存在过程中的演变。

通过对 `FULL OUTER` 连接的基本熟悉，我们现在了解流上下文中连接的所有核心概念。 不需要窗口，没有自定义触发器，没有什么特别痛苦或不直观的。 正如您所期望的那样，随着时间的推移，连接的每条记录都在演变。 更好的是，所有其他类型的连接都只是这个主题的变体（至少在概念上），本质上只是对`FULL OUTER`连接的每条记录流执行的额外过滤操作。 现在让我们更详细地看看它们中的每一个。

### LEFT OUTER

`LEFT OUTER` 连接只是一个 `FULL OUTER` 连接，删除了右侧数据集中任何未连接的行。 通过采用原始的“FULL OUTER”连接并将要过滤的行灰显，可以最清楚地看到这一点。 对于“LEFT OUTER”连接，它看起来像下面这样，其中左侧未连接的每一行都从原始“FULL OUTER”连接中过滤掉：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left LEFT OUTER JOIN Right          FROM Left LEFT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
```

要查看表和流在实践中的实际情况，让我们再次查看相同的查询，但这次完全省略了灰色行：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left LEFT OUTER JOIN Right          FROM Left LEFT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | L1   | null | 12:02 |      |
| L2   | R2   |                          | L3   | null | 12:03 |      |
| L3   | R3   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
```





### RIGHT OUTER

`RIGHT OUTER` 连接与左连接相反：在完全外连接中，来自左侧数据集的所有未连接的行都被右出，*咳嗽*，删除：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left RIGHT OUTER JOIN Right         FROM Left RIGHT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
```

在这里，我们看到呈现为实际`RIGHT OUTER`连接的查询将如何出现：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left RIGHT OUTER JOIN Right         FROM Left RIGHT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L2   | R2   |                          | null | R2   | 12:01 |      |
| L3   | R3   |                          | L3   | R3   | 12:04 |      |
| null | R4   |                          | null | R4   | 12:05 |      |
---------------                          | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
```





### INNER

`INNER` 联接本质上是 `LEFT OUTER` 和 `RIGHT OUTER` 联接的交集。 或者，从减法的角度考虑，从原始`FULL OUTER`连接中删除以创建`INNER`连接的行是从`LEFT OUTER`和`RIGHT OUTER`连接中删除的行的并集。 因此，`INNER`连接中不存在任一侧未连接的所有行：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left INNER JOIN Right               FROM Left INNER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
```

再一次，更简洁地呈现为 `INNER` 连接在现实中的样子：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left INNER JOIN Right               FROM Left INNER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L2   | R2   |                          | L3   | R3   | 12:04 |      |
| L3   | R3   |                          | L2   | R2   | 12:06 |      |
---------------                          ....... [12:00, 12:10] .......
```



### ANTI

`ANTI` 连接是 `INNER` 连接的反面：它们包含所有 *unjoined* 行。 并非所有 SQL 系统都支持干净的`ANTI`连接语法，但为了清楚起见，我将在此处使用最直接的语法：

```
                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left ANTI JOIN Right                FROM Left ANTI JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          -------------------------------
| L    | R    |                          | L    |    R | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
```

`ANTI` 连接的流渲染有点有趣的是，它最终包含一堆错误的开始和最终确实连接的行的撤回； 事实上，`ANTI`join 和 `INNER` join 一样轻量级的撤回。 更简洁的版本看起来像这样：
```
                                        12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left ANTI JOIN Right               FROM Left ANTI JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| null | R4   |                          | L1   | null | 12:02 |      |
---------------                          | L3   | null | 12:03 |      | 
                                         | L3   | null | 12:04 | undo |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         ....... [12:00, 12:10] .......
```



### SEMI

我们现在来到 `SEMI` 连接，而 `SEMI` 连接有点奇怪。 乍一看，它们基本上看起来像内部联接，其中联接值的一侧被删除了。 而且，事实上，在 `:` 的基数关系是 N:M 且 M ≤ 1 的情况下，这是可行的（请注意，我们将使用 keep=`Left`, dropped=`Right` 用于所有示例 跟随）。 例如，在我们目前使用的`Left`和`Right`数据集（连接数据的基数为 0:1、1:0 和 1:1）上，`INNER`和`SEMI` 连接变体看起来相同：

```
12:10> SELECT TABLE            12:10> SELECT TABLE
  Left.Id as L                   Left.Id as L
FROM Left INNER JOIN           FROM Left SEMI JOIN
Right ON L.Num = R.Num;        Right ON L.Num = R.Num;
---------------                ---------------
| L    | R    |                | L    | R    |
---------------                ---------------
| L1   | null |                | L1   | null |
| L2   | R2   |                | L2   | R2   |
| L3   | R3   |                | L3   | R3   |
| null | R4   |                | null | R4   |
---------------                ---------------
```

然而，在 M > 1 的 N:M 基数的情况下，`SEMI`连接还有一个额外的微妙之处：因为 M 侧的 *values* 没有被返回，`SEMI`连接简单地预测那里的连接条件 是右侧的*任何*匹配行，而不是为*每个*匹配行重复产生新结果。

为了清楚地看到这一点，让我们切换到一对稍微复杂的输入关系，突出显示其中包含的行的 N:M 连接基数。 在这些关系中，`N_M`列说明左右两边之间行的基数关系，`Id`列（如前所述）为每个输入关系中的每一行提供唯一的标识符：

```
12:15> SELECT TABLE * FROM LeftNM;    12:15> SELECT TABLE * FROM RightNM;
---------------------                 ---------------------
| N_M | Id  |  Time |                 | N_M | Id  |  Time |
---------------------                 ---------------------
| 1:0 | L2  | 12:07 |                 | 0:1 | R1  | 12:02 |
| 1:1 | L3  | 12:01 |                 | 1:1 | R3  | 12:14 |
| 1:2 | L4  | 12:05 |                 | 1:2 | R4A | 12:03 |
| 2:1 | L5A | 12:09 |                 | 1:2 | R4B | 12:04 |
| 2:1 | L5B | 12:08 |                 | 2:1 | R5  | 12:06 |
| 2:2 | L6A | 12:12 |                 | 2:2 | R6A | 12:11 |
| 2:2 | L6B | 12:10 |                 | 2:2 | R6B | 12:13 |
---------------------                 ---------------------
```



使用这些输入，FULL OUTER 联接扩展为如下所示：

```
                                       12:00> SELECT STREAM
                                                COALESCE(LeftNM.N_M, 
12:15> SELECT TABLE                                      RightNM.N_M) as N_M, 
         COALESCE(LeftNM.N_M,                     LeftNM.Id as L,
                  RightNM.N_M) as N_M,            RightNM.Id as R, 
         LeftNM.Id as L,                        Sys.EmitTime as Time, 
         RightNM.Id as R,                         Sys.Undo as Undo
       FROM LeftNM                            FROM LeftNM 
         FULL OUTER JOIN RightNM                FULL OUTER JOIN RightNM
         ON LeftNM.N_M = RightNM.N_M;           ON LeftNM.N_M = RightNM.N_M;
---------------------                  ------------------------------------
| N_M | L    | R    |                  | N_M | L    | R    | Time  | Undo |
---------------------                  ------------------------------------
| 0:1 | null | R1   |                  | 1:1 | L3   | null | 12:01 |      |
| 1:0 | L2   | null |                  | 0:1 | null | R1   | 12:02 |      |
| 1:1 | L3   | R3   |                  | 1:2 | null | R4A  | 12:03 |      |
| 1:2 | L4   | R4A  |                  | 1:2 | null | R4B  | 12:04 |      |
| 1:2 | L4   | R4B  |                  | 1:2 | null | R4A  | 12:05 | undo |
| 2:1 | L5A  | R5   |                  | 1:2 | null | R4B  | 12:05 | undo |
| 2:1 | L5B  | R5   |                  | 1:2 | L4   | R4A  | 12:05 |      |
| 2:2 | L6A  | R6A  |                  | 1:2 | L4   | R4B  | 12:05 |      |
| 2:2 | L6A  | R6B  |                  | 2:1 | null | R5   | 12:06 |      |
| 2:2 | L6B  | R6A  |                  | 1:0 | L2   | null | 12:07 |      |
| 2:2 | L6B  | R6B  |                  | 2:1 | null | R5   | 12:08 | undo |
---------------------                  | 2:1 | L5B  | R5   | 12:08 |      |
                                       | 2:1 | L5A  | R5   | 12:09 |      |
                                       | 2:2 | L6B  | null | 12:10 |      |
                                       | 2:2 | L6B  | null | 12:11 | undo |
                                       | 2:2 | L6B  | R6A  | 12:11 |      |
                                       | 2:2 | L6A  | R6A  | 12:12 |      |
                                       | 2:2 | L6A  | R6B  | 12:13 |      |
                                       | 2:2 | L6B  | R6B  | 12:13 |      |
                                       | 1:1 | L3   | null | 12:14 | undo |
                                       | 1:1 | L3   | R3   | 12:14 |      |
                                       .......... [12:00, 12:15] ..........
```



作为旁注，这些更复杂的数据集的一个额外好处是，当每一侧有多行匹配同一谓词时，连接的乘法性质开始变得更加清晰（例如，“2:2”行，它展开 从每个输入中的两行到输出中的四行；如果数据集有一组“3:3”行，它们将从每个输入中的三行扩展到输出中的九行，依此类推 ).

但是回到“SEMI”连接的微妙之处。 使用这些数据集，过滤后的`INNER`连接和`SEMI`连接之间的区别变得更加清晰：`INNER`连接为 N:M 基数 M > 1 的任何行产生重复值， 而 `SEMI` 连接没有（请注意，我在 `INNER` 连接版本中用红色突出显示了重复的行，并以灰色包含了在相应的 `INNER` 中省略的完整外部连接部分和 `SEMI` 版本）：

```sql
12:15> SELECT TABLE                       12:15> SELECT TABLE
         COALESCE(LeftNM.N_M,                      COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                      RightNM.N_M) as N_M,
         LeftNM.Id as L                            LeftNM.Id as L
       FROM LeftNM INNER JOIN RightNM            FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;              ON LeftNM.N_M = RightNM.N_M;
---------------------                     ---------------------
| N_M | L    | R    |                     | N_M | L    | R    |
---------------------                     ---------------------
| 0:1 | null | R1   |                     | 0:1 | null | R1   |
| 1:0 | L2   | null |                     | 1:0 | L2   | null |
| 1:1 | L3   | R3   |                     | 1:1 | L3   | R3   |
| 1:2 | L4   | R5A  |                     | 1:2 | L4   | R5A  |
| 1:2 | L4   | R5B  |                     | 1:2 | L4   | R5B  |
| 2:1 | L5A  | R5   |                     | 2:1 | L5A  | R5   |
| 2:1 | L5B  | R5   |                     | 2:1 | L5B  | R5   |
| 2:2 | L6A  | R6A  |                     | 2:2 | L6A  | R6A  |
| 2:2 | L6A  | R6B  |                     | 2:2 | L6A  | R6B  |
| 2:2 | L6B  | R6A  |                     | 2:2 | L6B  | R6A  |
| 2:2 | L6B  | R6B  |                     | 2:2 | L6B  | R6B  |
---------------------                     ---------------------
```

或者，更简洁地呈现：

```
12:15> SELECT TABLE                       12:15> SELECT TABLE
         COALESCE(LeftNM.N_M,                      COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                      RightNM.N_M) as N_M,
         LeftNM.Id as L                            LeftNM.Id as L
       FROM LeftNM INNER JOIN RightNM            FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;              ON LeftNM.N_M = RightNM.N_M;
-------------                             -------------
| N_M | L   |                             | N_M | L   |
-------------                             -------------
| 1:1 | L3  |                             | 1:1 | L3  |
| 1:2 | L4  |                             | 1:2 | L4  |
| 1:2 | L4  |                             | 2:1 | L5A |
| 2:1 | L5A |                             | 2:1 | L5B |
| 2:1 | L5B |                             | 2:2 | L6A |
| 2:2 | L6A |                             | 2:2 | L6B |
| 2:2 | L6A |                             -------------
| 2:2 | L6B |
| 2:2 | L6B |
-------------
```

然后，`STREAM`渲染提供了一些关于哪些行被过滤掉的上下文——它们只是后来到达的重复行（从被投影的列的角度来看）：

```
12:00> SELECT STREAM                        12:00> SELECT STREAM
         COALESCE(LeftNM.N_M,                        COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                        RightNM.N_M) as N_M,
         LeftNM.Id as L                              LeftNM.Id as L
         Sys.EmitTime as Time,                       Sys.EmitTime as Time,
         Sys.Undo as Undo,                           Sys.Undo as Undo,
       FROM LeftNM INNER JOIN RightNM              FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;                ON LeftNM.N_M = RightNM.N_M;
------------------------------------        ------------------------------------
| N_M | L    | R    | Time  | Undo |        | N_M | L    | R    | Time  | Undo |
------------------------------------        ------------------------------------
| 1:1 | L3   | null | 12:01 |      |        | 1:1 | L3   | null | 12:01 |      |
| 0:1 | null | R1   | 12:02 |      |        | 0:1 | null | R1   | 12:02 |      |
| 1:2 | null | R4A  | 12:03 |      |        | 1:2 | null | R4A  | 12:03 |      |
| 1:2 | null | R4B  | 12:04 |      |        | 1:2 | null | R4B  | 12:04 |      |
| 1:2 | null | R4A  | 12:05 | undo |        | 1:2 | null | R4A  | 12:05 | undo |
| 1:2 | null | R4B  | 12:05 | undo |        | 1:2 | null | R4B  | 12:05 | undo |
| 1:2 | L4   | R4A  | 12:05 |      |        | 1:2 | L4   | R4A  | 12:05 |      |
| 1:2 | L4   | R4B  | 12:05 |      |        | 1:2 | L4   | R4B  | 12:05 |      |
| 2:1 | null | R5   | 12:06 |      |        | 2:1 | null | R5   | 12:06 |      |
| 1:0 | L2   | null | 12:07 |      |        | 1:0 | L2   | null | 12:07 |      |
| 2:1 | null | R5   | 12:08 | undo |        | 2:1 | null | R5   | 12:08 | undo |
| 2:1 | L5B  | R5   | 12:08 |      |        | 2:1 | L5B  | R5   | 12:08 |      |
| 2:1 | L5A  | R5   | 12:09 |      |        | 2:1 | L5A  | R5   | 12:09 |      |
| 2:2 | L6B  | null | 12:10 |      |        | 2:2 | L6B  | null | 12:10 |      |
| 2:2 | L6B  | null | 12:10 | undo |        | 2:2 | L6B  | null | 12:10 | undo |
| 2:2 | L6B  | R6A  | 12:11 |      |        | 2:2 | L6B  | R6A  | 12:11 |      |
| 2:2 | L6A  | R6A  | 12:12 |      |        | 2:2 | L6A  | R6A  | 12:12 |      |
| 2:2 | L6A  | R6B  | 12:13 |      |        | 2:2 | L6A  | R6B  | 12:13 |      |
| 2:2 | L6B  | R6B  | 12:13 |      |        | 2:2 | L6B  | R6B  | 12:13 |      |
| 1:1 | L3   | null | 12:14 | undo |        | 1:1 | L3   | null | 12:14 | undo |
| 1:1 | L3   | R3   | 12:14 |      |        | 1:1 | L3   | R3   | 12:14 |      |
.......... [12:00, 12:15] ..........        .......... [12:00, 12:15] ..........
```

再一次，简洁地呈现：

```
12:00> SELECT STREAM                        12:00> SELECT STREAM
         COALESCE(LeftNM.N_M,                        COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                        RightNM.N_M) as N_M,
         LeftNM.Id as L                              LeftNM.Id as L
         Sys.EmitTime as Time,                       Sys.EmitTime as Time,
         Sys.Undo as Undo,                           Sys.Undo as Undo,
       FROM LeftNM INNER JOIN RightNM              FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;                ON LeftNM.N_M = RightNM.N_M;
----------------------------                ----------------------------
| N_M | L   | Time  | Undo |                | N_M | L   | Time  | Undo |
----------------------------                ----------------------------
| 1:2 | L4  | 12:05 |      |                | 1:2 | L4  | 12:05 |      |
| 1:2 | L4  | 12:05 |      |                | 2:1 | L5B | 12:08 |      |
| 2:1 | L5B | 12:08 |      |                | 2:1 | L5A | 12:09 |      |
| 2:1 | L5A | 12:09 |      |                | 2:2 | L6B | 12:11 |      |
| 2:2 | L6B | 12:11 |      |                | 2:2 | L6A | 12:12 |      |
| 2:2 | L6A | 12:12 |      |                | 1:1 | L3  | 12:14 |      |
| 2:2 | L6A | 12:13 |      |                ...... [12:00, 12:15] ......
| 2:2 | L6B | 12:13 |      |
| 1:1 | L3  | 12:14 |      |
...... [12:00, 12:15] ......      
```



`***12:15> SELECT TABLE                       12:15> SELECT TABLE\***         ***COALESCE(LeftNM.N_M,                      COALESCE(LeftNM.N_M,\***                  ***RightNM.N_M) as N_M,                      RightNM.N_M) as N_M,\***         ***LeftNM.Id as L                            LeftNM.Id as L\***       ***FROM LeftNM INNER JOIN RightNM            FROM LeftNM SEMI JOIN RightNM\***       ***ON LeftNM.N_M = RightNM.N_M;              ON LeftNM.N_M = RightNM.N_M;\*** ---------------------                     --------------------- | N_M | L    | R    |                     | N_M | L    | R    | ---------------------                     --------------------- | 0:1 | null | R1   |                     | 0:1 | null | R1   | | 1:0 | L2   | null |                     | 1:0 | L2   | null | | 1:1 | L3   | R3   |                     | 1:1 | L3   | R3   | | 1:2 | L4   | R5A  |                     | 1:2 | L4   | R5A  | | 1:2 | L4   | R5B  |                     | 1:2 | L4   | R5B  | | 2:1 | L5A  | R5   |                     | 2:1 | L5A  | R5   | | 2:1 | L5B  | R5   |                     | 2:1 | L5B  | R5   | | 2:2 | L6A  | R6A  |                     | 2:2 | L6A  | R6A  | | 2:2 | L6A  | R6B  |                     | 2:2 | L6A  | R6B  | | 2:2 | L6B  | R6A  |                     | 2:2 | L6B  | R6A  | | 2:2 | L6B  | R6B  |                     | 2:2 | L6B  | R6B  | ---------------------                     --------------------- `Or, rendered more succinctly:`***12:15> SELECT TABLE                       12:15> SELECT TABLE\***         ***COALESCE(LeftNM.N_M,                      COALESCE(LeftNM.N_M,\***                  ***RightNM.N_M) as N_M,                      RightNM.N_M) as N_M,\***         ***LeftNM.Id as L                            LeftNM.Id as L\***       ***FROM LeftNM INNER JOIN RightNM            FROM LeftNM SEMI JOIN RightNM\***       ***ON LeftNM.N_M = RightNM.N_M;              ON LeftNM.N_M = RightNM.N_M;\*** -------------                             ------------- | N_M | L   |                             | N_M | L   | -------------                             ------------- | 1:1 | L3  |                             | 1:1 | L3  | | 1:2 | L4  |                             | 1:2 | L4  | | 1:2 | L4  |                             | 2:1 | L5A | | 2:1 | L5A |                             | 2:1 | L5B | | 2:1 | L5B |                             | 2:2 | L6A | | 2:2 | L6A |                             | 2:2 | L6B | | 2:2 | L6A |                             ------------- | 2:2 | L6B | | 2:2 | L6B | ------------- `The `STREAM` renderings then provide a bit of context as to which rows are filtered out—they are simply the later-arriving duplicate rows (from the perspective of the columns being projected):`***12:00> SELECT STREAM                        12:00> SELECT STREAM\***         ***COALESCE(LeftNM.N_M,                        COALESCE(LeftNM.N_M,\***                  ***RightNM.N_M) as N_M,                        RightNM.N_M) as N_M,\***         ***LeftNM.Id as L                              LeftNM.Id as L\***         ***Sys.EmitTime as Time,                       Sys.EmitTime as Time,\***         ***Sys.Undo as Undo,                           Sys.Undo as Undo,\***       ***FROM LeftNM INNER JOIN RightNM              FROM LeftNM SEMI JOIN RightNM\***       ***ON LeftNM.N_M = RightNM.N_M;                ON LeftNM.N_M = RightNM.N_M;\*** ------------------------------------        ------------------------------------ | N_M | L    | R    | Time  | Undo |        | N_M | L    | R    | Time  | Undo | ------------------------------------        ------------------------------------ | 1:1 | L3   | null | 12:01 |      |        | 1:1 | L3   | null | 12:01 |      | | 0:1 | null | R1   | 12:02 |      |        | 0:1 | null | R1   | 12:02 |      | | 1:2 | null | R4A  | 12:03 |      |        | 1:2 | null | R4A  | 12:03 |      | | 1:2 | null | R4B  | 12:04 |      |        | 1:2 | null | R4B  | 12:04 |      | | 1:2 | null | R4A  | 12:05 | undo |        | 1:2 | null | R4A  | 12:05 | undo | | 1:2 | null | R4B  | 12:05 | undo |        | 1:2 | null | R4B  | 12:05 | undo | | 1:2 | L4   | R4A  | 12:05 |      |        | 1:2 | L4   | R4A  | 12:05 |      | | 1:2 | L4   | R4B  | 12:05 |      |        | 1:2 | L4   | R4B  | 12:05 |      | | 2:1 | null | R5   | 12:06 |      |        | 2:1 | null | R5   | 12:06 |      | | 1:0 | L2   | null | 12:07 |      |        | 1:0 | L2   | null | 12:07 |      | | 2:1 | null | R5   | 12:08 | undo |        | 2:1 | null | R5   | 12:08 | undo | | 2:1 | L5B  | R5   | 12:08 |      |        | 2:1 | L5B  | R5   | 12:08 |      | | 2:1 | L5A  | R5   | 12:09 |      |        | 2:1 | L5A  | R5   | 12:09 |      | | 2:2 | L6B  | null | 12:10 |      |        | 2:2 | L6B  | null | 12:10 |      | | 2:2 | L6B  | null | 12:10 | undo |        | 2:2 | L6B  | null | 12:10 | undo | | 2:2 | L6B  | R6A  | 12:11 |      |        | 2:2 | L6B  | R6A  | 12:11 |      | | 2:2 | L6A  | R6A  | 12:12 |      |        | 2:2 | L6A  | R6A  | 12:12 |      | | 2:2 | L6A  | R6B  | 12:13 |      |        | 2:2 | L6A  | R6B  | 12:13 |      | | 2:2 | L6B  | R6B  | 12:13 |      |        | 2:2 | L6B  | R6B  | 12:13 |      | | 1:1 | L3   | null | 12:14 | undo |        | 1:1 | L3   | null | 12:14 | undo | | 1:1 | L3   | R3   | 12:14 |      |        | 1:1 | L3   | R3   | 12:14 |      | .......... [12:00, 12:15] ..........       .......... [12:00, 12:15] .......... `And again, rendered succinctly:`***12:00> SELECT STREAM                        12:00> SELECT STREAM\***         ***COALESCE(LeftNM.N_M,                        COALESCE(LeftNM.N_M,\***                  ***RightNM.N_M) as N_M,                        RightNM.N_M) as N_M,\***         ***LeftNM.Id as L                              LeftNM.Id as L\***         ***Sys.EmitTime 

正如我们在许多示例过程中看到的那样，流连接确实没有什么特别之处。 考虑到我们对流和表的了解，它们的功能完全符合我们的预期，连接流会随着时间的推移捕获连接的历史记录。 这与连接表形成对比，连接表只是捕获特定时间点存在的整个连接的快照，因为我们可能更习惯了。

但是，更重要的是，通过流表理论的视角来观察连接已经更加清晰了。 核心底层连接原语是`FULL OUTER`连接，它是一个流→表分组操作，将关系中所有连接和未连接的行收集在一起。 我们详细查看的所有其他变体（`LEFT OUTER`、`RIGHT OUTER`、`INNER`、`ANTI` 和 `SEMI`）只是在 `FULL OUTER` 之后的连接流上添加额外的过滤层加入。[3](ch09.html#idm139957173841152)

# Windowed Joins

看过各种非开窗连接后，让我们接下来探索开窗添加到组合中的内容。 我认为对连接进行窗口化有两个动机：

- 以某种有意义的方式划分时间

   一个明显的例子是固定窗户； 例如，每日窗口，出于某些业务原因（例如，每日账单记录），应将同一天发生的事件连接在一起。 另一个可能出于性能原因限制连接内的时间范围。 然而，事实证明，在联接中有更复杂（和有用）的时间分区方法，包括一个特别有趣的用例，据我所知，目前没有任何流媒体系统原生支持：*时间有效性联接*。 稍后会详细介绍这一点。


- 为连接超时提供有意义的参考点

   这对于许多无界连接情况很有用，但它可能最明显地适用于外部连接等用例，因为连接的一侧是否会出现是先验未知的。 对于经典的批处理（包括标准的交互式 SQL 查询），仅当有界输入数据集已被完全处理时，外连接才会超时。 但是在处理无界数据时，我们不能等到所有数据都处理完。 正如我们在第 [2](ch02.html#the_what_where_when_and_how)和 [3](ch03.html#watermarks_chapter)章节中讨论的那样，水印提供了一个进度指标，用于衡量输入源在事件时间内的完整性。 但是要使用该指标来使连接超时，我们需要一些参考点来进行比较。 通过将连接的范围限制在窗口的末尾，对连接进行窗口化提供了该引用。 在水印通过窗口的末端之后，系统可以认为窗口的输入完成。 在那一点上，就像在有界连接的情况下一样，让任何未连接的行超时并具体化它们的部分结果是安全的。


也就是说，正如我们之前看到的，开窗绝对不是流连接的必要条件。 在很多情况下这很有意义，但绝不是必需的。

在实践中，窗口连接（例如每日窗口）的大多数用例都相对简单，并且很容易从我们到目前为止学到的概念中推断出来。 为了了解原因，我们简要地看一下将固定窗口应用于我们已经遇到的一些连接示例意味着什么。 之后，我们将在本章的剩余部分研究*时间有效性连接*这个更有趣（和令人费解）的主题，首先详细了解我所说的时间有效性窗口的含义，然后继续研究什么 在此类窗口的上下文中加入意味着。

### Fixed Windows

窗口连接将时间维度添加到连接标准本身中。 这样做时，窗口用于将要连接的行集的范围限定为仅包含在窗口时间间隔内的行。 举个例子可能会更清楚地看到这一点，所以让我们把原来的`Left`表和`Right`表放在五分钟的固定窗口中：

```sql
12:10> SELECT TABLE *,                     12:10> SELECT TABLE *,
       TUMBLE(Time, INTERVAL '5' MINUTE)          TUMBLE(Time, INTERVAL '5' MINUTE)
       as Window FROM Left;                       as Window FROM Right
-------------------------------------      -------------------------------------
| Num | Id | Time  | Window         |      | Num | Id | Time  | Window         |
-------------------------------------      -------------------------------------
| 1   | L1 | 12:02 | [12:00, 12:05) |      | 2   | R2 | 12:01 | [12:00, 12:05) |
| 2   | L2 | 12:06 | [12:05, 12:10) |      | 3   | R3 | 12:04 | [12:00, 12:05) |
| 3   | L3 | 12:03 | [12:00, 12:05) |      | 4   | R4 | 12:05 | [12:05, 12:06) |
-------------------------------------      -------------------------------------
```

在我们之前的`Left`和`Right`示例中，连接条件只是`Left.Num = Right.Num`。 要将其转换为窗口连接，我们将扩展连接条件以包括窗口相等性，以及：`Left.Num = Right.Num AND Left.Window = Right.Window`。 知道这一点，我们已经可以从前面的窗口表中推断出我们的连接将如何变化（为清楚起见突出显示）：因为“L2”和“R2”行不在同一个五分钟固定窗口内，它们不会 在我们的连接的窗口变体中连接在一起。事实上，如果我们将非窗口和窗口变体作为表格并排比较，我们可以清楚地看到这一点（相应的“L2”和“R2”行突出显示在每个 连接的一侧）：

通过将连接的时间区域限定为固定的五分钟间隔，我们将数据集分成两个不同的时间窗口：`[12:00, 12:05)`和`[12:05, 12:10)` . 然后将我们之前观察到的完全相同的连接逻辑应用于这些区域，对于`L2`和`R2`行落入不同区域的情况，会产生略有不同的结果。 在基本层面上，这就是窗口连接的全部内容。

```
                                 12:10> SELECT TABLE 
                                          Left.Id as L,
                                          Right.Id as R,
                                          COALESCE(
                                            TUMBLE(Left.Time, INTERVAL '5' MINUTE),
                                            TUMBLE(Right.Time, INTERVAL '5' MINUTE)
12:10> SELECT TABLE                       ) AS Window
         Left.Id as L,                  FROM Left
         Right.Id as R,                   FULL OUTER JOIN Right 
       FROM Left                          ON L.Num = R.Num AND 
         FULL OUTER JOIN Right              TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
         ON L.Num = R.Num;                  TUMBLE(Right.Time, INTERVAL '5' MINUTE);
---------------                  --------------------------------
| L    | R    |                  | L    | R    | Window         |
---------------                  --------------------------------
| L1   | null |                  | L1   | null | [12:00, 12:05) |
| L2   | R2   |                  | null | R2   | [12:00, 12:05) |
| L3   | R3   |                  | L3   | R3   | [12:00, 12:05) |
| null | R4   |                  | L2   | null | [12:05, 12:10) |
---------------                  | null | R4   | [12:05, 12:10) |
                                 --------------------------------
```



将非窗口连接和窗口连接作为流进行比较时，差异也很明显。 正如我在下面的示例中强调的那样，它们的主要区别在于最后几行。 未开窗的一侧完成了`Num = 2`的连接，除了为已完成的`L2，R2`连接生成一个新行之外，还为未连接的`R2`行产生了一个缩回。 另一方面，有窗口的一侧只是产生一个未连接的 `L2` 行，因为 `L2` 和 `R2` 落在不同的五分钟窗口内：

```
                                 12:10> SELECT STREAM 
                                          Left.Id as L,
                                          Right.Id as R,
                                          Sys.EmitTime as Time,
                                          COALESCE(
                                            TUMBLE(Left.Time, INTERVAL '5' MINUTE),
12:10> SELECT STREAM                        TUMBLE(Right.Time, INTERVAL '5' MINUTE)
         Left.Id as L,                    ) AS Window,
         Right.Id as R,                 Sys.Undo as Undo
         Sys.EmitTime as Time,          FROM Left
         Sys.Undo as Undo                 FULL OUTER JOIN Right
       FROM Left                          ON L.Num = R.Num AND
         FULL OUTER JOIN Right              TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
         ON L.Num = R.Num;                  TUMBLE(Right.Time, INTERVAL '5' MINUTE);
------------------------------   -----------------------------------------------
| L    | R    | Time  | Undo |   | L    | R    | Time  | Window         | Undo |
------------------------------   -----------------------------------------------
| null | R2   | 12:01 |      |   | null | R2   | 12:01 | [12:00, 12:05) |      |
| L1   | null | 12:02 |      |   | L1   | null | 12:02 | [12:00, 12:05) |      |
| L3   | null | 12:03 |      |   | L3   | null | 12:03 | [12:00, 12:05) |      |
| L3   | null | 12:04 | undo |   | L3   | null | 12:04 | [12:00, 12:05) | undo |
| L3   | R3   | 12:04 |      |   | L3   | R3   | 12:04 | [12:00, 12:05) |      |
| null | R4   | 12:05 |      |   | null | R4   | 12:05 | [12:05, 12:10) |      |
| null | R2   | 12:06 | undo |   | L2   | null | 12:06 | [12:05, 12:10) |      |
| L2   | R2   | 12:06 |      |   ............... [12:00, 12:10] ................
....... [12:00, 12:10] .......
```


至此，我们现在了解了开窗对`FULL OUTER`连接的影响。 通过应用我们在本章前半部分学到的规则，很容易推导出`LEFT OUTER`、`RIGHT OUTER`、`INNER`、`ANTI`和`SEMI`连接的窗口变体。 我将把这些推导中的大部分作为练习留给你来完成，但举一个例子，我们了解到，`LEFT OUTER` 连接只是连接左侧带有空列的 `FULL OUTER` 连接 删除（再次突出显示`L2`和`R2`行以比较差异）：

```
                                 12:10> SELECT TABLE 
                                          Left.Id as L,
                                          Right.Id as R,
                                          COALESCE(
                                            TUMBLE(Left.Time, INTERVAL '5' MINUTE),
                                            TUMBLE(Right.Time, INTERVAL '5' MINUTE)
12:10> SELECT TABLE                       ) AS Window
         Left.Id as L,                  FROM Left
         Right.Id as R,                   LEFT OUTER JOIN Right 
       FROM Left                          ON L.Num = R.Num AND 
         LEFT OUTER JOIN Right              TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
         ON L.Num = R.Num;                  TUMBLE(Right.Time, INTERVAL '5' MINUTE);
---------------                  --------------------------------
| L    | R    |                  | L    | R    | Window         |
---------------                  --------------------------------
| L1   | null |                  | L1   | null | [12:00, 12:05) |
| L2   | R2   |                  | L2   | null | [12:05, 12:10) |
| L3   | R3   |                  | L3   | R3   | [12:00, 12:05) |
---------------                  --------------------------------
```



通过将连接的时间区域限定为固定的五分钟间隔，我们将数据集分成两个不同的时间窗口：`[12:00, 12:05)`和`[12:05, 12:10)` . 然后将我们之前观察到的完全相同的连接逻辑应用于这些区域，对于`L2`和`R2`行落入不同区域的情况，会产生略有不同的结果。 在基本层面上，这就是窗口连接的全部内容。



### 时间有效性

了解了窗口连接的基础知识后，我们现在使用高级方法：时间有效性窗口。

### 时间有效性窗口

时间有效性窗口适用于关系中的行有效地将时间划分为给定值有效的区域的情况。 更具体地说，想象一个执行货币转换的金融系统。[4](ch09.html#idm139957173727936) 这样的系统可能包含一个时变关系，用于捕获各种类型货币的当前转换率。 例如，可能存在从不同货币转换为日元的关系，如下所示：

```
12:10> SELECT TABLE * FROM YenRates;
--------------------------------------
| Curr | Rate | EventTime | ProcTime |
--------------------------------------
| USD  | 102  | 12:00:00  | 12:04:13 |
| Euro | 114  | 12:00:30  | 12:06:23 |
| Yen  | 1    | 12:01:00  | 12:05:18 |
| Euro | 116  | 12:03:00  | 12:09:07 |
| Euro | 119  | 12:06:00  | 12:07:33 |
--------------------------------------
```

为了强调我所说的时间有效性窗口“有效地将时间划分为给定值有效的区域”的意思，请仅考虑该关系中的欧元兑日元汇率：

```
12:10> SELECT TABLE * FROM YenRates WHERE Curr = "Euro";
--------------------------------------
| Curr | Rate | EventTime | ProcTime |
--------------------------------------
| Euro | 114  | 12:00:30  | 12:06:23 |
| Euro | 116  | 12:03:00  | 12:09:07 |
| Euro | 119  | 12:06:00  | 12:07:33 |
--------------------------------------
```

从数据库工程的角度来看，我们理解这些值并不意味着将欧元转换为日元的汇率恰好在 12:00 时为 114 ¥/€，在 12:03 时为 116 ¥/€，在 12 时为 119 ¥/€： 06，其他时间未定义。 相反，我们知道此表的目的是捕捉欧元到日元的兑换率在 12:00 之前未定义的事实，从 12:00 到 12:03 为 114 ¥/€，从 12 开始为 116 ¥/€： 03 点到 12:06 点，之后是 119 ¥/€。 或者在时间轴上绘制：

```
        Undefined              114 ¥/€                116 ¥/€              119 ¥/€
|----[-inf, 12:00)----|----[12:00, 12:03)----|----[12:03, 12:06)----|----[12:06, now)----→
```

现在，如果我们提前知道所有的比率，我们就可以在行数据本身中明确地捕获这些区域。 但是，如果我们需要增量地建立这些区域，仅基于给定速率生效的开始时间，我们就会遇到问题：给定行的区域将随着时间的推移而变化，具体取决于它之后的行。 即使数据按顺序到达，这也是一个问题（因为每次新汇率到达时，之前的汇率从永远有效变为有效，直到新汇率到达时间），但如果它们可以到达则进一步复杂 *故障*。 例如，使用前面的`YenRates`表中的处理时间顺序，我们的表将有效表示随时间变化的时间线序列如下：

```
Range of processing time | Event-time validity timeline during that range of processing-time
=========================|==============================================================================
                         |
                         |      Undefined
        [-inf, 12:06:23) | |--[-inf, +inf)---------------------------------------------------------→
                         |
                         |      Undefined          114 ¥/€
    [12:06:23, 12:07:33) | |--[-inf, 12:00)--|--[12:00, +inf)--------------------------------------→
                         |
                         |      Undefined          114 ¥/€                              119 ¥/€
    [12:07:33, 12:09:07) | |--[-inf, 12:00)--|--[12:00, 12:06)---------------------|--[12:06, +inf)→
                         |
                         |      Undefined          114 ¥/€            116 ¥/€           119 ¥/€
         [12:09:07, now) | |--[-inf, 12:00)--|--[12:00, 12:03)--|--[12:03, 12:06)--|--[12:06, +inf)→

```

或者，如果我们想将其呈现为随时间变化的关系（每个快照关系之间的变化以黄色突出显示）：

```
12:10> SELECT TVR * FROM YenRatesWithRegion ORDER BY EventTime;
---------------------------------------------------------------------------------------------
|              [-inf, 12:06:23)               |            [12:06:23, 12:07:33)             |
| ------------------------------------------- | ------------------------------------------- |
| | Curr | Rate |  Region        | ProcTime | | | Curr | Rate |  Region        | ProcTime | |
| ------------------------------------------- | ------------------------------------------- |
| ------------------------------------------- | | Euro | 114  | [12:00, +inf)  | 12:06:23 | |
|                                             | ------------------------------------------- |
---------------------------------------------------------------------------------------------
|            [12:07:33, 12:09:07)             |              [12:09:07, +inf)               |
| ------------------------------------------- | ------------------------------------------- |
| | Curr | Rate |  Region        | ProcTime | | | Curr | Rate |  Region        | ProcTime | |
| ------------------------------------------- | ------------------------------------------- |
| | Euro | 114  | [12:00, 12:06) | 12:06:23 | | | Euro | 114  | [12:00, 12:03) | 12:06:23 | |
| | Euro | 119  | [12:06, +inf)  | 12:07:33 | | | Euro | 116  | [12:03, 12:06) | 12:09:07 | |
| ------------------------------------------- | | Euro | 119  | [12:06, +inf)  | 12:07:33 | |
|                                             | ------------------------------------------- |
---------------------------------------------------------------------------------------------
```

这里需要注意的是，一半的更改涉及对多行的更新。 这听起来可能还不错，直到您回想起这些快照中的每一个之间的区别恰好是一个新行的到来。 换句话说，单个新输入行的到来导致对多个输出行的事务性修改。 这听起来不太好。 另一方面，它听起来也很像构建会话窗口所涉及的多行事务。 事实上，这是窗口化的另一个例子，它提供的好处超出了简单的时间划分：它还提供了以涉及复杂的多行事务的方式这样做的能力。

为了实际看到这一点，让我们看一个动画。 如果这是一个 Beam 管道，它可能看起来像下面这样：

```
PCollection<Currency, Decimal> yenRates = ...;
PCollection<Decimal> validYenRates = yenRates
    .apply(Window.into(new ValidityWindows())
    .apply(GroupByKey.<Currency, Decimal>create());
```

在流/表动画中呈现，该管道看起来像 [Figure 9-1](#temporal_validity_windowing_over_time_chap_nine).

![img](./media/stsy_0901.mp4)

<center><i>Figure 9-1. Temporal validity windowing over time</center></i>

该动画强调了时间有效性的一个关键方面：缩小窗口。 有效性窗口必须能够随着时间的推移而缩小，从而缩小其有效性的范围并将其中包含的任何数据拆分到两个新窗口中。 有关部分实施示例，请参阅 [GitHub 上的代码片段](http://bit.ly/2N7Nn3A)。[5](ch09.html#idm139957173684048)

在 SQL 术语中，这些有效性窗口的创建将类似于以下内容（使用假设的“VALIDITY_WINDOW”构造），将其视为一个表：

```
12:10> SELECT TABLE 
         Curr,
         MAX(Rate) as Rate,
         VALIDITY_WINDOW(EventTime) as Window
       FROM YenRates 
       GROUP BY
         Curr,
         VALIDITY_WINDOW(EventTime)
       HAVING Curr = "Euro";
--------------------------------
| Curr | Rate | Window         |
--------------------------------
| Euro | 114  | [12:00, 12:03) |
| Euro | 116  | [12:03, 12:06) |
| Euro | 119  | [12:06, +inf)  |
--------------------------------
```


<center> VALIDITY WINDOWS IN STANDARD SQL</center>
>
> Note that it’s possible to describe validity windows in standard SQL using a three-way self-join:
>
> ```
> SELECT
>   r1.Curr,
>   MAX(r1.Rate) AS Rate,
>   r1.EventTime AS WindowStart,
>   r2.EventTime AS WIndowEnd
> FROM YenRates r1
> LEFT JOIN YenRates r2
>   ON r1.Curr = r2.Curr
>      AND r1.EventTime < r2.EventTime
> LEFT JOIN YenRates r3
>   ON r1.Curr = r3.Curr
>      AND r1.EventTime < r3.EventTime 
>      AND r3.EventTime < r2.EventTime
> WHERE r3.EventTime IS NULL
> GROUP BY r1.Curr, WindowStart, WindowEnd
> HAVING r1.Curr = 'Euro';
> ```
>
> Thanks to Martin Kleppmann for pointing this out.

或者，也许更有趣的是，将其视为一个流：

```
12:00> SELECT STREAM
         Curr,
         MAX(Rate) as Rate,
         VALIDITY_WINDOW(EventTime) as Window,
         Sys.EmitTime as Time,
         Sys.Undo as Undo,
       FROM YenRates
       GROUP BY
         Curr,
         VALIDITY_WINDOW(EventTime) 
       HAVING Curr = "Euro";
--------------------------------------------------
| Curr | Rate | Window         | Time     | Undo |
--------------------------------------------------
| Euro | 114  | [12:00, +inf)  | 12:06:23 |      |
| Euro | 114  | [12:00, +inf)  | 12:07:33 | undo |
| Euro | 114  | [12:00, 12:06) | 12:07:33 |      | 
| Euro | 119  | [12:06, +inf)  | 12:07:33 |      |
| Euro | 114  | [12:00, 12:06) | 12:09:07 | undo | 
| Euro | 114  | [12:00, 12:03) | 12:09:07 |      |
| Euro | 116  | [12:03, 12:06) | 12:09:07 |      |
................. [12:00, 12:10] .................
```

太好了，我们了解了如何使用时间点值有效地将时间分割成这些值有效的范围。 但这些时间有效性窗口的真正威力在于将它们应用于将它们与其他数据连接的情况下。 这就是时间有效性连接进来的地方。

### 时间有效性连接

为了探索时间有效性连接的语义，假设我们的金融应用程序包含另一个时变关系，一个跟踪从各种货币到日元的货币兑换订单的关系：

```
12:10> SELECT TABLE * FROM YenOrders;
----------------------------------------
| Curr | Amount | EventTime | ProcTime |
----------------------------------------
| Euro | 2      | 12:02:00  | 12:05:07 |
| USD  | 1      | 12:03:00  | 12:03:44 |
| Euro | 5      | 12:05:00  | 12:08:00 |
| Yen  | 50     | 12:07:00  | 12:10:11 |
| Euro | 3      | 12:08:00  | 12:09:33 |
| USD  | 5      | 12:10:00  | 12:10:59 |
----------------------------------------
```

为了简单起见，和以前一样，让我们关注欧元转换：

```
12:10> SELECT TABLE * FROM YenOrders WHERE Curr = "Euro";
----------------------------------------
| Curr | Amount | EventTime | ProcTime |
----------------------------------------
| Euro | 2      | 12:02:00  | 12:05:07 |
| Euro | 5      | 12:05:00  | 12:08:00 |
| Euro | 3      | 12:08:00  | 12:09:33 |
----------------------------------------
```

我们希望将这些订单稳健地连接到`YenRates`关系，将`YenRates`中的行视为定义有效窗口。 因此，我们实际上想要加入我们在上一节末尾构建的“YenRates”关系的有效性窗口版本：

```
12:10> SELECT TABLE
         Curr,
         MAX(Rate) as Rate,
         VALIDITY_WINDOW(EventTime) as Window
       FROM YenRates
       GROUP BY
         Curr,
         VALIDITY_WINDOW(EventTime)
       HAVING Curr = "Euro";
--------------------------------
| Curr | Rate | Window         |
--------------------------------
| Euro | 114  | [12:00, 12:03) |
| Euro | 116  | [12:03, 12:06) |
| Euro | 119  | [12:06, +inf)  |
--------------------------------
```

幸运的是，在我们将转换率放入有效窗口后，这些汇率和`YenOrders`关系之间的窗口连接给了我们想要的东西：

```
12:10> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT TABLE
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order, 
         ValidRates.Window as "Rate Window"
       FROM YenOrders FULL OUTER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro";
-------------------------------------------
| E | Y/E | Y   | Order  | Rate Window    |
-------------------------------------------
| 2 | 114 | 228 | 12:02  | [12:00, 12:03) |
| 5 | 116 | 580 | 12:05  | [12:03, 12:06) |
| 3 | 119 | 357 | 12:08  | [12:06, +inf)  |
-------------------------------------------
```

回想一下我们最初的`YenRates`和`YenOrders`关系，这个联合关系看起来确实是正确的：三个转换中的每一个都以（最终）适当的速率结束给定的事件时间窗口，在这些时间窗口内，它们的相应顺序就落在了这个窗口内。 所以我们有一种体面的感觉，这个连接正在做我们想要的，为我们提供我们想要的最终正确性。

也就是说，这个关系的简单快照视图是在所有值都已到达且尘埃落定之后拍摄的，掩盖了此连接的复杂性。 要真正了解这里发生了什么，我们需要查看完整的 TVR。 首先，回想一下有效性窗口转换率关系实际上比以前的简单表快照视图可能让您相信的要复杂得多。 作为参考，这里是有效性窗口关系的`STREAM`版本，它更好地突出了这些转化率随时间的演变：

```
12:00> SELECT STREAM
         Curr,
         MAX(Rate) as Rate,
         VALIDITY(EventTime) as Window,
         Sys.EmitTime as Time,
         Sys.Undo as Undo,
       FROM YenRates
       GROUP BY
         Curr,
         VALIDITY(EventTime)
       HAVING Curr = "Euro";
--------------------------------------------------
| Curr | Rate | Window         | Time     | Undo |
--------------------------------------------------
| Euro | 114  | [12:00, +inf)  | 12:06:23 |      |
| Euro | 114  | [12:00, +inf)  | 12:07:33 | undo |
| Euro | 114  | [12:00, 12:06) | 12:07:33 |      | 
| Euro | 119  | [12:06, +inf)  | 12:07:33 |      |
| Euro | 114  | [12:00, 12:06) | 12:09:07 | undo | 
| Euro | 114  | [12:00, 12:03) | 12:09:07 |      |
| Euro | 116  | [12:03, 12:06) | 12:09:07 |      |
................. [12:00, 12:10] .................
```

因此，如果我们查看有效性窗口连接的完整 TVR，您会发现随着时间的推移，此连接的相应演变要复杂得多，这是由于 加入：

```
12:10> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT TVR
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         ValidRates.Window as "Rate Window"
       FROM YenOrders FULL OUTER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro";
-------------------------------------------------------------------------------------------
|              [-inf, 12:05:07)              |            [12:05:07, 12:06:23)            |
| ------------------------------------------ | ------------------------------------------ |
| | E | Y/E | Y   | Order | Rate Window    | | | E | Y/E | Y   | Order | Rate Window    | |
| ------------------------------------------ | ------------------------------------------ |
| ------------------------------------------ | | 2 |     |     | 12:02 |                | |
|                                            | ------------------------------------------ |
-------------------------------------------------------------------------------------------
|            [12:06:23, 12:07:33)            |            [12:07:33, 12:08:00)            |
| ------------------------------------------ | ------------------------------------------ |
| | E | Y/E | Y   | Order | Rate Window    | | | E | Y/E | Y   | Order | Rate Window    | |
| ------------------------------------------ | ------------------------------------------ |
| | 2 | 114 | 228 | 12:02 | [12:00, +inf)  | | | 2 | 114 | 228 | 12:02 | [12:00, 12:06) | |
| ------------------------------------------ | |   | 119 |     |       | [12:06, +inf)  | |
|                                            | ------------------------------------------ |
-------------------------------------------------------------------------------------------
|            [12:08:00, 12:09:07)            |            [12:09:07, 12:09:33)            |
| ------------------------------------------ | ------------------------------------------ |
| | E | Y/E | Y   | Order | Rate Window    | | | E | Y/E | Y   | Order | Rate Window    | |
| ------------------------------------------ | ------------------------------------------ |
| | 2 | 114 | 228 | 12:02 | [12:00, 12:06) | | | 2 | 114 | 228 | 12:02 | [12:00, 12:03) | |
| | 5 | 114 | 570 | 12:05 | [12:03, 12:06) | | | 5 | 116 | 580 | 12:05 | [12:03, 12:06) | |
| |   | 119 |     |       | [12:06, +inf)  | | |   | 119 |     | 12:08 | [12:06, +inf)  | |
| ------------------------------------------ | ------------------------------------------ |
-------------------------------------------------------------------------------------------
|               [12:09:33, now)              |
| ------------------------------------------ |
| | E | Y/E | Y   | Order | Rate Window    | |
| ------------------------------------------ |
| | 2 | 114 | 228 | 12:02 | [12:00, 12:03) | |
| | 5 | 116 | 580 | 12:05 | [12:03, 12:06) | |
| | 3 | 119 | 357 | 12:08 | [12:06, +inf)  | |
| ------------------------------------------ |
----------------------------------------------
```

特别是，5 欧元订单的结果最初报价为 570 ¥，因为该订单（发生在 12:05）最初属于 114 ¥/€ 费率的有效窗口。 但是，当事件时间 12:03 的 116 ¥/€ 费率出现故障时，5 € 订单的结果必须从 570 ¥ 更新为 580 ¥。 如果您以流的形式观察连接的结果，这一点也很明显（这里我用红色突出显示了不正确的 570 ¥，并以蓝色突出显示了 570 ¥ 的撤回和随后的 580 ¥ 更正值）：

```
12:00> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT STREAM
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         ValidRates.Window as "Rate Window",
         Sys.EmitTime as Time,
         Sys.Undo as Undo
       FROM YenOrders FULL OUTER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = “Euro”;
------------------------------------------------------------
| E | Y/E | Y   | Order | Rate Window    | Time     | Undo | 
------------------------------------------------------------
| 2 |     |     | 12:02 |                | 12:05:07 |      |
| 2 |     |     | 12:02 |                | 12:06:23 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, +inf)  | 12:06:23 |      |
| 2 | 114 | 228 | 12:02 | [12:00, +inf)  | 12:07:33 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, 12:06) | 12:07:33 |      |
|   | 119 |     |       | [12:06, +inf)  | 12:07:33 |      |
| 5 | 114 | 570 | 12:05 | [12:00, 12:06) | 12:08:00 |      |
| 2 | 114 | 228 | 12:02 | [12:00, 12:06) | 12:09:07 | undo |
| 5 | 114 | 570 | 12:05 | [12:00, 12:06) | 12:09:07 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, 12:03) | 12:09:07 |      |
| 5 | 116 | 580 | 12:05 | [12:03, 12:06) | 12:09:07 |      |
|   | 119 |     |       | [12:06, +inf)  | 12:09:33 | undo |
| 3 | 119 | 357 | 12:08 | [12:06, +inf)  | 12:09:33 |      |
...................... [12:00, 12:10] ......................

```

值得一提的是，由于使用了`FULL OUTER`连接，这是一个相当混乱的流。 实际上，当将转换订单作为流使用时，您可能并不关心未连接的行； 切换到 `INNER` 连接有助于消除这些行。 您可能也不关心费率窗口发生变化的情况，但实际转化价值不受影响。 通过从流中删除速率窗口，我们可以进一步减少它的冗长：

```
12:00> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT STREAM
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         ValidRates.Window as "Rate Window",
         Sys.EmitTime as Time,
         Sys.Undo as Undo
       FROM YenOrders INNER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro";
-------------------------------------------
| E | Y/E | Y   | Order | Time     | Undo |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | 12:06:23 |      |
| 5 | 114 | 570 | 12:05 | 12:08:00 |      |
| 5 | 114 | 570 | 12:05 | 12:09:07 | undo |
| 5 | 116 | 580 | 12:05 | 12:09:07 |      |
| 3 | 119 | 357 | 12:08 | 12:09:33 |      |
............. [12:00, 12:10] ..............
```

好多了。 我们现在可以看到这个查询非常简洁地完成了我们最初打算做的事情：以一种能够容忍乱序到达的数据的健壮方式连接两个 TVR 以获取货币兑换率和订单。 [图 9-2](#temporal_validity_join_converting_euros_to_yen_with_per_record_triggering) 将此查询可视化为动画图。 在其中，您还可以非常清楚地看到事物的整体结构随着时间的推移而变化的方式。




![img](./media/stsy_0902.mp4)

<center><i>Figure 9-2. Temporal validity join, converting Euros to Yen with per-record triggering</center></i>

#### 水印和时间有效性连接

在这个例子中，我们强调了本节开头提到的窗口化连接的第一个好处：窗口化连接允许您根据实际业务需求在一定时间内对连接进行分区。 在这种情况下，业务需要将时间划分为货币兑换率的有效区域。

然而，在我们结束之前，事实证明这个例子也提供了一个机会来强调我提到的第二点：窗口连接可以为水印提供一个有意义的参考点。 要了解这有何用处，想象一下更改先前的查询以将隐式默认的每条记录触发器替换为显式水印触发器，该触发器仅在水印通过连接中的有效性窗口末尾时触发一次（假设我们有一个水印 可用于我们的输入 TVR，准确跟踪事件时间中这些关系的完整性，以及知道如何考虑这些水印的执行引擎）。 现在，我们的流不再包含多个输出和因乱序到达的费率而撤回，我们最终可以得到一个包含每个订单的单个正确转换结果的流，这显然比以前更理想：

```
12:00> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT STREAM
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         Sys.EmitTime as Time,
         Sys.Undo as Undo
       FROM YenOrders INNER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro"
       EMIT WHEN WATERMARK PAST WINDOW_END(ValidRates.Window);
-------------------------------------------
| E | Y/E | Y   | Order | Time     | Undo |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | 12:08:52 |      |
| 5 | 116 | 580 | 12:05 | 12:10:04 |      |
| 3 | 119 | 357 | 12:08 | 12:10:13 |      |
............. [12:00, 12:11] ..............
```

或者，呈现为动画，它清楚地显示了如何在水印超出它们之前将连接结果发送到输出流中，如 [图 9-3](#temporal_validity_join_converting_euros_to_yen_with_watermark_triggering) 中所示。




![img](./media/stsy_0903.mp4)

<center><i>Figure 9-3. Temporal validity join, converting Euros to Yen with watermark triggering</center></i>

无论哪种方式，看到此查询如何将如此复杂的一组交互封装到所需结果的简洁明了的呈现中，都令人印象深刻。




## 总结

在本章中，我们分析了流处理上下文中的连接世界（使用 SQL 的连接词汇表）。 我们从无窗口连接开始，并从概念上了解所有连接如何以流式连接为核心。 我们看到了基本上所有其他连接变体的基础是`FULL OUTER`连接，并讨论了作为`LEFT OUTER`、`RIGHT OUTER`、`INNER`、`ANTI`、` SEMI`，甚至`CROSS` 连接。 此外，我们还看到了所有这些不同的连接模式如何在 TVR 和流的世界中交互。

接下来我们转向窗口连接，并了解到窗口连接通常是由以下一个或两个好处驱动的：

- 为某些业务需求*在一段时间内对连接进行分区*的能力
- 能够将连接的结果*与水印的进度*联系起来*

最后，我们深入探讨了一种关于加入的更有趣和有用的窗口类型：时间有效性窗口。 我们看到时间有效性窗口如何非常自然地将时间划分为给定值的有效性区域，仅基于这些值发生变化的特定时间点。 我们了解到，在有效性窗口内加入需要一个窗口框架，该框架支持可以随时间拆分的窗口，这是目前现有的流媒体系统不支持的。 我们看到了有效性窗口是如何简洁地让我们以稳健、自然的方式解决将货币兑换率和订单的 TVR 连接在一起的问题。

连接通常是数据处理、流式传输或其他方面中比较令人生畏的方面之一。 然而，通过了解连接的理论基础以及我们如何直接从该基础推导出所有不同类型的连接，连接就变得不那么可怕了，即使流式处理增加了额外的时间维度。
