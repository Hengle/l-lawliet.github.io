---
title: Unity3D插件开发教程（一）：自定义锚点组件
date: 2017-02-03 17:25:44
tags: 
	- unity3d
	- unity3d editor tutorial
---
教你如何扩展Unity3D编辑器，订制锚点组件，在Scene视图上绘制线框物体。<!-- more -->

### 前言 Preface

在制作游戏（应用）时，很多时候不能把所有对象堆放到场景里，因为这样，场景的加载和实例化会非常慢，会有明显的卡顿感觉。在这种情况下就需要有顺序的初始化场景的对象，例如地面会在一开始初始化，小草石头等小物件放后面，场景上粒子特效（小草上冒起的光点等）放到最后。

那小草和特效应该放置在哪个位置呢、应该旋转多少度、缩放多少呢，这个时候我们可能需要定制一个组件：锚点。

这里所说的锚点（Anchor）相对于网页制作的锚点只有记录坐标、旋转、缩放和层级（parent是谁）等Transform组件内容的作用。

既然用于记录Transform内容，那么使用空的GameObject或者使用Cube等组件就行了，为什么要定制一个锚点组件呢？
这里说说他们的优缺点。

**空GameObject：** 只有在选中对象时候，才会显示相应坐标轴、旋转轴、缩放轴，而且一次只能选择一个轴，非常的不方便。

**Cube等基础物体：** 相对于空GameObject，Cube比较直观的能观察到变换，可缺点也很明显，自带的MeshFilter、MeshRenderer、BoxCollider等组件会降低运行效率。

**自定义锚点组件：** 在使用以上两种方案都感到不方便的时候，我们就需要创建一种编辑状态拥有可视化图像可以变换的，并且在运行状态不会影响效率的锚点组件。利用编辑器的Handles类在Scene界面绘制线框。

