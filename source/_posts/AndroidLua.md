---
title: 记Android层执行Lua脚本的一次实践
date: 2019-07-23 14:41:24
header-img: "cover.jpg"
tags:
- Android
- lua
---

# 0. 前言
&nbsp;&nbsp;&nbsp;&nbsp;最近一直在写Lua脚本，有时候出了问题，不知道是Lua层的问题，还是上游的问题，不知道从何下手。于是我学习了一点C/C++和JNI，把整个解析Lua脚本包、执行Lua脚本的流程全部都读了一遍。熟悉了一遍之后，就萌生了自己封一个Android跑Lua脚本库的想法。于是就有这篇博文。C/C++和Kotlin我都不熟，所以这次我主要用这两种语言来写（所以会很Java Style）。

<!--more-->

# 1. 环境搭建
首先现在Lua[官网](https://www.lua.org/download.html)下载Lua的源码，我用的是5.3.5版本的。然后把源码导入到Project中，写好CMakeList：
{% codeblock lang:CMake %}
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.
add_definitions(-Wno-deprecated)

add_library( # Sets the name of the library.
        luabridge

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        src/main/jni/lua/lapi.c
        src/main/jni/lua/lauxlib.c
        src/main/jni/lua/lbaselib.c
        src/main/jni/lua/lbitlib.c
        src/main/jni/lua/lcode.c
        src/main/jni/lua/lcorolib.c
        src/main/jni/lua/lctype.c
        src/main/jni/lua/ldblib.c
        src/main/jni/lua/ldebug.c
        src/main/jni/lua/ldo.c
        src/main/jni/lua/ldump.c
        src/main/jni/lua/lfunc.c
        src/main/jni/lua/lgc.c
        src/main/jni/lua/linit.c
        src/main/jni/lua/liolib.c
        src/main/jni/lua/llex.c
        src/main/jni/lua/lmathlib.c
        src/main/jni/lua/lmem.c
        src/main/jni/lua/loadlib.c
        src/main/jni/lua/lobject.c
        src/main/jni/lua/lopcodes.c
        src/main/jni/lua/loslib.c
        src/main/jni/lua/lparser.c
        src/main/jni/lua/lstate.c
        src/main/jni/lua/lstring.c
        src/main/jni/lua/lstrlib.c
        src/main/jni/lua/ltable.c
        src/main/jni/lua/ltablib.c
        src/main/jni/lua/ltm.c
        src/main/jni/lua/lua.c
        #src/main/jni/lua/luac.c
        src/main/jni/lua/lundump.c
        src/main/jni/lua/lutf8lib.c
        src/main/jni/lua/lvm.c
        src/main/jni/lua/lzio.c)

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

find_library( # Sets the name of the path variable.
        log-lib

        # Specifies the name of the NDK library that
        # you want CMake to locate.
        log)

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
        luabridge

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib})
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;我要跑是 **\*.lua** 类型的脚本，那就留下**lua.c**并删掉**luac.c**，CMakeList里面也要跟着注释掉。另外，因为我把Lua的源代码导入进来当做一个库，所以也不需要main方法了，把lua.c里面的main方法注释掉。最后Rebuild一下Project就可以了。

# 2. Android单向调用Lua
&nbsp;&nbsp;&nbsp;&nbsp;先定一个小目标，Android层调用Lua层的函数，Lua层做一个加法后把结果返回给Android层。先写好Lua脚本：
{% codeblock lang:Lua %}
function test(a, b)
	return a + b
end
{% endcodeblock %}
&nbsp;&nbsp;&nbsp;&nbsp;这个Lua脚本很简单，把传过来的a和b相加后返回。现在我们可以开始考虑Native层的实现。在考虑实现之前，需要了解Lua虚拟栈和几个Lua C API。

## 2.1. Lua虚拟栈
&nbsp;&nbsp;&nbsp;&nbsp;Lua层和Native层的数据交换是通过Lua虚拟栈来完成的。这个虚拟栈和普通的栈略有不同，它可以通过负值索引来访问指定元素。如图：
{% asset_img lua_stack.png Lua虚拟栈 %}
&nbsp;&nbsp;&nbsp;&nbsp;和普通的栈一样，Lua虚拟栈同样遵循先进后出原则，索引从下往上增加。不同的是Lua虚拟栈支持负值索引，使用负值索引可以自栈顶向下索引。

