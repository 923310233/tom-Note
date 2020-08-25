## theos语法

- **`%hook`**
  指定需要hook的class,**必须以**`%end`结尾，如下：

```
 %hook SpringBoard
 -(void)_menuButtonDown:(id)down
 {
   NSLog(@"You've  pressed home button");
   %orig; //call the original _menuButtonDown
 }
 %end
```

这句话的意思是先勾住（Hook）SpringBoard类里的_menuButtonDown：函数，先将一句话写入 syslog,再执行函数的原始操作。



**`%org`**
该指定在`%hook`内部使用，执行被勾住（hook）的函数的原始代码



还可以利用`%orig`更改原始函数的参数，例如：

```
%hook SBLockScreenDateViewController 

- (void)setCustromSubtitleText :(id)arg1 withColor :(id) arg2
{
        %orig(@"我是你的某某 ",arg2);
}
@end
```



**`group`**
该指令用于将`%hook`分组，便于代码管理及按条件初始化分组，**必须以**`%end`结尾：一个`%group`可以包含多个`%hook`，**所有不属于某个自定义group的`%hook`会被隐式归类到`%group _ungrounped`中**，`%gruop`的用法如下：

```
%group iOS7Hook
%hook iOS7Class
-(id) iOS7Method
{

    id result = %orig;
    NSlog(@"This class & method only exist in ios 8.");
    return result;
}
@end
@end // iOS7Hook
```

**需要注意：**  `%group`必须配合`%init`使用才能生效。



- **`%init`**
  该指令用于初始化某个`%group`，必须在`%hook`或`%ctor`内调用，**如果带参数，则初始化指定的group,如果不带参数，则初始化_ungrouped**

**注意：**只有调用了`%init`,对应的`%group`才能起作用、

********

**`%ctor`**
tweak 的constructor ，完成初始化工作；如果不是显示定义，theos会自动生成一个一个`%ctor`并在其中调用`%init(_ungrouped)`。



```
%hook SpringBoard

-(void) reboot
{
    NSlog(@"你好");
    %orig;
}
%end


%ctor
{
    // need to call %init  explicitly!
}
```

**这里 `%hook`无法生效**，因为这里显示定义了`%ctor`，却没有显示的调用`%init`,因此`%group(_ungrouped)`不起作用



**注意：**`%ctor`不需要以`%end`结尾。



- **`%new`**
  在`%hook`内部使用，给一个现有class增加新函数，功能与class_addMethod相同。

```
%hook SpringBoard
%new
-(void) namespaceNewMethod
{
        NSlog(@"你好");
}
@end
```







**%c**
该指令作用等同于objec_getClass或NSClassFromString，及动态的获取一个类的定义在`%hook`或`%ctor`内使用。



**%c**的用法：

假设ios上app中定义了类CLASS_A,CLASS_B;

定义如下：

```
CLASS_A｛

-（void） fun_A(){
......

}

.....

｝



CLASS_B｛

-（void） fun_B(){
......

}

.....

｝
```

问题：在hook CLASS_A的函数 fun_a  时，如何调用CLASS_B的函数fun_B()?

解决方法：

1，把CLASS_B的相关头文件copy到 theos/include/目录下

2，在Tweak.xm文件头声明相关类

```
/**

**Tweak.xm

**/

@interface CLASS_A : NSObject

_(void) fun_B;

@end

%hook CLASS_A

-(void) fun_A{

//do some thing

[ %c(CLASS_B) fun_B ];

....

}
%end
```







## 其它

Objective-C 中的id：

因为objc中所有的类类型都继承自NSObject，id相当指向NSObject的指针，因而id能用于指向所有的类类型的变量

