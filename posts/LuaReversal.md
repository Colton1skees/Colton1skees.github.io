---
layout: default
title: "Reverse Engineering Programs That Use Lua"
---

# Lua

Lua is a lightweight embeddable scripting language that is commonly used in a variety of applications. Because lua has C/C++ bindings, it is very easy to embed. As a result of this, lua is widely used in many applications(especially games). Lua's C++ bindings can be found [here](https://pgl.yoyo.org/luai/i/3.7+Functions+and+Types).

Most of this information will be targeted torwards reverse engineering game's that use lua, but it can be applied to any application that uses lua.

# Reversing Lua's C API

For the rest of this tutorial I'm going to be using IDA Pro. I'm not going to explain basic lua concepts because it is out of the scope of this tutorial.

If you click on the link above, you'll see a list of lua's C API Functions. They are basically just functions you can call to interact/manipulate lua through C/C++. The easiest way that I've found to locate specific C API function's is to just download the lua source code and look for string references. You can find the lua version that the program uses's by searching for the string
`$Lua`

![Branching](https://i.imgur.com/dqaqao2.png)

Once you've figured out what lua version the program uses, download the source code and open it up in a text editor. At this point you can just use strings to locate any C API function. I'm using lua version 5.1.1, but the process doesn't change much throughout versions.

### How to find luaL_loadbuffer and lua_pcall

If you look at the lua method "db_debug", you'll see that it calls luaL_loadbuffer and lua_pcall.
![Branching](https://i.imgur.com/Td7NSTR.png)

If you cross reference "=(debug command)" you'll be brought here
![Branching](https://i.imgur.com/C2lOgIO.png)

If you look at this line, you'll see luaL_loadbuffer and lua_pcall. With those two functions alone you can execute any custom lua script(see [here](http://gouthamanbalaraman.com/blog/minimal-example-lua-function-cpp.html))

```cpp
if ( sub_35548930(a1, &v9, strlen(&v9), "=(debug command)") || sub_35547920(a1, 0, 0, 0) )
```

I'm not going to cover finding any more lua API functions because the process is the same for every lua method.

# Interacting with lua scripts

### Dumping Lua Scripts

I'm not entirely sure if there are other ways to load lua scripts, but in every program I've seen, lua scripts are loaded through luaL_loadbuffer. You can dump lua scripts by hooking luaL_loadbuffer and logging scripts to a file.

The typedef for luaL_loadbuffer is

```cpp
typedef int(__cdecl* tLua_loadbuffer)(int* lua_State, const char* buff, size_t sz, const char* name);
```

I'm not going to provide any code for hooking luaL_loadbuffer(because it depends on which library you use), but I will provide some code for dumping lua scripts.

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

However, there is a problem with that method above. luaL_loadbufffer supports passing a file buffer or plain lua text in the buff parameter. You will likely crash if you try to create a file with plain text as the bytes. I haven't ran into this issue yet, so I'm not quite sure how to avoid that issue.

One solution could be to check the size of the buffer with strlen. If the script is compiled it should have a length of 5 and the text "LuaQ" when converted to a string.

```cpp
if(strlen(buff) == 5 && strcmp(buff, "LuaQ") == 0)
```

Adding that check should work, but it might not work in every program.

One thing to note is that some programs pass the files name(if applicable) in the description parameter, so you might be able to assign files their real name.

### Modifying/Replacing lua scripts in memory

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
}
```

The code above is an example of how you could replace a script in memory. The only issue is that it assumes the script name is passed in the description parameter, and that the script is being loaded from a file. If thats not the case then you'll have to find some way to identify the script you're looking to modify(I would typically use the size parameter)

If the script is not being loaded from a file, then you'd likely have to do some string processing on the buffer parameter to modify/parse its contents.
