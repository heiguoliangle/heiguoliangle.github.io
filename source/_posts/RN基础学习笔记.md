---
title: RN基础学习
layout: post
date: 2018-01-23 13:44:34
comments: true
tags: [RN]

---

# RN学习笔记
## 基本语法
### 1. 关于`{}` 和 `()` 的用法
  > `{}`  主要用途
  
* 对于库中非默认组件例如
  
```
import React, { Component } from 'react';
```
* 包装对象

```
container: {
    flex: 1,
    justifyContent: 'center',// 主轴的对齐方式
    alignItems: 'center',// 侧轴的对齐方式
    backgroundColor: '#F5FCFF',
    marginTop:20,
    flexDirection:'row-reverse',// xy 轴
    flexWrap:'wrap'// 是否换行
}
```
* 对于引用变量

```
var a = 1,
<Text>{a}</Text>

```

* 表达式都需要

```
 <View style={styles.container}>
```

* 数字需要用 `{}`包装

```
{1}
```

> `()` 主要用途
	
* 包装组件标签时候
	
```
(
   <View style={styles.container}>
     </View>
 );
```

> `,` 的使用
当在分割对象时候使用功能`,`

### 2. CSS 布局 
*决定一个子控件的布局*
> 基本的CSS布局

```
var styles = StyleSheet.create({
  container: {
    flex: 1,// flex = 1 全屏占据
    justifyContent: 'center',// 主轴的对齐方式
    alignItems: 'center',// 侧轴的对齐方式
    backgroundColor: '#F5FCFF',
      marginTop:20,
      flexDirection:'row-reverse',// xy 轴
      flexWrap:'wrap'// 是否换行

  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  }
}
```
> 基本语法

`margin`,如果是第一个子控件,参照父控件;不是的话参照上一个控件;
`border`,设置边框,包括颜色,宽度,圆角半径*Radius*;
`padding`,内间距,针对子控件的位置处理,top, left,right,bottom;

`position`,让当前控件参照父控件 **absolute**,之后可添加Bottom,Right等参照父控件布局;**relative**,参照自己,相当于**bounds**;

### 3. flex布局
*针对多个控件的布局*
> 主轴/侧轴 : 垂直关系,默认主轴为column方向

`flexDirection` 主轴方向,row 从左向右,column 自上而下,row-reverse | column-reverse 反转;
`flexWrap` 布局是否可以换行, wrap 换行,nowrap,不换行(默认的);
`justifyContent` center,flex-end,flex-start,space-between(每个间距都是一样的),space-around(两边间距是中间间距的一半);
`alignItems` 决定侧轴的位置,flex-end,flex-start,center,stretch(拉升,在侧轴方向上全部拉伸,可填满 );
`alignSelf` 单独设置一个控件的布局;
`flex` 决定主轴中平均分成几部分;

* 主轴默认的flex = 1,侧轴默认是flex= stretch,子控件的flex的和就是主轴能分成几等分

`Props` 自定义属性,每个组件都有props属性,自动生成的属性都会放入这个里面;*不能再自己的组件中修改,只能在父控件总修改,修改时候会刷新界面*,需要在父控件中指定,内部不能修改;(会刷新界面)

```
使用时候:
1. <HGL name = "hgl" age = {18}/>
2. 	<View style = {styles.HGL}>
                <Text>
		 名字 + {this.props.name} + 年龄 + {this.props.age}
                </Text>
      </View>
```
`state` 在内部修改属性,修改时候需要设置 'this.setState`方法

```
  updataMoney(){

        var name = this.state.name;
        name += 1000
        this.setState(
            {
                name : name
            }
        );
    }
    constructor(props){
        super(props);
        this.state = {
            name : 10
        }
        setInterval(this.updataMoney.bind(this),1000);
    }
```

### 4. 传值
#### 1. 正向传值
 - setState
 
```
<Son ref = "son" name = "sub" getMoney = {this.getMoney.bind(this)}/>
```
使用ref 将该子控件作为属性记录;

``` 
sendMoney() {
        var son = this.refs.son;
        var name = son.state.name;
        son.setState(
            {
                name : ++name
            }
        )
    }
```
使用this.refs.son 获取到ref引用的子控件,之后重写子控件的setState.

#### 2. 逆向传值
在父控件中定义方法,通过props中定义一个属性传给子控件

```
<Son ref = "son" name = "sub" sonGetMoney = {this.getMoney.bind(this)}/>
```
sonGetMoeny 为props中的一个属性,getMoney 为父控件的方法,提供给子控件进行调用.

```
 answer() {
        this.props.sonGetMoney(this.state.name)
    }
```
子控件在合适的方法中调研this.props.sonGetMoney,就是调用父控件的方法了.
#### 3. 没有关联控件间传值
使用通知进行传值,需要导入框架`DeviceEventEmitter`;
发通知:

```
DeviceEventEmitter.emit("gege",100);
```

接受通知:(需要在constructor方法中进行添加)

```
DeviceEventEmitter.addListener("gege",this.getMoney.bind(this));
```








 
 





	
	
	
	
	
	


		