## 2.2. Lua C APIs
Lua提供了C APIs，方便Native层和Lua层之间的通讯。下面的Demo会用到这几个C API。

*    lua_State \*luaL_newstate (void);

     新建一个Lua的context。

*    int luaL\_loadbuffer (lua\_State \*L, const char \*buff, size_t sz, const char \*name);
     
     编译一个Lua chunk。如果编译成功，它会把编译结果包装成一个函数，并把这个函数推入到栈中；否则，编译失败，它会把错误信息推入栈中。

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | buff |    const char\* | 需要加载的Lua脚本buffer   |
     | sz |    size_t | Lua脚本buffer的长度   |
     | name |    const char\* | 这个chunk的名称，可空   |

*    int lua\_pcall (lua_State \*L, int nargs, int nresults, int errfunc);
     
     以安全模式调用一个函数，即使抛出异常也不会崩溃。当抛出异常时，如果errfunc为0，Lua虚拟机会把错误信息推入到Lua虚拟栈中，如果errfunc不为0，则错误处理会交由Lua虚拟栈中索引为errfunc的函数处理。执行结束后，Lua虚拟机会把参数以及调用的函数从栈中弹出。

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | nargs |    int | 需要调用的函数的参数个数   |
     | nresults |    int | 需要调用的函数的返回结果个数   |
     | errfunc |    int |  错误处理函数在Lua虚拟栈中的索引，如果为0，错误信息会推入到Lua虚拟栈中   |

*    void lua\_getglobal (lua_State \*L, const char \*name);

     获取名字为name的全局变量，并推入栈中。

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | name |    const char\* | 变量名称   |

* void lua\_pushinteger (lua\_State \*L, lua_Integer n);
* 
     推入一个lua_Integer类型的数据到栈中

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | n |    lua_Integer | 需要推入的数字   |  

*    lua\_Integer lua\_tointeger (lua_State \*L, int index); 
     
     将栈中的索引为index的元素转lua_Integer并返回

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | index |    int | 指定元素在栈中的索引   | 

