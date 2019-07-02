---
layout: default
title: "How To Reverse Programs That Use Lua"
---

# Lua

Lua is a lightweight embeddable scripting language that is commonly used in a variety of applications. Because Lua has C/C++ bindings, it is very easy to embed. As a result of this, Lua is widely used in many applications(especially games). Lua's C++ bindings can be found [here](https://pgl.yoyo.org/Luai/i/3.7+Functions+and+Types).

# Reversing Lua's C API

For the rest of this tutorial I'm going to be using IDA Pro. This tutorial assumes a basic knowledge of the Lua programming language.

If you click on the link above in the Lua section, you'll see a list of Lua's C API Functions. They are basically just functions you can call to interact/manipulate Lua through C/C++. The easiest way that I've found to locate specific C API function's is to just download the Lua source code and look for string references. You can find the Lua version that the program uses's by searching for the string
`$Lua`

![Branching](https://i.imgur.com/dqaqao2.png)

Once you've figured out what Lua version the program uses, download the source code and open it up in a text editor. At this point you can just use strings to locate any C API function. I'm using Lua version 5.1.1, but the process doesn't change much throughout versions.

### How to find luaL_loadbuffer and lua_pcall

If you look at the Lua method "db_debug", you'll see that it calls luaL_loadbuffer and lua_pcall.
![Branching](https://i.imgur.com/Td7NSTR.png)

Open up a disassembler and cross reference "=(debug command)". You should get 1 result, which will bring you inside of db_debug.
![Branching](https://i.imgur.com/C2lOgIO.png)

If you look at this line, you'll see luaL_loadbuffer and lua_pcall being called. With those two functions alone you can execute any custom Lua script(see [here](http://gouthamanbalaraman.com/blog/minimal-example-lua-function-cpp.html))

```cpp
if ( sub_35548930(a1, &v9, strlen(&v9), "=(debug command)") || sub_35547920(a1, 0, 0, 0) )
```

I'm not going to cover finding any more lua API functions because the process is the same for every lua method.

# Interacting with Lua scripts

### Dumping Lua Scripts

I'm not entirely sure if there are other ways to load Lua scripts, but in every program I've seen, Lua scripts are loaded through luaL_loadbuffer. You can dump Lua scripts by hooking luaL_loadbuffer and writing scripts to a file.

The typedef for luaL_loadbuffer is

```cpp
typedef int(__cdecl* tlua_loadbuffer)(int* lua_State, const char* buff, size_t sz, const char* name);
```

I'm not going to provide any code for hooking luaL_loadbuffer(because it depends on which library you use), but I will provide some code for dumping Lua scripts.

Note that I use int* in place of lua_State* because I personally have no need to reverse lua_State.

```cpp
static int fileCount;
std::string path = "Insert A Path to where you want to save the file"

...

int __cdecl Hooked_LoadBuffer(int* lua_State, const char* buff, size_t sz, const char* description)
{
    std::ofstream outfile(path + "script" + std::to_string(fileCount), std::ofstream::binary);
	outfile.write(buff, sz);
	outfile.close();
}
```

Adding this code to your luaL_loadbuffer hooks allows you to dump Lua scripts. However, there is a problem with the method above. luaL_loadbufffer supports passing a file buffer OR plain Lua text in the buff parameter. You will likely crash if you try to create a file with plain text passed in the buff parameter. I haven't ran into this issue yet, so I can't provide a definitive solution.

One solution could be to check the size of the buffer with strlen. If the script is compiled it should have a length of 5 and the text "LuaQ" when converted to a string.

```cpp
if(strlen(buff) == 5 && strcmp(buff, "LuaQ") == 0)
```

Adding that check to your hook should work, but it might not work in every case.

Another thing to note is that some programs pass the files name(if applicable) in the description parameter, so you might be able to assign files their real name.

### Modifying/Replacing Lua scripts in memory

If you a script gets loaded from a file in luaL_loadbuffer, then you can just swap the buffer and size in your hook.

```cpp
int __cdecl Hooked_LoadBuffer(int* lua_State, const char* buff, size_t sz, const char* description)
{
    if(strcmp(description, "SomeScriptYouWantToReplace") == 0)
    {
        std::ifstream infile("ModifiedScriptPath", std::ofstream::binary);
        infile.seekg(0, infile.end);
        size_t size = infile.tellg();
        infile.seekg(0);
        char* buffer = new char[size];
        infile.read(buffer, size);
        return Original_LoadBuffer(lua_State, buffer, size, description)
    }

    return Original_LoadBuffer(lua_State, buff, sz, description) // Original_LoadBuffer is the original loadbuffer function that isn't hooked.
}
```

The code above is an example of how you could replace a script in memory. The only issue is that it assumes the script name is passed in the description parameter, and that the script is being loaded from a file. If thats not the case then you'll have to find some way to identify the script you're looking to modify(I would typically use the size parameter)

If the script is not being loaded from a file, then you'd likely have to do some string processing on the buffer parameter to modify/parse its contents.

# Using the Lua C API

Assuming you've found whatever API function's you want to use, the biggest issue to tackle is dealing with multithreading/Lua states. If you're not familiar with what a Lua state is then see [here](https://stackoverflow.com/a/4201531)

When I first started reversing apps that use Lua, I always had an issue with random crashes when trying to execute API functions. In my case, the issue was that the program I was reversing had multiple Lua states, and was multi-threaded.

Multi-threading is relevant to Lua because it is a stack based language. Here's an example issue you might run into if threading is a problem:

Imagine you're trying to call a method, so you push a parameter on the stack. If the program is multithreaded and you're not making the API calls in the Lua thread, then there's a chance that something else could get pushed onto the stack before you can call the method that uses the parameter. This would corrupt the stack causing the program to crash.

When I ran into this problem, I remember reading some post saying that I had to "find the program's internal Lua locking mechanisms". In my experience, I've never actually had to do that.

The solution is simple: hook any Lua API function that gets executed often, and execute whatever API function's you want in your hook. Generally I use lua_pcall. Some programs don't use lua_pcall though, so you may have to hook a different method(such as lua_gettop).

The only other issue is figuring out which Lua state to use. If a program uses multiple Lua states then you have to figure out which one you want to use. If there is a specific Lua state you need to use, then you just have to find a global value specific to that state.
Here is an example of executing your own Lua code in a hooked function.

```cpp
bool bExecuteCustomLuaCode = false;
int* targetLuaState;
...

int __cdecl Hooked_LuaPCall(int* lua_State, int nargs, int nresults, int errFunc)
{

    if(targetLuaState == 0) // checks if you've found the target lua state
    {
        lua_getglobal(lua_State, "SomeGlobalValueExclusiveToTheTargetLuaState"); // pushes the global, or nil if the state does not have the global
		if (!lua_isnil(lua_State, -1)) // sets the target Lua state if it was found
		{
            targetLuaState = lua_State;
		}

        lua_pop(lua_State, 1);
    }

    if(targetLuaState != 0 && bExecuteCustomLuaCode)
    {
        bExecuteCustomLuaCode = false; // setting this variable to false prevents a stack overflow incase you call any API methods that call PCall
        // execute your custom Lua code here
    }

    return Original_LuaPCall(lua_State, nargs, nresults, errFunc)
}
```

If you aren't targetting a specific Lua State, then you can just remove the code in relation to setting the target lua state. However, you have to make sure you're executing all the API functions on the same Lua State. Assuming it doesn't matter which Lua State you use, I reccomend creating a global variable to store whichever lua state you use(which you can just choose randomly in your hook) and executing all API function's in that lua state.
