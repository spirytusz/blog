---
title: Lua语言学习笔记
date: 2019-04-26 20:48:21
header-img: "cover.jpg"
author:     "spirytusz"
tags:
- lua
---

# 0. 前言
最近学习了Lua语言，记录一下自己觉得对几个重要概念的学习过程。

<!--more-->

# 1. Table
table是Lua语言的一个重要的数据结构。它很像一个Map，我们可以通过给出一个key来获得对应的value。并且，table的key可以是除nil以外的任意类型。看代码：
{% codeblock lang:Lua %}
local tab = {}

tab.a = 1

tab['b'] = '233'

tab[f] = function()
	print('call a function')
end

for k, v in pairs(tab) do
	print(string.format('tab.%s = %s', tostring(k), tostring(v)))
end

-- Output:
-- tab.a = 1
-- tab.b = 233
-- tab.f = function
{% endcodeblock %}

Lua的table不止于此，还有很多骚操作。

## 1.1. MetaTable
MetaTable是Lua中元表。个人认为，元表是对table操作时触发的行为的集合。「触发的行为」是什么？它可以是一个function，定义这个行为做什么；也可以是一个table，定义这个行为的备选table。元表可以有很多属性，具体参照[官网](https://www.lua.org/pil/13.html)，我以\__index为例。

### 1.1.1. __index

\__index定义了在table中通过给定的key找到的value为nil时怎么办的行为。话不多说看代码：
{% codeblock lang:Lua %}
local aTable = {}
local aMetatable = {}

print(aTable.y)

setmetatable(aTable, aMetatable)
print(aTable.y)

aMetatable.__index = function(t,k)
	-- t就是aTable
	local tempTable = { y = 666 }
	return tempTable[k]
end
print(aTable.y)

-- Output:
-- nil
-- nil
-- 666
{% endcodeblock %}

首先先声明和定义两个table，aMetatable后面用作aTable的元表。元表同样也是一个表，所以这么声明没毛病。然后获取aTable的y属性的值，不用想，肯定是获得的是一个空值。接着，把aTable的元表设为aMetatable，然后再获取一次aTable的y属性的值。同样的，获得的是一个空值。为什么？因为aTable的元表没有任何可以触发的行为。那就为aTable的元表增加一个行为\__index，在打印一个aTable的y属性的值，这会就打印出666了。总结一下这个过程：当我们访问aTable的y属性时，Lua虚拟机发现它是空值，所以他就会在aTable的元表中找到\__index这个属性，如果这个属性是一个function，那就执行它，并把它的执行结果，返回作aTable的y属性的值。

当然上面的代码在设置元表时可以更加简化：
{% codeblock lang:Lua %}
aMetatable.__index = { y = 666 }
{% endcodeblock %}

执行完这段语句，元表中\__index这个行为就是一个table了。这个当我们访问aTable的y属性时，Lua虚拟机发现aTable.y是空的，就会去aMetatable.\__index这个「表」里面把y作为key去取一个值并返回。这与上面的代码是等价的。

然而我总感觉还少了点什么，上面的代码，我只是根据输出来猜测它的行为，而不能确定它是怎么做到的。于是我在Lua的源代码里，全局搜索关键词「\__index」，成功定位到\__index的实现：
{% codeblock lang:C lvm.c %}
/*
** Finish the table access 'val = t[key]'.
** if 'slot' is NULL, 't' is not a table; otherwise, 'slot' points to
** t[k] entry (which must be nil).
*/
void luaV_finishget (lua_State *L, const TValue *t, TValue *key, StkId val,
                      const TValue *slot) {
  int loop;  /* counter to avoid infinite loops */
  const TValue *tm;  /* metamethod */
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    if (slot == NULL) {  /* 't' is not a table? */
      lua_assert(!ttistable(t));
      tm = luaT_gettmbyobj(L, t, TM_INDEX);
      if (ttisnil(tm))
        luaG_typeerror(L, t, "index");  /* no metamethod */
      /* else will try the metamethod */
    }
    else {  /* 't' is a table */
      lua_assert(ttisnil(slot));
      tm = fasttm(L, hvalue(t)->metatable, TM_INDEX);  /* table's metamethod */
      if (tm == NULL) {  /* no metamethod? */
        setnilvalue(val);  /* result is nil */
        return;
      }
      /* else will try the metamethod */
    }
    if (ttisfunction(tm)) {  /* is metamethod a function? */
      luaT_callTM(L, tm, t, key, val, 1);  /* call it */
      return;
    }
    t = tm;  /* else try to access 'tm[key]' */
    if (luaV_fastget(L,t,key,slot,luaH_get)) {  /* fast track? */
      setobj2s(L, val, slot);  /* done */
      return;
    }
    /* else repeat (tail call 'luaV_finishget') */
  }
  luaG_runerror(L, "'__index' chain too long; possible loop");
}
{% endcodeblock %}