&nbsp;&nbsp;&nbsp;&nbsp;除了这些C API，其他的介绍及其用法可以查看官网的[说明](https://www.lua.org/manual/5.1/manual.html#3.7)。


&nbsp;&nbsp;&nbsp;&nbsp;通过理解Lua虚拟栈和了解一些Lua C API，我们就可以实现一个简单的Native层调用Lua层函数的功能。
{% codeblock lang:C %}
jint startScript(JNIEnv* env, jobject obj, jstring jLuaStr, jint a, jint b) {
	// 创建一个lua context
	lua_State* luaContext = lua_newstate();
	// 初始化lua lib
	luaL_openlibs(luaContext);
	const char* cLuaStr = env->GetStringUTFChars(jLuaStr, NULL);
	
	// 加载buff到内存
	int loadStatus = luaL_loadbuffer(luaContext, cLuaStr, strlen(cLuaStr), NULL);
	if (LUA_OK != loadStatus) {
        const char *szError = luaL_checkstring(luaContext, -1);
        Log_e(LOG_TAG, "%s", szError);
        return -1;
    }
	env->ReleaseStringUTFChars(jLuaStr, cLuaStr);
	int callStatus = lua_pcall(luaContext, 0, LUA_MULTRET, 0);
    if (LUA_OK != callStatus) {
        const char *szError = luaL_checkstring(luaContext, -1);
        Log_e(LOG_TAG, "%s", szError);
        return -1;
    }
	
	// 获取test方法
	lua_getglobal(luaContext, "test");
    if (LUA_TFUNCTION != lua_type(luaContext, -1)) {
        Log_d(LOG_TAG, "can not found func : %s", "test");
        return false;
    }
	
	// 推入参数
	lua_pushinteger(luaContext, a);
	lua_pushinteger(luaContext, b);
	
	// 执行test方法
	int callTestStatus = lua_pcall(luaContext, 2, 1, 0);
	if(LUA_OK == callTestStatus) {
		int ret = lua_tointeger(luaContext, 1)
		return ret;
	} else {
		const char* errMsg = lua_tostring(luaContext, 1)
		Log_e(LOG_TAG, "%s", errMsg);
		return -1;
	}
}
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;流程如注释。在这一个过程中，Lua虚拟栈的内容变化如图，从**luaL_loadbuffer**开始：
{% asset_img lua_stack_content_change.png %}
&nbsp;&nbsp;&nbsp;&nbsp;首先，经过luaL\_loadbuffer之后，luaL\_loadbuffer会把传过来的\*.lua文件的buffer作为一个Lua Chunk，接着编译它。编译完后，把编译结果包装成一个function并推入Lua虚拟栈中。经过lua\_pcall后，Lua虚拟机会把所执行的function及其参数从Lua虚拟栈中弹出。接着，通过lua\_getglobal获取Lua层的全局变量「test」，lua\_getglobal会把这个变量的值推入Lua虚拟栈中。函数已经准备好，再经过lua\_pushinteger(a)和lua\_pushinteger(b)后，函数和参数都已经顺序推入了，调用lua\_pcall的先决条件已经满足。接下来，调用lua\_pcall后，Lua虚拟机会根据调用lua\_pcall是传入的nresults，将结果推入Lua虚拟栈中。最后，我们只需要lua_tointeger(index)来获取执行结果，返回给Android层即可。可以看到，自始至终，Lua虚拟栈充当一个数据交换的桥梁，是一个十分重要的角色。

&nbsp;&nbsp;&nbsp;&nbsp;接下来，只需要在Native层Register一下NativeMethods，并在Android层声明一下native方法就可以使用了。
{% codeblock lang:Kotlin %}
class LuaExecutor {
    init {
        System.loadLibrary("luabridge")
    }

    external fun startScript(luaString: String): Boolean
}
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;然而，上面的实现只有启动脚本的功能。在实际中，我们总不可能启动脚本之后，就没有对脚本执行流程有一点控制吧。因此有必要加一个停止脚本的功能。如何停止正在执行的脚本？先来看看Lua提供的C API：
* int luaL\_error (lua\_State \*L, const char \*fmt, ...);
     
     抛出一个异常，错误信息为fmt。

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | fmt |    const char\* | 错误信息   |

*    int lua\_sethook (lua\_State \*L, lua\_Hook f, int mask, int count);

     设置一个钩子函数。

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | f |    lua_Hook | 钩子函数，包含需要执行的语句   | 
     | mask |    int | 指定被调用的时机，取值为常量LUA_MASKCALL，LUA_MASKRET，LUA_MASKLINE和LUA_MASKCOUNT的按位或。   | 

     | mask取值   |      说明      |
     |:----------:|-------------|
     | LUA_MASKCALL |  代表钩子函数f会在进入任意函数后执行 |
     | LUA_MASKRET |    代表钩子函数在退出任意函数前执行   | 
     | LUA_MASKLINE |    代表钩子函数f会在执行函数内一行代码前执行   | 
     | LUA_MASKCOUNT |    代表钩子函数f会在lua解释器执行了count条指令后执行   | 

&nbsp;&nbsp;&nbsp;&nbsp;有了这两个C API，脚本的停止功能就可以实现了：

{% codeblock lang:C %}
void stopLuaHooker(lua_State *L, lua_Debug *ar) {
    luaL_error(L, "quit Lua");
}

void forceStopLua(lua_State *L) {
    int mask = LUA_MASKCOUNT;
    lua_sethook(L, &stopLuaHooker, mask, 1);
}
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;当我们调用forceStopLua时，会为Lua脚本的执行设置一个钩子函数，这个钩子函数的执行时机是：lua\_sethook执行之后，Lua解释器执行完一条指令时。也就是说，我们在Lua层代码执行到任意地方时调用forceStopLua后，Lua解释器会在执行完一条指令后，接着执行stopLuaHooker，进而执行lua_error，抛出异常，脚本即终止。因此，脚本的启动和停止的功能已经实现好了，封到一个类里，叫做LuaEngine：
{% codeblock lang:C LuaEngine.h %}
#ifndef ANDROIDLUA_LUAENGINE_H
#define ANDROIDLUA_LUAENGINE_H

#include <cstring>
#include <string>
#include <jni.h>
#include "lua/lua.hpp"

#include "utils/Log.h"
#include "JniManager.h"

#define LOG_TAG "LuaEngine"

class LuaEngine {
public:
    LuaEngine();

    virtual ~LuaEngine();

    lua_State *getScriptContext() {
        return mScriptContext;
    }

    bool startScript(jstring jBuff, const char *functionName);

    bool isScriptRunning() {
        return scriptRunning;
    }

    bool stopScript();

private:
    lua_State *mScriptContext;
    bool scriptRunning;

    bool loadBuff(jstring jBuff);

    bool runLuaFunction(const char *functionName);
};

void quitLuaThread(lua_State *L);

void quitLuaThreadHooker(lua_State *L, lua_Debug *ar);

#endif //ANDROIDLUA_LUAENGINE_H
{% endcodeblock %}

{% codeblock lang:C LuaEngine.cpp %}
#include "LuaEngine.h"

LuaEngine::LuaEngine() {
    mScriptContext = luaL_newstate();
    scriptRunning = false;
}

LuaEngine::~LuaEngine() {
    if (isScriptRunning()) {
        stopScript();
    }
    mScriptContext = nullptr;
}

bool LuaEngine::startScript(jstring jBuff, const char *functionName) {
    scriptRunning = true;
    luaL_openlibs(mScriptContext);
    if (this->loadBuff(jBuff)) {
        Log_d(LOG_TAG, "script start running..");
        bool success = this->runLuaFunction(functionName);
        scriptRunning = false;
        return success;
    } else {
        scriptRunning = false;
        return false;
    }
}

bool LuaEngine::stopScript() {
    if (scriptRunning) {
        quitLuaThread(mScriptContext);
        scriptRunning = false;
        return true;
    } else {
        Log_d(LOG_TAG, "script is Not running");
        return false;
    }
}

bool LuaEngine::loadBuff(jstring jBuff) {
    // 读取buff
    JNIEnv *env;
    JniManager::getInstance()->getJvm()->GetEnv((void **) &env, JNI_VERSION_1_6);
    const char *cBuff = env->GetStringUTFChars(jBuff, nullptr);
    if (LUA_OK != luaL_loadbuffer(mScriptContext, cBuff, strlen(cBuff), NULL)) {
        const char *szError = luaL_checkstring(mScriptContext, -1);
        Log_e(LOG_TAG, "%s", szError);
        return false;
    }
    // 加载buff到内存
    if (LUA_OK != lua_pcall(mScriptContext, 0, LUA_MULTRET, 0)) {
        const char *szError = luaL_checkstring(mScriptContext, -1);
        Log_e(LOG_TAG, "%s", szError);
        return false;
    }
    env->ReleaseStringUTFChars(jBuff, cBuff);
    env->DeleteGlobalRef(jBuff);
    return true;
}

bool LuaEngine::runLuaFunction(const char *functionName) {
    // 获取errorFunc
	// 错误由__TRACKBACK__来处理，可以用来打印错误信息，
	// __TRACKBACK__函数需要自己定义在lua脚本中
    lua_getglobal(mScriptContext, "__TRACKBACK__");
    if (lua_type(mScriptContext, -1) != LUA_TFUNCTION) {
        Log_d(LOG_TAG, "can not found errorFunc : __TRACKBACK__");
        return false;
    }
    int errfunc = lua_gettop(mScriptContext);

    // 获取指定的方法
    lua_getglobal(mScriptContext, functionName);
    if (lua_type(mScriptContext, -1) != LUA_TFUNCTION) {
        Log_d(LOG_TAG, "can not found func : %s", functionName);
        return false;
    }

    // 跑指定的方法
    return LUA_OK == lua_pcall(mScriptContext, 0, 0, errfunc);
}

void quitLuaThread(lua_State *L) {
    int mask = LUA_MASKCOUNT;
    lua_sethook(L, &quitLuaThreadHooker, mask, 1);
}

void quitLuaThreadHooker(lua_State *L, lua_Debug *ar) {
    luaL_error(L, "quit Lua");
}
{% endcodeblock %}

# 3. Lua单向调用Android
&nbsp;&nbsp;&nbsp;&nbsp;前面的实现，只允许Android层调用Lua的方法，而Lua层并不能调用Android层的方法。可不可以在Lua层调用Android层的方法？答案是可以的。一个思路是，Lua层调用Native层的方法，Native层再通过反射调用Android层的方法。先看看Lua层是怎么调用Native层的方法。Lua提供了一个C API：lua_register，它的原型是：
* void lua\_register (lua\_State \*L, const char \*name, lua\_CFunction f);

     注册一个CFunction。

     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | L |  lua_State\* | Lua的context |
     | name |    const char\* | Lua层全局变量的名称  | 
     | f |    lua_CFunction | C函数。原型是：int functionXXX(lua_State\* L);其返回值的意义代表返回结果的个数。   | 

&nbsp;&nbsp;&nbsp;&nbsp;我们可以用这个C API实现Lua层调用Native层的方法：
{% codeblock lang:C %}
lua_register(mScriptContext, "getString" , getString);

int getString(lua_State *L) {
    const char *cStr = "String From C Layer";
    lua_pushstring(L, cStr);
    return 1;
}
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;上面的代码很简单，先注册一个名字为getString的全局变量，指向C函数getString。C函数getString中，先声明并分配一个字符串cStr，再把这个字符串推入到Lua栈中，并返回结果个数。因此，在Lua层，如果执行getString()，则会得到字符串"String From C Layer"，Lua层就可以调用Native层的方法了。

&nbsp;&nbsp;&nbsp;&nbsp;然后看看Native层调用Android层的方法。代码如下：
{% codeblock lang:C %}
int getString(lua_State *L) {
	JNIEnv* env;
	g_pJvm->GetEnv((void **) &env, JNI_VERSION_1_6);
	
    jclass clazz = env->FindClass("com/zspirytus/androidlua/shell/ShellBridge");
	if (!clazz) {
        Log_d(LOG_TAG, "class not found!");
        return 0;
    }
	
    jmethodID methodId = env->GetStaticMethodID(clazz, "getStringFromKotlinLayer", "()Ljava/lang/String;");
    if (!methodId) {
        Log_d(LOG_TAG, "method %s not found!", "getStringFromStaticJavaMethod");
        return 0;
    }
	
    jstring jStr = (jstring) env->CallStaticObjectMethod(clazz, methodId);
	
    const char *cStr = env->GetStringUTFChars(jStr, NULL);
    lua_pushstring(L, cStr);
    env->ReleaseStringUTFChars(jStr, cStr);
    env->DeleteLocalRef(jStr);
    return 1;
}
{% endcodeblock %}
&nbsp;&nbsp;&nbsp;&nbsp;解释一下，首先通过在JNI_OnLoad保存下来的JavaVM指针指针获得Jni的环境变量，再用Jni的环境变量找到class和method，最后通过env、class和method反射调用Android层的方法获得返回的jstring，转成C-style的string后推入lua栈中，释放资源，并返回结果个数。

&nbsp;&nbsp;&nbsp;&nbsp;在Android层，留下一个方法以供调用：
{% codeblock lang:Kotlin %}
@Keep
object ShellBridge {

    private val TAG = ShellBridge.javaClass.simpleName

    @Keep
    @JvmStatic
    fun getStringFromKotlinLayer(): String {
        return "String From Android Layer"
    }
}
{% endcodeblock %}
至此，Android层与Lua层的交互已经实现了。

# 4. 避免ANR
&nbsp;&nbsp;&nbsp;&nbsp;然而上面的实现可能会导致ANR，原因在于Lua脚本的执行可能是耗时的。如果Lua脚本的执行时间超过5秒，必然ANR。一个解决方法是，把Lua脚本的执行放到子线程当中。这个子线程应当给Native层管理比较好，还是Android层管理比较好？我个人觉得放在Native层比较好，这样Android层就不需要专为执行Lua脚本而新建和管理线程，代码就不会太复杂；即使Native层的逻辑比较复杂，编好了so，一般就会当做一个库来使用，而不会去动它。所以，还是在Native层创建和管理线程。
pthread_create是Unix、Linux等系统创建线程的函数，它的原型是：
* int pthread\_create(pthread\_t \*restrict tidp, const pthread\_attr\_t \*restrict attr, void \*(\*start_rtn)(void \*), void \*restrict arg);
     
     | 参数   | 类型 |      说明      |
     |:----------:|:----------:|-------------|
     | tidp |  pthread_t \*restrict | 线程ID |
     | attr |    const pthread_attr_t\*restrict | 线程属性，默认为NULL  | 
     | \*(\*start_rtn)(void \*) |    void | 运行在新线程的函数   |
     | \*restrict arg |  void | start_rtn的所需参数 | 

&nbsp;&nbsp;&nbsp;&nbsp;因此，我们可以把执行Lua脚本的逻辑移到新线程中：
{% codeblock lang:C %}
void startWork() {
    pthread_create(&mThreadId, NULL, &startWorkInner, (void*)this);
}

void stopWork() {
    stopScript();
    mThreadId = 0;
}

void* startWorkInner(void *args) {
    startScript();
    return nullptr;
}
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;这样，startScript()就运行在新线程中，就不会有ANR的风险。我们把它封到一个类中，叫LuaTask，一次Lua脚本的开始与结束，都由这个类来管理。
{% codeblock lang:C LuaTask.h %}
#ifndef ANDROIDLUA_LUATASK_H
#define ANDROIDLUA_LUATASK_H

#include <sys/types.h>
#include <pthread.h>
#include <jni.h>

#include "LuaEngine.h"

class LuaTask {

public:
    LuaTask(jstring jBuff);

    virtual ~LuaTask();

    void startWork();

    void stopWork();

    bool isRunning();

private:
    static void *startWorkInner(void *args);

private:
    jstring mLuaBuff;
    pthread_t mThreadId;
    LuaEngine *mLuaEngine;
};

#endif //ANDROIDLUA_LUATASK_H
{% endcodeblock %}

{% codeblock lang:C LuaTask.cpp %}
#include "LuaTask.h"

LuaTask::LuaTask(jstring jBuff) {
    mLuaBuff = jBuff;
    mLuaEngine = new LuaEngine();
    mThreadId = 0;
}

LuaTask::~LuaTask() {
    delete mLuaEngine;
}

void LuaTask::startWork() {
    pthread_create(&mThreadId, NULL, &LuaTask::startWorkInner, (void*)this);
}

void LuaTask::stopWork() {
    mLuaEngine->stopScript();
    mThreadId = 0;
}

void* LuaTask::startWorkInner(void *args) {
    LuaTask* task = (LuaTask*) args;
    task->mLuaEngine->startScript(task->mLuaBuff, "main");
    return nullptr;
}

bool LuaTask::isRunning() {
    return mThreadId != 0;
}
{% endcodeblock %}

&nbsp;&nbsp;&nbsp;&nbsp;但是，这是我们新创建的线程，还没有attach到JavaVM。如果没有attach到JavaVM，就会找不到JNIEnv，所以必须要attach到JavaVM，这样才能拿到JavaVM的JNI环境变量，从而可以调用到Android层的方法。因此startWorkInner要改进一下：
{% codeblock lang:C %}
void* startWorkInner(void *args) {
    JNIEnv* env = nullptr;
    JavaVMAttachArgs args{JNI_VERSION_1_6, nullptr, nullptr};
    g_pJvm->AttachCurrentThread(&env, &args);
    startScript()
    g_pJvm->DetachCurrentThread();
    return nullptr;
}
{% endcodeblock %}
&nbsp;&nbsp;&nbsp;&nbsp;线程退出之前，记得要和JavaVM detach一下，这样线程才能正常退出。

# 5. 运行脚本包
&nbsp;&nbsp;&nbsp;&nbsp;至此，我们完成了能够随时开始、停止，出错能打印堆栈信息的执行Lua脚本功能。但实际上，我们不可能只跑单个脚本，并且脚本可能需要一些资源文件。因此我们一般会把脚本和资源文件打包成一个脚本包。在运行脚本之前，先解包，把脚本解析出来后再运行。
所以这个解析脚本的逻辑放在Native层还是Android层？我个人觉得放在Android层比较好。有两点原因：
1. 脚本包格式不确定，Native层不可能为每种情况进行适配，既然如此那就交由使用者来解析。
2. 单一职责的原则，Native层负责还是只负责一种功能比较好。而且为解析脚本包而重新编译一个so文件又太小题大做，所以解析的任务就交给使用者吧。

既然提到脚本包，我就简单谈谈我的实现。我的实现是把lua脚本和资源文件一起压缩成一个zip文件，在zip文件中有一个config文件，里面写好了所有lua脚本的相对路径。在解析的时候，先在内存中把config解压出来，读出所有lua脚本的相对路径，然后在内存中把所有lua脚本文件都解压出来后，拼接起来，在交给Native层运行。至于资源文件，根据脚本的运行情况进行动态解压。我简单的封装了一下：
```
private external fun startScript(luaString: String): Boolean
external fun stopScript(): Boolean
external fun isScriptRunning(): Boolean

fun runScriptPkg(scriptPkg: File, configFile: String) {
    mThreadPool?.execute {
        val start = System.currentTimeMillis()
        initScriptPkg(scriptPkg)
        val zipFile = ZipFile(scriptPkg)
        val config = ZipFileUtils.getFileContentFromZipFile(zipFile, configFile)
        val luaScriptPaths = config.split("\r\n")
        val luaScript = ZipFileUtils.getFilesContentFromZipFile(zipFile, luaScriptPaths)
        Log.d("USE_TIME", "${System.currentTimeMillis() - start} ms")
        mHandler?.post {
            startScript(luaScript)
        }
    }
}

object ZipFileUtils {

    fun getFileContentFromZipFile(zipFile: ZipFile, targetFile: String): String {
        var ins: InputStream? = null
        try {
            val ze = zipFile.getEntry(targetFile)
            return if (ze != null) {
                ins = zipFile.getInputStream(ze)
                FileUtils.readInputStream(ins)
            } else {
                ""
            }
        } finally {
            ins?.close()
        }
    }

    fun getFilesContentFromZipFile(zipFile: ZipFile, targetFiles: List<String>): String {
        val stringBuilder = StringBuilder()
        targetFiles.filter { it.isNotEmpty() and it.isNotBlank() }.forEach {
            val content = getFileContentFromZipFile(zipFile, it)
            stringBuilder.append(content).append('\n')
        }
        return stringBuilder.toString()
    }
}

object FileUtils {

    fun readInputStream(ins: InputStream): String {
        return ins.bufferedReader().use(BufferedReader::readText)
    }
}
```

&nbsp;&nbsp;&nbsp;&nbsp;至此，我们在原有功能的基础上，增加了跑脚本包的功能。完整的代码可以看[这里](https://github.com/zkw012300/AndroidLua)。

# 6. 总结
{% asset_img Android_call_lua.png %}

{% asset_img lua_call_android.png %}

# 7. 感想
&nbsp;&nbsp;&nbsp;&nbsp;Android跑Lua脚本这个过程其实是很简单的，不是主要难点。这次主要卡住的地方是在JNI部分，因为我发现我所了解的C语言语法太古老了，跟不上现在的C语言。虽然我的C语言的代码量也不多，加上我对JNI的一些编程规范不太了解，所以一路磕磕绊绊，但是总算是写出来了。Kotlin和C/C++还是要多熟悉熟悉，多练练。



















