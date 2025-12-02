---
title: "三门问题（蒙提霍尔悖论）分析与Golang模拟"
date: 2019-10-10T00:00:00+08:00
---

### 问题描述

三门问题（Monty Hall problem）亦称为蒙提霍尔问题、蒙特霍问题或蒙提霍尔悖论，大致出自美国的电视游戏节目 Let's Make a Deal。问题名字来自该节目的主持人蒙提·霍尔（Monty Hall）。参赛者会看见三扇关闭了的门，其中一扇的后面有一辆汽车，选中后面有车的那扇门可赢得该汽车，另外两扇门后面则各藏有一只山羊。当参赛者选定了一扇门，但未去开启它的时候，节目主持人开启剩下两扇门的其中一扇，露出其中一只山羊。主持人其后会问参赛者要不要换另一扇仍然关上的门。问题是：换另一扇门会否增加参赛者赢得汽车的机率？

### 答案

答案是会。不换门的话，赢得汽车的几率是 1/3。换门的话，赢得汽车的几率是 2/3。

### 争议

有人认为，在主持人排除了一个门之后，汽车只可能在另外两个门中，所以在两扇门的概率各是 1/2。

### 分析

首先参赛者选定了一扇门，主持人未开启门时，汽车在这扇门的概率为 1/3，在另外两扇门中的概率为 2/3，此时争议不大。而另外两扇门中必定至少有一扇是山羊，所以即使主持人指出这两扇门中一扇是山羊，并不会影响这两扇门的概率，两扇概率和仍为 2/3，此时一扇已知是山羊，所以两扇中的另外一扇是汽车的概率是 2/3。所以换门会提高概率。

### 思考

如果主持人开启揭露一扇门是山羊后，另外一个人 B 此时在剩下的两扇门中做抉择，并且他不知道其他信息，只知道一扇是汽车，一扇是羊，那么此时 B 选择到汽车的概率是 1/2。
这是因为没有之前的信息，B 不知道那扇门概率大，B 此时是在两扇门中做随机选择，B 可能有 1/2 的概率选择 A(参赛者)开始选择的门，也有 1/2 的概率选择 A 将要换的门。所以 B 选择到汽车的概率为 1/2 _ 1/3 + 1/2 _ 2/3 = 1/2。

### 程序模拟

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	totalTimes := 1000000
	aRightTimes := 0
	bRightTimes := 0

	rand.Seed(time.Now().UnixNano())

	for time := 1; time <= totalTimes; time++ {
		doors := [3]bool{false, false, false}
		doors[rand.Intn(3)] = true

		aFirstChoice := rand.Intn(3)
		var hostChoice, aChangedChoice int

		for i, door := range doors {
			if i != aFirstChoice && !door {
				hostChoice = i
			}
		}

		for i, _ := range doors {
			if i != aFirstChoice && i != hostChoice {
				aChangedChoice = i
			}
		}

		var bChoice int

		if rand.Intn(2) == 0 {
			bChoice = aFirstChoice
		} else {
			bChoice = aChangedChoice
		}

		if doors[aChangedChoice] {
			aRightTimes++
		}

		if doors[bChoice] {
			bRightTimes++
		}
	}

	fmt.Println("totalTimes: ", totalTimes)
	fmt.Println("aChangedChoice: ", aRightTimes)
	fmt.Println("bChoice: ", bRightTimes)
}
```

结果符合预期，A 换门后正确概率为 2/3，B 随机选择正确的概率为 1/2：

```
totalTimes:  1000000
aChangedChoice:  667407
bChoice:  499262
```

### reference

问题描述引自百度百科
