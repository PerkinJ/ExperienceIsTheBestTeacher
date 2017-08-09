---
title: 由Facebook面试题：判断是否为凸边形而想到实际场景的应用
category: FrontEnd
tags: [算法,raphael.js,svg]
---
今天无意之中看到一道面试题,判断是否为凸边形，想到之前自己做过一个项目有一个类似的场景，暂且称为标注系统，场景是切割图片中的车牌，将图片导入canavas，并使用SVG来切割图片中的车牌。

那么，此时若有如下需求：

前提条件：鼠标每点击一次产生一个点，移动时生成直线，再点击一次生成一条线段。
结果：要生成一个多边形，且为凸多边形，将车牌切割出来。
注意：我们每次生成多边形之后，生成的是以canvas左上角为原点的坐标组合。

下面我们来分析一下解题思路，如何判断是凸多边形：
- 判断所有的点不在任意一条边的同侧，既点到直线的垂直矢量应一致。
- 角度法：判断每个顶点所对应的内角是否小于180度，如果小于180度，则是凸的，如果大于180度，则是凹多边形
- 叉乘法：即利用两条向量叉乘的结果，来判断。根据向量的叉积我们就可以判断这个多边形的内角是否均小于180度，相邻两条边的向量均保持顺时针或逆时针旋转才符合条件。
>A,B同为向量，**|A×B|=Ax*By-Ay*Bx，若这个值大于0，则说明B指向A逆时针旋转0到180的方向，若这个值小于0，则说明B指向A顺时针旋转0到180的方向，若等于0，则两向量共线。**

====需要注意的是==，我们需要额外判断一下(n-1,n)×(n,0)和(n,0)×(0,1)这两个叉积。并且叉积为0(即相邻的边共线，但不包括所有点共线)在本题中是可以被接受的。==

- ...还有多种数学方法


对于前端同学来说，JS没有那么强大的数学处理能力，而分析上述方法，叉乘法无非是最简单最直接的方法。

我们先来看看效果：

![image](https://github.com/PerkinJ/ExperienceIsTheBestTeacher/resource/2017/08/09.gif)

这是具体操作的情况，我们可以看到，画凸四边形跟三角形时，弹窗弹了true，表明为凸变形，画凹四边形，表明为凹变形。

查看demo请点击：[demo]()

下面是具体的代码实现：
```javascript
//判断是否为凸变形
function isConvex(arr){
    if(arr.length < 3){ alter('未知错误')}
    var clockwise = false //顺时针初始值
    var anticlockwise = false  //逆时针初始真
    for(var i = 2;i < arr.length;i++){
        if(cross_result(arr[i],arr[i-1],arr[i-2])){
            clockwise = true
        }else{
            anticlockwise = true
        }
    }
    if(clockwise && anticlockwise || (!clockwise && !anticlockwise)){
        return false
    }else{
        return true
    }
}
//假设传进来的参数是object,分别有x,y属性
function  cross_result (A,B,C){
    let AB = [B.x - A.x,B.y - A.y]
    let BC = [C.x - B.x,C.y - B.y]
    return multiplication_cross(AB,BC)
}

function multiplication_cross(arr1,arr2){
    return arr1[0]*arr2[1] - arr1[1]*arr2[0]>0?true:false
}
```

其实以上的代码还是比较好理解的，无非就是对数组进行遍历，每三个点进行叉乘判断，从而来判断方向是否都为顺时针或者都为逆时针，即证明这是凸变形。

*对于代码中的一些边界case，由于canvas中已经要求闭合图形，所以对于直线等情况，这里不做考虑*

当然如果您对此小画布感兴趣，可以来star一下:grin:[raphael-draw](https://github.com/PerkinJ/raphael-draw)项目