解释一下，首先定义声明一个loop防止死循环，tm存储在元表中查找\__index的结果。至于为什么要防止死循环可以不管，因为不是我们读源码的目的。接着定位到for循环内的第一个if-else分支，if分支内，注释说这是t不是一个table的情况。我们可以跳过，看看else分支，else分支是t是table的情况。else分支会去找table: t的元表，如果找到的元表为空，或者是元表中找不到\__index属性，那就把结果设置为空，提前返回。如果找到了\__index那就继续。接着看第二个if分支，如果\__index是一个函数，那就用luaT_callTM调用它，luaT_callTM的代码如下：

{% codeblock lang:C %}
void luaT_callTM (lua_State *L, const TValue *f, const TValue *p1,
                  const TValue *p2, TValue *p3, int hasres) {
  ptrdiff_t result = savestack(L, p3);
  StkId func = L->top;
  setobj2s(L, func, f);  /* push function (assume EXTRA_STACK) */
  setobj2s(L, func + 1, p1);  /* 1st argument */
  setobj2s(L, func + 2, p2);  /* 2nd argument */
  L->top += 3;
  if (!hasres)  /* no result? 'p3' is third argument */
    setobj2s(L, L->top++, p3);  /* 3rd argument */
  /* metamethod may yield only when called from Lua code */
  if (isLua(L->ci))
    luaD_call(L, func, hasres);
  else
    luaD_callnoyield(L, func, hasres);
  if (hasres) {  /* if has result, move it to its place */
    p3 = restorestack(L, result);
    setobjs2s(L, p3, --L->top);
  }
}
{% endcodeblock %}

可以看到，luaT_callTM先把栈的状态保存起来，再把\__index这个函数，及其第一个参数，第二个参数推入，因为hasres为1，所以第一个if分支不执行。接着，第二个if-else就调用\__index方法。到了第三个if分支，因为hasres为1，所以会执行这个分支。这个if分支会还原栈的状态，并把结果赋值给p3，也就是上游传过来的val，然后把结果推入栈中。结束。
再回到luaV_finishget，到了最后一个if分支，看代码的意思，就是直接把\__index当做一个table，在这个table中以给定的key查找value，并把查找结果返回。至此\__index的实现原理就结束了。
结论是，如果\__index是一个function，那就会把原table以及key传入给这个function，这个function处理后把结果返回，Lua虚拟机会把这个结果当做是查询结果；如果\__index是一个table，那就用给定的key在\__index中查询，并把结果返回。这和上面的猜测是相符的。

## 1.2. Function的默认参数
我们初始化一个对象，这个对象里面可能有些属性不是必填的。比如一个person，它的属性name、age、sex都是必填的，而height、weight是选填的。我们很自然的就会这么定义一个函数来初始化person：
{% codeblock lang:Lua %}
function initPerson(name, age, sex, height, weight)
	-- 初始化..
	local person = getDefault()
	person.name = name
	person.age = age
	person.sex = sex
	person.height = height or 0
	person.weight = weight or 0
	return person
end

function printPerson( person )
	print(string.format(
			'name = %s, age = %d, sex = %s, height = %d, weight = %d', 
			person.name, 
			person.age, 
			person.sex, 
			person.height, 
			person.weight
	))
end

-- 仅传入必填属性
local p1 = initPerson('Q1', 23, 'female')
printPerson(p1)

-- 传入必填属性+身高？
local p2 = initPerson('Q2', 23, 'female', 169)
printPerson(p1)

-- 传入必填属性+体重？
local p3 = initPerson('Q3', 23, 'female', 55)
printPerson(p1)

-- Output:
-- name = Q1, age = 23, sex = female, height = 0, weight = 0
-- name = Q2, age = 23, sex = female, height = 169, weight = 0
-- name = Q3, age = 23, sex = female, height = 55, weight = 0
{% endcodeblock %}

