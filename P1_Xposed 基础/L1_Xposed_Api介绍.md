当你可以在 Android Studio 里面打开反编译好的微信项目的时候，我们便可以开始开发了。

你可以像创建普通项目一样创建一个 Android Application 项目。

然后对这个项目作如下修改：





做了这些修改后，你已经在项目中完成了 Xposed 的配置，也接入了相应的 Api，接下来我们可以进入到开发过程中了。

简单的介绍一下 Xposed Api，他可以在 Android 系统里面的**任何应用**运行**任何方法**时进行进行拦截并修改，例如对下面这段代码：

```kotlin
class TextColorDemo {

    private fun setFailText(view: TextView, msg: String) {
        view.text = msg
        view.setTextColor(0xFFFF0000)
    }

    private fun setSuccessText(view: TextView, msg: String) {
        view.text = msg
        view.setTextColor(0xFF00FF00)
    }

    private fun invoke(){
        setFailText(textView2,"失败操作")
        setSuccessText(textView3,"成功操作")
    }

}
```

这是一段 Kotlin 代码，逻辑十分简单，前两个方法分别给传入的 TextView 制定了不同的颜色，分别是错误、成功两种颜色。最后一个方法则是使用两个 TextView 来显示文字。

所以效果应该是这个样子的：                                

![](https://github.com/zhudongya123/WechatChatRoomHelper_Tutorial/blob/master/resource/pict_1.png)

现在我使用 Xposed Api 拦截修改一下，编写下列代码：

```kotlin

        XposedHelpers.findAndHookMethod(TextColorDemo::class.java, "setFailText", TextView::class.java, String:class.java, object : XC_MethodHook() {

            override fun afterHookedMethod(param: MethodHookParam) {
            	val textView = param.args[0] as TextView
            	val msg = param.args[1] as String

                val currentString = textView.getText().toString()
            	textView.setText(currentString + "·失败文本和颜色被修改！！")
            	textView.setTextColor(0xFFFF8080)
            	param.setResult(null)

            }
        })

        XposedHelpers.findAndHookMethod(TextColorDemo::class.java, "setSuccessText", TextView::class.java, String:class.java, object : XC_MethodHook() {

            override fun beforeHookedMethod(param: MethodHookParam) {
                val textView = param.args[0] as TextView
                val msg = param.args[1] as String

                val currentString = textView.getText().toString()
                textView.setText(currentString + "·成功文本和颜色被修改！！")
                textView.setTextColor(0xFF80FF80)
                param.setResult(null)

            }
        })

```

经过上述修改，现在效果变成了这个样子：

![](https://github.com/zhudongya123/WechatChatRoomHelper_Tutorial/blob/master/resource/pict_2.png)

findAndHookMethod 是 Xposed Api 里面最常用的一个方法，字面的意思是寻找并钩住方法，实际上在这里实际上是拦截修改之意。参数表分别是类（Class），方法名（String），参数类型（Class），最后传入一个叫  XC_MethodHook 的匿名内部类的实例，这样正确的输入参数，即可在 Android Application 运行时执行这个方法的时候提供回调，这两个回调方法分别在原函数执行前和执行后调用，同时提供运行时环境来帮助拦截修改逻辑。

上面这段代码，在 afterHookedMethod 方法中，通过 param.args 参数取到了 setWarmText 每次执行时的参数表，（param 的成员变量 args 通过一个数组保存了运行时的参数表实例）

然后获取到了 TextView 的实例并指定了自定义的文本和颜色，注意，文本最后为：**警告操作警告文本被修改！！**  因为是在 afterHookedMethod 方法后调用，原方法已经执行，TextView 已经存在 text，故 currentString 可以获得字符串 "警告操作"，加上后面添加的字符串产生了最后的效果，同样的情况在第二个方法 setSuccessText 中的 beforeHookedMethod 中出现，则发现并没有附加文本的出现，因为该方法回调是在原方法执行前调用。

最后调用了 param.setResult(null) ，setResult 在不同方法下效果不同，当在 afterHookedMethod 方法中调用时，修改了**原函数**的返回结果，（此处因为**setWarmText**方法只是修改颜色文本，并没有方法的返回值，所以并不影响运行结果）但是如果在 beforeHookedMethod 函数中调用 setResult ，不仅修改了原函数的返回结果，同时也阻止了原函数的运行，（参见）。

所以对于这段 Api 的总结就是：

- beforeHookedMethod，afterHookedMethod 方法都可以获得运行时的参数表实例，可以进行拦截修改等逻辑。
- param 的成员变量 args 数组可以获得参数表实例。
- param 的成员变量 thisObject 为被拦截方法的类实例（例如拦截 Activity 的 onCreate 方法，则 thisObject 为 activity 实例）。
- param 的 setResult 方法用来修改被拦截函数的返回值，而且在 beforeHookedMethod 方法中调用，可以阻止原方法的调用。

