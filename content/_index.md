---
title: "Index"
description: "主页内容配置"
---
<style>
.my-typeit
{
    color:aliceBlue;
    font-size:25px;
}
</style>
<div class="my-typeit">
    <span>欢迎，我</span>
    {{< typeit 
    tag=span
    speed=120
    breakLines=false
    loop=true
    lifeLike=true
    >}}
    是一名大学牲
    来自福建师范大学协和学院
    热衷于学习.NET技术
    可能不常写文章
    喜欢睡觉
    偶尔玩玩碧蓝档案
    不喜欢欧老师
    {{< /typeit >}}
</div>