输出不符合我们的预期，因为Lua在传递参数是会把实参顺序推入到栈中，再按顺序对号入座到形参。如何解决默认参数的问题，我们可以传入一个table，这个table中以key为参数，value为参数的值。在初始化person的函数中，我们用key来在传来的table中取出对应参数的值，如果取出来的value为空，那就或一下，给它设置一个默认值就好了。代码如下：

{% codeblock lang:Lua %}
function initPerson( tPerson )
	-- 初始化..
	local person = getDefault()
	person.name = tPerson.name
	person.age = tPerson.age
	person.sex = tPerson.sex
	person.height = tPerson.height or 0
	person.weight = tPerson.weight or 0
	return person
end

-- 仅传入必填属性
local p1 = initPerson({name = 'Q1', age = 23, sex = 'female'})
printPerson(p1)

-- 传入必填属性+身高？
local p2 = initPerson({name = 'Q1', age = 23, sex = 'female', height = 169})
printPerson(p1)

-- 传入必填属性+体重？
local p3 = initPerson({name = 'Q1', age = 23, sex = 'female', weight = 55})
printPerson(p1)

-- Output:
-- name = Q1, age = 23, sex = female, height = 0, weight = 0
-- name = Q2, age = 23, sex = female, height = 169, weight = 0
-- name = Q3, age = 23, sex = female, height = 0, weight = 55
{% endcodeblock %}

结果符合预期。不过，上面的代码，严格意义上来说，person的五个属性都成了可选参数，因为开发者是可能会忘了填name、age或sex属性。解决方法是：要么在开发的时候，开发者要知道name，age和sex一定要填值；要么就直接把name，age和sex单独抽出来，在加上一个table作为initPerson的参数列表，像这样
{% codeblock lang:Lua %}
function initPerson(name, age, sex, tOptArgs )
	-- 初始化..
	local person = getDefault()
	person.name = name
	person.age = age
	person.sex = sex
	tOptArgs = tOptArgs or {}
	person.height = tOptArgs.height or 0
	person.weight = tOptArgs.weight or 0
	return person
end
{% endcodeblock %}
才能做到完美的必选参数+可选参数的初始化。

# 2. Lua中的面向对象

Lua支持一定的OOP。Lua本身没有提供面向对象编程的支持，当时我们可以用Lua的一个重要数据结构「table」来模拟OOP的过程。不多说，上代码。
{% codeblock lang:Lua %}
MyObject = {
	name = "MyObject",
	doWhat = "something"
}

function MyObject:newInstance( obj )
    obj = obj or {}
    setmetatable(obj, self)
    self.__index = function(t,k)
    	return self[k]
    end
    obj.name = "Q"
    obj.fieldB = "eat"
    return obj
end

function  MyObject:doSomething()
	print(string.format('%s do %s.', self.name, self.doWhat))
end

local oneObj = MyObject:newInstance()
oneObj:doSomething()

-- Output:
-- Q do eat.
{% endcodeblock %}

MyObject这个表，有两个属性，name和doWhat，我们可以把它看做一个“类”；并且还定义了两个方法newInstance和doSomething。形如「XXX.xxx()」和「XXX:xxx()」的形式是Lua语言的语法糖，同样都是在“类”中声明一个函数：
{% codeblock lang:Lua %}
// 1
Person.say = function(self)
end

// 2
function Person.say(self)
end

// 3
function Person:say()
end
{% endcodeblock %}

上面的代码中，三者是等价的，同样为Person中的say属性赋值一个函数。对于1和2，2是Lua的语法糖，2等价于1。对于2和3，3是Lua的语法糖，「.」号和「:」号的区别在于，「:」号会在调用函数时，首先推入一个self，再推入函数的参数。

然后看看newInstance函数。它首先对obj进行或操作，确保传进来的obj不为空，保证其至少是一个空表。然后，就是为obj设置元表，设置为self，而self就是MyObject。接着就是为self设置一个属性\__index，这个属性的值是一个function。和上面的setmetatable联合来看，这两句语句的意思是：
如果在obj中，根据一个key找到的结果是nil，那就去执行\__index这个function。在这个function中，会去查找self这个表并返回，self就是MyObject。所以，如果我们访问obj的doSomething属性，因为obj没有，那就执行\__index，在MyObject中查找，找到了，那就返回作查询结果。所以newInstance还有另一个版本：
{% codeblock lang:Lua %}
function MyObject:newInstance( obj )
    obj = obj or {}
    setmetatable(obj, self)
    self.__index = self
    obj.name = "Q"
    obj.fieldB = "eat"
    return obj
