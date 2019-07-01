---
layout: default
title: "Reverse Engineering Programs That Use Lua"
---

# Lua

Lua is a lightweight embeddable scripting language that is commonly used in a variety of applications. Because lua has C/C++ bindings, it is very easy to embed. As a result of this, lua is widely used in many applications(especially games). Lua's C++ bindings can be found [here](https://pgl.yoyo.org/luai/i/3.7+Functions+and+Types).

Most of this information will be targeted torwards reverse engineering game's that use lua, but it can be applied to any application that uses lua.

# Reversing Lua's C API

For the rest of this tutorial I'm going to be using IDA Pro. I'm not going to explain basic lua concepts because it is out of the scope of this tutorial.

If you click on the link above, you'll see a list of lua's C API Functions. They are basically just functions you can call to interact/manipulate lua through C/C++. The easiest way that I've found to locate specific C API function's is to just download the lua source code and look for string references. You can find the lua version that the game uses's by searching for the string
`$Lua`

![Branching](https://i.imgur.com/dqaqao2.png)

Once you've figured out what lua version the game uses, download the source code and open it up in a text editor. At this point you can just use strings to locate any C API function. I'm using lua version 5.1.1, but the process doesn't change much throughout versions.

### How to find luaL_loadbuffer and lua_pcall

![Branching](https://i.imgur.com/Td7NSTR.png)
If you look at the lua method "db_debug", you'll see that it calls luaL_loadbuffer and lua_pcall.

![Branching](https://i.imgur.com/C2lOgIO.png)
If you cross reference "=(debug command)" you'll be brought here

```cpp
if ( sub_35548930(a1, &v9, strlen(&v9), "=(debug command)") || sub_35547920(a1, 0, 0, 0) )
```

If you look at this that line, you'll see luaL_loadbuffer and lua_pcall. With those two functions alone you can execute any custom lua script(See [here](http://gouthamanbalaraman.com/blog/minimal-example-lua-function-cpp.html))
