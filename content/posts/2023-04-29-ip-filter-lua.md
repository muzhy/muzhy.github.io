---
categories: lua ipv4 ip
date: "2023-04-29T00:00:00Z"
title: lua实现简单的ip4过滤
---
# lua 实现简单的ip4过滤
最近参与了一个使用openresty开发的服务，有个需求是要某些接口只能允许指定的ip进行访问，在`nginx.conf`进行配置虽然也能做到限定ip访问，但是却需要重启服务。需求是要求能在界面上配置后不用重启就生效，并且由于可能要允许通过的ip地址会很多，希望能支持掩码或网络段的配置。

场景目前只要求支持ipv4，实现起来也比较简单，后续有需要再考虑支持ipv6。

## 支持的匹配方式
需要支持的匹配方式有以下四种
1. ip地址： 即配置一个ip，只允许该ip进行访问
2. 掩码模式：`192.168.0.1/16`这种方式
3. 通配符模式：`192.168.201.*`，对于前面是192.168.201的ip都要允许访问，`*`在最后面的时候看起来和掩码效果类型，但还要能支持类似这种`192.168.*.201`的情况
4. 网络段模式：`192.168.100-200.201`，对于ip地址中第三部分中的值在`100`和`200`之间，且其余部分匹配的ip地址要允许访问

对于以上四种方式，第三种和第四种可以在一个模式串中的不同段一起存在，即类似这种:`192.168.100-200.*`的格式是要能支持的。将这两种合并称为网络段匹配模式

由于掩码与网络段匹配模式存在冲突，因此同一模式串只能支持掩码或网络段匹配其中一种，当两种都存在时，忽略掩码，此时只有网络端的配置生效，即`192.168.100-200.201`的效果与`192.168.100-200.201/16`一样。

## 掩码匹配方式的实现
关于子网掩码时已经明确定义好的概念，在实现上可以通过对ip和模式串进行比较位运算来比较得到

对于所给的模式串，需要先计算出对应的用数字表示的ip地址
```lua
local bit = require("bit")

local function str_trim(str)
    return str:match'^()%s*$' and '' or str:match'^%s*(.*%S)'
end

local function split_str_to_array(str, sep, trim)
    if not str or not sep then
        return nil
    end

    local arr = {}

    for s in str:gmatch("([^" .. sep .. "]+)") do
        if trim then 
            table.insert(arr, str_trim(s))
        else
            table.insert(arr, s)
        end
    end

    return arr
end

local function ip4_str_to_num(ip, ip_part_arr)
    if not ip then
        return nil, "no ip"
    end

    if not ip_part_arr then 
        ip_part_arr = split_str_to_array(ip, ".", true)
    end

    if not ip_part_arr or #ip_part_arr ~= 4 then
        return nil, "ip format error"
    end

    local ip_num = 0
    for _, part in ipairs(ip_part_arr) do
        local num = tonumber(part)
        if not num then
            return nil, "ip format error"
        end
        ip_num = bit.bor(bit.lshift(ip_num, 8), num)
    end

    return ip_num, nil
end

local pattern_split_result = split_str_to_array(pattern, '/')
if not pattern_split_result or #pattern_split_result ~= 2 then
    error("pattern format error")
    return nil
end
local pattern_ip_num, err = ip4_str_to_num(pattern_split_result[1], nil)
if not pattern_ip_num then 
    error(err)
    return nil
end

local mask_num = bit.lshift(bit.bnot(0), 32 - tonumber(pattern_split_result[2]))
local pattern_mask_res = bit.band(mask_num, pattern_ip_num)
```

在`lua`中使用`bit`来实现位运算
封装了`split_str_to_array`函数来分割字符串，在网络段匹配的时候也需要使用这个函数来将字符串表示的ip地址拆分位多段分别进行计算。

`ip4_str_to_num`将所给的用字符串表示的ip地址转换成数字表示的ip地址，方便后面进行位运算。

在掩码的计算需要根据'/'后面的数字将前面的若干位置1，后面置0，可以通过对1取反然后左移得到

将`mask_num`与`pattern_mask_res`进行与运算，模式串对应的掩码的数字表示，需要检查ip时，只需要将ip转换位数字，然后与`mask_num`进行与运算截取前面若干位，然后与`pattern_mask_res`比较是否相等即可：
```lua
local ip_num, err = ip4_str_to_num(ip, nil)
if not ip_num then
    error(err)
end
local ip_mask_res = bit.band(mask_num, ip_num)
return bit.bxor(ip_mask_res, pattern_mask_res) == 0
```

## 网络段匹配
网络段的匹配方式实际上是将ip地址按照书写的方式，根据'.'分段进行匹配，更符合人的直觉。在实现上，也是将ip进行分段然后逐段进行匹配：
```lua
local ip_part_arr = split_str_to_array(ip, ".", true)
if not ip_part_arr or #ip_part_arr ~= 4 then
    error(string.format("ip[%] illegal", ip))
end

for i, ip_part in ipairs(ip_part_arr) do
    if pattern_part_arr[i] ~= "*" then           
        local ip_part_num = tonumber(ip_part)
        if not ip_part_num then
            error(string.format("ip[%] illegal", ip))
        end
        local pattern_part_str = pattern_part_arr[i]           
        if pattern_part_str:find("/") then
            pattern_part_str = pattern_part_str:sub(1, pattern_part_str:find("/") - 1)
        end
        local pattern_part_num = tonumber(pattern_part_str)
        if not pattern_part_num then 
            local pattern_part_arr = split_str_to_array(pattern_part_str, "-")
            if not pattern_part_arr or #pattern_part_arr ~= 2 then
                error(string.format("pattern[%s] illegal", pattern))
            end
            local lnum = tonumber(pattern_part_arr[1])
            local rnum = tonumber(pattern_part_arr[2])
            if not lnum or not rnum then
                error(string.format("pattern[%s] illegal", pattern))
            end
            if ip_part_num < lnum or ip_part_num > rnum then 
                return false
            end
        else
            if pattern_part_num ~= ip_part_num then
                return false
            end
        end
    end
end
```
模式串中是数字的部分，直接与ip中对应部分的数字比较是否相等即可

模式串中是`*`的部分则直接方向

模式串中`a-b`的部分，需要分离出左右两个数字然后判断ip地址对应的部分的数字是否在这个两个数字之间

## 总结
总体而言程序是比较简单的，特别是不用考虑ipv6之后显得更加的简单。

功能上与业务无关，又感觉在这个小系统，特别是对于部署在局域网内且只需要部分ip返回的具有较高权限要求的服务的时候算是一个比较通用的功能，因此就将这块代码摘出来。

完整的代码放在[ip-filter](https://github.com/muzhy/ip-filter),将功能封装为一个类，使用方法可参考`test_ip-filter.lua`,其中也包含了几个测试用例

