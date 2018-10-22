#  YYLabel
1. 在layerClass 中替换显示的layer 为 YYAsyncLayer,渲染时调用的是YYAsyncLayer的display方法.
2. layer 成为 YYAsyncLayerDelegate 代理,调用代理的newAsyncDisplayTask方法,返回一个YYAsyncLayerDisplayTask对象,该对象有 willDisplay,display,didDisplay三个block回调,处理绘制状态.
	1. 在 代理的 newAsyncDisplayTask 方法中
		

	
	
	
# YYTextContainer

定义了一个矩形或者不规则的范围来布局text,YYTextLayout可以有一个或者多个YYTextContainer来生成

* The YYTextContainer class defines a region in which text is laid out. YYTextLayout class uses one or more YYTextContainer objects to generate生成 layouts.
* A YYTextContainer defines rectangular regions (`size` and `insets`) or 
 nonrectangular shapes (`path`), and you can define exclusion paths inside the text container's bounding rectangle so that text flows around the exclusion 
 path as it is laid out. 
	
# YYTextLayout

* 所有属性都是只读的,存放的是布局的结果
* YYTextLayout class is a readonly class stores text layout result. All the property in this class is readonly, and should not be changed. The methods in this class is thread-safe (except some of the draw methods). *

## 构造方法
1. + (YYTextLayout *)layoutWithContainer:(YYTextContainer *)container text:(NSAttributedString *)text
2. + (YYTextLayout *)layoutWithContainer:(YYTextContainer *)container text:(NSAttributedString *)text range:(NSRange)range