![image](https://github.com/L-Lawliet/UnityEditorTutorial/blob/master/AnchorComponent/Picture2.png?raw=true)

> **Editor入门要点：**
Unity编译器扩展的开发有如下规定，如果是编辑器脚本，你需要在Editor文件夹下创建你的插件代码，项目中任何目录下创建的Editor文件夹都会被编译成编译器使用的代码。
ps:插件开发最好有一套自己的分类管理，方便管理和修改，例如很多人喜欢把所有插件放置在Plugin目录下。

### 准备 Prepare

知识要点：	
- **Editor**
- **Handles**
- **MenuItem**
- **Selection**

使用版本：
- **Unity3D 5.3.3**

目标：
- **使用Handles类在Scene视图绘制一个正方体**

整个插件的结构：

![image](https://github.com/L-Lawliet/UnityEditorTutorial/blob/master/AnchorComponent/Picture4.png?raw=true)

### 正文 Main

首先我们在AnchorComponent创建一个AnchorComponent.cs代码（注意命名空间可能会报错，可以修改一下命名空间）。

AnchorComponent.cs就是我们要实现的锚点组件了。它只有简单几个属性用于记录锚点的参数。所以在运行时并不会对效率产生多大的影响。

``` CSharp
public class AnchorComponent : MonoBehaviour
{
    /// <summary>
    /// 线框颜色
    /// </summary>
    public Color color = new Color(0.3f, 0.9f, 0.3f);

    /// <summary>
    /// 是否使用全局缩放
    /// </summary>
    public bool isUserLossyScale = false;

    /// <summary>
    /// 是否使用Handles标准尺寸
    /// </summary>
    public bool isUserHandleSize = false;
}
```
![image](https://github.com/L-Lawliet/UnityEditorTutorial/blob/master/AnchorComponent/Picture3.png?raw=true)

以上的属性待会在使用到的时候会一一给予介绍。

---

然后我们在AnchorComponent/Editor文件夹下创建AnchorComponentEditor.cs代码，此代码用于在Scene视图上绘制线框。


``` CSharp
[CustomEditor(typeof(AnchorComponent))]
public class AnchorComponentEditor : Editor
{
    void OnSceneGUI()
    {
        //....
    }
}
```

AnchorComponentEditor需要继承Editor类，扩展Scene、 Hierarchy、Inspector等视图的插件都需要继承Editor类，并创建特定的函数用于相应的扩展。

例如，本文需要在Scene视图上绘制，则要创建OnSceneGUI函数。并且在OnSceneGUI函数里书写绘制逻辑。

接着在类的声明前加入[CustomEditor(Type t)]，这是要告诉编辑器，我这个编辑类是给哪个组件使用的。

---

然后我们要在上面的OnSceneGUI函数里添加如下代码。
``` CSharp
void OnSceneGUI()
{
    AnchorComponent compontent = target as AnchorComponent;

    Handles.color = compontent.color;

    //绘制实心正方体
    //Handles.CubeCap(compontent.GetInstanceID(), compontent.transform.position, compontent.transform.rotation, 1.0f);

    //绘制线框正方体
    DrawWireframeBox(compontent.transform, compontent.isUserLossyScale, compontent.isUserHandleSize);
}
```

Editor类的target属性可以获取到你正在选择操作的组件。当然，要强制性转换成相应的组件类才能使用。

**Handles类：** 包含了大量在Scene视图绘制直线、弧线、锥体、圆等基础图形的函数。

**Handles.color：** 很好理解，就是设置接下来绘制的对象的颜色。

**Handles.CubeCap：** 绘制一个实心的正方体，这里会介绍到是因为可能有人比较喜欢看实心的正方体。

接下来是绘制线框正方体，由于逻辑比较多，所以另外写成一个函数**DrawWireframeBox**。

> PS：5.4版本后，Handles新增了绘制线框正方体DrawWireCube的方法，由于暂无接触，所以这里不多介绍，使用5.4版本的童鞋可以实践下

``` CSharp
public void DrawWireframeBox(Transform transform, bool isUserLossyScale = false, bool isUserHandleSize = false)
{
    Matrix4x4 matrix = Handles.matrix;

    Vector3 size;
    
    if (isUserHandleSize)
    {
        float handleSize = HandleUtility.GetHandleSize(transform.position);

        size = Vector3.one * handleSize;
    }
    else
    {
        size = transform.localScale;
    }

    if (isUserLossyScale)
    {
        size.Scale(transform.lossyScale);
    }

    //创建一个无缩放影响的矩阵
    Matrix4x4 tempMatrix = Matrix4x4.TRS(transform.position, transform.rotation, Vector3.one);

    //把矩阵赋予给Handles
    Handles.matrix = tempMatrix;

    Vector3 vector = size * 0.5f;

    Vector3[] points = new Vector3[8];

    //设置正方体八个顶点
    points[0] = new Vector3(vector.x, vector.y, vector.z);
    points[1] = new Vector3(vector.x, -vector.y, vector.z);
    points[2] = new Vector3(vector.x, -vector.y, -vector.z);
    points[3] = new Vector3(vector.x, vector.y, -vector.z);
    points[4] = new Vector3(-vector.x, vector.y, vector.z);
    points[5] = new Vector3(-vector.x, -vector.y, vector.z);
    points[6] = new Vector3(-vector.x, -vector.y, -vector.z);
    points[7] = new Vector3(-vector.x, vector.y, -vector.z);

    //绘制正方体12条线
    Handles.DrawLine(points[0], points[1]);
    Handles.DrawLine(points[1], points[2]);
    Handles.DrawLine(points[2], points[3]);
    Handles.DrawLine(points[3], points[0]);

    Handles.DrawLine(points[4], points[5]);
    Handles.DrawLine(points[5], points[6]);
    Handles.DrawLine(points[6], points[7]);
    Handles.DrawLine(points[7], points[4]);

    Handles.DrawLine(points[0], points[4]);
    Handles.DrawLine(points[1], points[5]);
    Handles.DrawLine(points[2], points[6]);
    Handles.DrawLine(points[3], points[7]);

    //用与显示顶点位置与索引
    /*for (int i = 0; i < 8; i++)
    {
        Handles.Label(points[i], i.ToString());
    }*/

    Handles.matrix = matrix;
}
```
由于这部分代码比较多，所以分开一块块来解释。
``` CSharp
//首先缓存了Handles的矩阵，因为后面需要重新赋值。作用不大，不过养成良好的习惯。
Matrix4x4 matrix = Handles.matrix;
```

``` CSharp
//然后判断是否使用Handles的固定尺寸。
//如果是，就使用HandleUtility.GetHandleSize方法获取，这是根据坐标到摄像机的距离返回的尺寸。
if (isUserHandleSize)
{
    float handleSize = HandleUtility.GetHandleSize(transform.position);

    size = Vector3.one * handleSize;
}
else
{
    size = transform.localScale;
}
```

``` CSharp
//是否使用全局缩放，此方法是为了应对父对象们的缩放导致的缩放，建议为true，因为这才是加载进去的物体的正确缩放。
if (isUserLossyScale)
{
    size.Scale(transform.lossyScale);
}
```

``` CSharp
//这里是过去锚点的全局坐标和旋转，然后生成矩阵给Handles使用，注意，这里是使用全局的，而且缩放使用一就是了。
Matrix4x4 tempMatrix = Matrix4x4.TRS(transform.position, transform.rotation, Vector3.one);

//把矩阵赋予给Handles
Handles.matrix = tempMatrix;
```
接下来就是设置8个点了，代码就不重复贴了，如果对8个点的位置没有概念，可以使用后面注释了的Handles.Label把索引显示出来，可以直观看出效果如何。

![image](https://github.com/L-Lawliet/UnityEditorTutorial/blob/master/AnchorComponent/Picture1.png?raw=true)

接下来是绘制12根线，这里由于比较直观让大家知道对应的线的位置，所以使用DarwLine来一根根画出来。

接下来的就是超纲内容了，跟锚点没关系的了。
就是，每次生成锚点的时候都要添加一个GameObject，然后挂一个锚点组件好像比较麻烦哦。
那么就可以把锚点组件加入到菜单里。

首先在AnchorComponent/Editor文件夹下创建AnchorComponentMenu.cs代码，用于增加添加锚点组件的菜单选项。


``` CSharp
[MenuItem("GameObject/3D Object/AnchorComponent", false, 10000)]
public static void CreateAnchorComponent()
{
    GameObject anchorObject = new GameObject("AnchorObject");

    anchorObject.AddComponent<AnchorComponent>();

    if (Selection.activeObject != null) 
    {
        GameObject parent = Selection.activeObject as GameObject;
        anchorObject.transform.SetParent(parent.transform);
    }
    
}
```

MenuItem是Unity添加菜单的类，使用方法之一（后面会教其他方法）就是在static函数上面增加MenuItem特性(Attribute)。

- 第一个参数：菜单的地址，我们使用GameObject/3D Object，因为Hierarchy视图中的菜单一级目录是Unity默认绘制的，如果不想整个菜单重写，那么只能使用一个二级目录。
- 第二个参数：未试验过。
- 第三个参数：菜单的优先级，值越大，排的越后。使用10000可以保证在默认组件最后。

然后在函数里创建一个GameObject，然后为其添加锚点组件。
接着使用Selection.activeObject属性获取当前选择的对象，然后把新建的锚点添加到选择对象的子级。

### 源码 Code
> [源代码](https://github.com/L-Lawliet/UnityEditorTutorial/tree/master/AnchorComponent)

