---
layout: default
title: "Reverse Engineering Programs That Use Lua"
---

# Lua

Lua is a lightweight embeddable scripting language that is commonly used in a variety of applications. Because lua has C/C++ bindings, it is very easy to embed. As a result of this(and ofcourse, lua's speed and short learning curve) it is widely used in many applications(especially games). Lua's C++ bindings can be found [here](https://pgl.yoyo.org/luai/i/3.7+Functions+and+Types).

Most of this information will be targeted torwards reverse engineering game's that use lua, but it can be applied to any application that uses lua.

# Reversing Lua's C API

For the rest of this tutorial I'm going to be using IDA Pro. I'm not going to explain basic lua concepts to keep the tutorial short.

If you click on the link above, you'll see a list of lua's C API Functions. They are basically just functions you can call to interact/manipulate lua through C/C++. The easiest way that I've found to locate specific C API function's is to just download the lua source code and look for string references. You can find the lua version that the game uses's by searching for the string `$Lua`
