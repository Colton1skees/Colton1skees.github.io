---
layout: default
title: "How To Reverse Programs That Use Lua"
---

# Lua

Lua is a lightweight embeddable scripting language that is commonly used in a variety of applications. Because Lua has C bindings, it is very easy to embed. As a result of this, Lua is widely used in many applications(especially games). Lua's C bindings can be found [here](https://pgl.yoyo.org/Luai/i/3.7+Functions+and+Types).

# Reversing Lua's C API

For the rest of this writeup I'm going to be using IDA Pro. This tutorial assumes a basic knowledge of both IDA(or another disassembler) and the Lua programming language.

If you click on the link above in the Lua section, you'll see a list of Lua's C API Functions. They are basically just functions you can call to interact/manipulate Lua through C. The easiest way that I've found to locate specific C API function's is to just download the Lua source code and look for string references. You can find the Lua version that the program uses's by searching for the string
`$Lua`

![Branching](https://i.imgur.com/dqaqao2.png)

Once you've figured out what Lua version the program uses, download the source code and open it up in a text editor. At this point you can just use strings to locate any C API function. I'm using Lua version 5.1.1, but the process doesn't change much throughout versions.

### Executing custom lua scripts

In order to execute custom lua scripts, you need to find two functions: luaL_loadbuffer and lua_pcall.

If you open up the lua source code and look at the Lua method "db_debug", you'll see that it calls both luaL_loadbuffer and lua_pcall.
![Branching](https://i.imgur.com/Td7NSTR.png)

Open up a disassembler and cross reference "=(debug command)". You should get 1 result, which will bring you inside of db_debug.
![Branching](https://i.imgur.com/C2lOgIO.png)

If you look at line 22, you'll see luaL_loadbuffer and lua_pcall being called. With those two functions alone you can execute any custom Lua script(see [here](http://gouthamanbalaraman.com/blog/minimal-example-lua-function-cpp.html))

```cpp
if ( sub_35548930(a1, &v9, strlen(&v9), "=(debug command)") || sub_35547920(a1, 0, 0, 0) )
```

I'm not going to cover finding any more lua API functions because the process is mostly the same for every lua method.

# Interacting with Lua scripts

### Dumping Lua Scripts

I'm not entirely sure if there are other ways to load Lua scripts, but in every program I've seen, Lua scripts are loaded through luaL_loadbuffer. You can dump Lua scripts by hooking luaL_loadbuffer and writing scripts to a file.

The typedef for luaL_loadbuffer is

```cpp
typedef int(__cdecl* tlua_loadbuffer)(int* lua_State, const char* buff, size_t sz, const char* name);
```

Here is an example of dumping lua scripts from a hooked luaL_loadbuffer method(note that I use int* in place of lua_State* because I personally have no need to reverse lua_State):

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

Adding this code to your luaL_loadbuffer hook allows you to dump Lua scripts. However, this code might not work in every situation. luaL_loadbufffer supports passing a file buffer OR plain Lua text in the buff parameter. You could end up crashing if you try to create a file with plain text passed in the buff parameter. I haven't ran into this issue yet, so I can't provide a definitive solution.

One solution could be to check the size of the buffer with strlen. If the buffer stores the bytes of a lua file, then it should have a length of 5 and the text "LuaQ" when the buffer is converted to a string.

```cpp
if(strlen(buff) == 5 && strcmp(buff, "LuaQ") == 0)
```

Adding that check to your hook should work, but it might not work in every case.

Another thing to note is that some programs pass the files name(if applicable) in the description parameter, so you might be able to assign files their real name.

If you successfully dump a program's Lua files and notice they look like garbage in a text editor, then it usually means the lua scripts are compiled. You can use a tool called unluac to decompile those scripts and retrieve the source code if that is the case.

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

The code above is an example of how you could replace a script in memory. The only issue is that it assumes the script name is passed in the description parameter, and that the script is being loaded from a file. If thats not the case then you'll have to find some way to identify the script you're looking to modify(possibly use the size parameter)

If the script is not being loaded from a file, then one option would be to do some string processing on the buffer parameter to modify/parse its contents.

# Using the Lua C API

Assuming you've found whatever API function's you want to use, the biggest issue to tackle is dealing with multithreading/Lua states. If you're not familiar with what a Lua state is then see [here](https://stackoverflow.com/a/4201531)

When I first started reversing apps that use Lua, I always had an issue with random crashes when trying to execute API functions. In my case, the issue was that the program I was reversing had multiple Lua states, and was multi-threaded.

Multi-threading is relevant to Lua because it is a stack based language. Here's an example issue you might run into if threading is a problem:

Imagine you're trying to call a method, so you push a parameter on the stack. If the program is multithreaded and you're not making the API calls in the Lua thread, then there's a chance that another thread could push something else onto the stack before you can call the method that uses the parameter. This would corrupt the stack causing the program to crash.

When I ran into this problem, I remember reading some post saying that I had to "find the program's internal Lua locking mechanisms". In my experience, I've never actually had to do that.

The solution is simple: hook any Lua API function that gets executed often, and execute whatever API function's you want in your hook. Generally I hook lua_pcall. Some programs don't use lua_pcall though, so you may have to hook a different method(such as lua_gettop).

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

##### Things to note

1. The variable bExecuteCustomLuaCode gives you control of when you want to execute code. The variable can be removed so long as you have proper control flow to determine when you want to execute custom lua code.
2. Because sleep() cannot be used in your hook(unless you want to block the entire thread that lua is executing in), I recommend using the clock_t class. The clock_t class allows you to easily keep track of time between function calls, and can be used for some very hacky solutions.
3. If you aren't targetting a specific Lua State, then you can just remove the code in relation to setting the target lua state. However, you have to make sure you're executing all the API functions on the same Lua State. Assuming it doesn't matter which Lua State you use, I reccomend creating a global variable to store whichever lua state you use(which you can just choose randomly in your hook) and executing all API function's in that lua state.
4. Additionally, if the program you're reversing only has one lua state, then you can just remove all the code relating to using a specific lua state. Sadly that's not the case in all programs, so its just a matter of luck.

# Closing Thoughts

The Lua API is generally pretty easy to reverse. Alot of programs that embed Lua implement a large portion of their functionality in Lua(even going as far as making C functions callable through lua). Whether or not reversing the lua API is worth it depends on what functionality they implement in lua, and what functionality you are looking for.

Here are my signatures for some of lua's C API functions(version 5.1.1):

```cpp
PBYTE sigGettop = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x08\x8B\x41\x08\x2B\x41\x0C";
char* maskGettop = "xxxxxxxxxxxx";

PBYTE sigGetfield = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x0C\x83\xEC\x08\x53\x56\x8B\x75\x08\x57\x8B\xD6\xE8\x00\x00\x00\x00\x8B\x55\x10\x8B\xF8\x8B\xC2\x8D\x58\x01\x8A\x08\x40\x84\xC9\x75\xF9\x2B\xC3\x50\x52\x56\xE8\x00\x00\x00\x00\x89\x45\xF8\x8B\x46\x08\x50";
char* maskGetfield = "xxxxxxxxxxxxxxxxxx????xxxxxxxxxxxxxxxxxxxxxxx????xxxxxxx";

PBYTE sigPushnumber = (PBYTE)"\x55\x8B\xEC\x8B\x45\x08\x8B\x48\x08\xF3\x0F\x10\x45";
char* maskPushnumber = "xxxxxxxxxxxxx";

PBYTE sigPcall = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x14\x83\xEC\x08";
char* maskPcall = "xxxxxxxxx";

PBYTE sigPushstring = (PBYTE)"\x55\x8B\xEC\x8B\x45\x0C\x85\xC0";
char* maskPushstring = "xxxxxxxx";

PBYTE sigPushboolean = (PBYTE)"\x55\x8B\xEC\x8B\x45\x08\x8B\x48\x08\x33\xD2";
char* maskPushboolean = "xxxxxxxxxxx";

PBYTE sigLoadbuffer = (PBYTE)"\x55\x8B\xEC\x83\xEC\x08\x8B\x45\x0C\x8B\x55\x14";
char* maskLoadbuffer = "xxxxxxxxxxxx";

PBYTE sigLuatype = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x0C\x8B\x55\x08\xE8\x00\x00\x00\x00\x3D\x00\x00\x00\x00\x75\x05";
char* maskLuatype = "xxxxxxxxxx????x????xx";

PBYTE sigTonumber = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x0C\x8B\x55\x08\x83\xEC\x08\xE8\x00\x00\x00\x00\x83\x78\x04\x03\x74\x17";
char* maskTonumber = "xxxxxxxxxxxxx????xxxxxx";

PBYTE sigPushnil = (PBYTE)"\x55\x8B\xEC\x8B\x45\x08\x8B\x48\x08\xC7\x41";
char* maskPushnil = "xxxxxxxxxxx";

PBYTE sigPushvalue = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x0C\x56\x8B\x75\x08\x8B\xD6\xE8\x00\x00\x00\x00\x8B\x4E\x08\x8B\x10\x89\x11";
char* maskPushvalue = "xxxxxxxxxxxxx????xxxxxxx";

PBYTE sigNext = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x0C\x56\x8B\x75\x08\x8B\xD6\xE8\x00\x00\x00\x00\x8B\x4E\x08\x8B\x10\x83\xE9\x08\x51\x52\x56";
char* maskNext = "xxxxxxxxxxxxx????xxxxxxxxxxx";

PBYTE sigSettop = (PBYTE)"\x55\x8B\xEC\x8B\x4D\x0C\x8B\x45\x08";
char* maskSettop = "xxxxxxxxx";

PBYTE sigToString = (PBYTE)"\x55\x8B\xEC\x56\x8B\x75\x08\x57\x8B\x7D\x0C\x8B\xCF";
char* maskToString = "xxxxxxxxxxxxx";
```

The signatures should work for other versions of lua, but they could break in older/newer versions of lua.
