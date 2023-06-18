---
title: "复习 计算机图形学基础 第一章 绪论"
date: 2023-06-13T16:25:26+08:00
tags: ["计算机图形学"]
categories: ["期末复习"]
series: ["复习 计算机图形学基础"]
series_order: 1
---

## 第一章
### 图形与图像的区别
|内容|图形|图像|
|---|---|---|
|基本元素|点、线、面等几何元素，比如直线，圆，多边形。。。|像素|
|存储数据|各个矢量的参数|各个像素的灰度或颜色|
|处理方式|旋转，扭曲或拉伸等|对比度增强，边缘检测等|
|缩放结果|不会失真，适应不同分辨率|放大会失真|
|其他|图形不是客观存在的，是根据客观事物而主观形成|对客观事物的真实描述|
|实例|工程图纸|照片|
|应用领域|计算机辅助设计|图像处理|


### 计算机图形学概念
**研究通过计算机将信息、数据转换为图形，并在专门显示设备上显示的原理、方法和技术的学科。**

### OpenGL中基本的图元，常用的函数
#### 基本函数
```cpp
glutInit(int *argc,char **argv)// 使用外部参数初始化GLUT
glutInitDisplayMode(unsigned int mode) // 使用GLUT的枚举值初始化显示模式，指定缓冲区模式和颜色格式，使用按位或来同时设置
glutInitWindowPosition(int x,int y) //指定了窗口左上角的屏幕位置
glutInitWindowSize(int width,int size) //指定了窗口的大小（以像素为单位）
glutCreateWindow(char *string) //创建一个支持OpenGL渲染环境的窗口。在调用glutMainLoop()函数之前，这个窗口并没有显示。
glutDisplayFunc(void(*func)(void)) //OpenGL绘制回调函数，将绘制函数作为参数
glutMainLoop(void) //开始OpenGL绘制循环
```

#### 输入事件处理
```cpp
glutReshapeFunc(void(*func)(int w,int h)) //注册当前窗口大小变化时的回调函数
glutKeyboardFunc(void(*func)(unsigned char key,int x,int y)) //注册当前窗口的键盘回调函数
glutMouseFunc(void (*func)(int button, int state, int x, int y)); 
```
`glutMouseFunc`为注册当前窗口的鼠标回调函数，func为注册的鼠标回调函数,这个函数完成鼠标事件的处理，`button`为鼠标的按键,为以下定义的常量：  
| | |
|---|---|
|GLUT_LEFT_BUTTON|鼠标左键|
|GLUT_MIDDLE_BUTTON|鼠标中键|
|GLUT_RIGHT_BUTTON|鼠标右键|

`state`为鼠标按键的动作：
| | |
|---|---|
|GLUT_UP|鼠标释放|
|GLUT_DOWN|鼠标按下|

#### 绘制函数
**清除窗口**
下面两行代码把一个RGBA模式的窗口清除为黑色
```cpp
glClearColor(0.0, 0.0, 0.0, 0.0);
glClear(GL_COLOR_BUFFER_BIT);
```

**指定绘制颜色**
在绘制前使用下列函数指定绘制颜色
```cpp
glColor3f(float r, float g, float b); //三个参数的范围为0-1
glColor3ub(unsigned char r, unsigned char g, unsigned char b) // 三个参数的范围为0-255
```

**完成绘制**
```cpp
glFlush()
```
如果使用双缓冲区模式，则需要交换缓冲区
```cpp
glutSwapBuffers()
```

**绘制**  
- 使用`glBegin(unsigned int mode)`函数指定图元并开始绘制  
- 使用`glVertex..函数绘制顶点`，满足了顶点的数量则会自动根据顶点生成图元对应的图形  
- 使用`glEnd()`函数结束绘制

OpenGL几何图元
|模式|图元类型|
|---|---|
|GL_POINTS|将每个顶点绘制为点|
|GL_LINES|将每两个顶点绘制为一条线|
|GL_LINE_STRIP|将指定的顶点用于创建线条。第一个顶点之后的每个顶点指定的是线条延伸到的下一个点|
|GL_LINE_LOOP|与GL_LINE_STRIP类似，但是绘制完后起始顶点会和终结顶点连接|
|GL_TRIANGLES|将每三个顶点绘制为一个三角形|
|GL_TRIANGLE_STRIP|将指定的顶点用于创建三角条。指定前三个顶点之后，后继的每个顶点与它前面两个顶点一起用来构造下一个三角形。每个顶点三元组（在最初的组之后）会自动重新排列以确保三角形绕法的一致性|
|GL_TRIANGLE_FAN|将指定的顶点用于构造三角扇形。第一个顶点充当原点，第三个顶点之后的每个顶点与它的前一个顶点还有原点一起组合|
|GL_QUADS|将每四个顶点绘制为一个四边形|
|GL_QUADS_STRIP|将指定的顶点用于构造四条形边。在第一对顶点之后，每对顶点定义一个四边形。和GL_QUADS的顶点顺序不一样，每对顶点以指定顺序的逆序使用，以便保证绕法的一致|
|GL_POLYGON|将指定的顶点用于构造一个凸多边形。多边形的边缘决不能相交。最后一个顶点会自动连接到第一个顶点以确保多边形是封闭的|
![图元](./%E5%9B%BE%E7%89%871.png "图元示例")