end
{% endcodeblock %}
更加的简化，意思是如果在obj中，根据的key到的结果是空，那就用这个key去self中查找，并作为查询结果。（这个版本我一开始无法理解，看了Lua的源码才知道是什么意思，还是function版的好理解..）

回到newInstance中，接下来就是为obj设置一些属性，然后返回。在doSomething中，因为我们执行的是
{% codeblock lang:Lua %}
oneObj:doSomething()
{% endcodeblock %}
所以在doSomething中，self就是oneObj。oneObj的name属性和doWhat属性是'Q'和'eat'，所以输出符合预期。

# 3. 函数式编程
Lua支持函数式编程。因为我之前更熟悉Java，转到Lua一时半会理解不了函数式编程。所以新的概念，我喜欢和Java比较。Lua中的函数式编程，就是把function看成是一个「值」，你可以在任意一个地方声明它，也可以把它赋值到某一个变量中。所以，只要把Lua中的函数当成一个值就好了，只不过这个值不能加减乘除和逻辑变换罢了。所以，下面的代码在Lua中是合法的：
{% codeblock lang:Lua %}
local f = function()
	return '2333'
end

function test()
	print(f())
	f = function()
		return '666'
	end
	print(f())
end

-- Output:
-- 2333
-- 666
{% endcodeblock %}

可以看到上面的代码，test中有嵌套了一个function。我在想，如果这个function访问了test的局部变量，那会是什么情形？做个实验：
{% codeblock lang:Lua %}
function getIncreaser()
	local level = 0
	return function()
		level = level + 1
		return level
	end
end

local increaser = getIncreaser()
for i = 1, 5 do
	print(increaser())
end

-- Output:
-- 1
-- 2
-- 3
-- 4
-- 5
{% endcodeblock %}

讲道理，getIncreaser的level仅在getIncreaser的生命周期内有效。然后，getIncreaser返回的function中持有了level，所以在getIncreaser退出后，level并没有释放，因为increaser持有了它。所以每调用一次increaser，level就会自增一次，就是一个简单的自增器。这种现象，有一个很厉害的名字，叫做「闭包(Closure)」

简单的了解了函数式编程后，我继续和Java比较。Java中，回调函数怎么做？传一个函数？不行，因为Java不能把function作为参数。那就把这个function包装成一个类，再把这个类的实例作为参数就好了：
{% codeblock lang:Java%}
public interface Callback {
	void callback();
}

public class MyProcessor {

	private Callback mCallback;
	
	public void setCallback(Callback callback) {
		mCallback = callback
	}

	public void notifyCallback() {
		if (mCallback != null) {
			mCallback.callback();
		}
	}
}
{% endcodeblock %}
好啰嗦啊，我只是要回调而已，如果是观察者模式，那我还要维护一个List。Lua支持函数式编程，那就只需这样：
{% codeblock lang:Lua %}
function setCallback(callback)
	myProcessor.callback = callback
end

function notifyCallback()
	if myProcessor.callback then
		myProcessor.callback()
	end
end
{% endcodeblock %}
很简洁。如果是观察者模式，那就把callback插入到一个table就可以了，需要notify的时候遍历一下，挨个调用就好了。

# 4. 总结
1. table是Lua的一个数据结果，其行为类似于一个map。
2. metatable是对table操作时触发的行为的一个集合。
3. 可以用table来实现function的默认参数。
4. 运用table + metatable可以实现简单的OOP。
5. Lua支持函数式编程与闭包。

# 5. 感想
刚开始学Lua的时候，感觉它就是一个动态类型的语言。学完之后，觉得table很重要，只要精通table，我觉得就能精通Lua的七八成。另外，学了Lua之后，有了比较，才觉得Java有点啰嗦（非贬义，Java有他的道理），才能理解Kotlin中一些api为什么要这么设计，以及设计的理由是什么。虽然说技多不压身，但是学完之后一定要比较，我觉得才能理解作者设计某一门语言的理由，它适用于什么情况，不适用于什么情况。有了比较，才能更好地使用一门语言，写出更好的代码，因为编程是一门艺术。没有比较，我觉得学再多也没用。













