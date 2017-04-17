# DiscardFilePlugin

An android gradle plugin for discard class or method in compile time.

用于在编译构建时期忽略清空类和方法的一个Android Gradle插件。

## 1.1 使用场景

在`debug`模式下加入`DebugPanelActivity`（调试面板工具页面，提供比如“切换服务器”等操作）。我们需要在正式上线的release版本中清空相关类和方法，或者修改`boolean isProductionEnvironment()`方法，让它永远返回`true`.

## 1.2 `@Discard`注解

### 1.2.1 使用Target

- `ElementType.METHOD`: 表示清空方法中的代码，编译过程中该方法中代码被清空。

- `ElementType.TYPE`: 表示清空类，其实是清空类中的所有方法。

### 1.2.2 参数：

#### 1.2.2.1 `applyParam`

参数key，默认是`isRelease`，表示是否是生产环境。

#### 1.2.2.2 `applyParamValue`

参数value，默认是`true`，与`applyParam`一起使用，表示如果`${applyParam}=${applyParamValue}，默认为isRelease=true`时，Discard才会生效，才会真正在编译时去对方法或者类进行清空。因此可以在每个方法或者类中去进行不同的配置，在不同状态下通过如下方式对不同方法进行Discard：

```java
@Discard(applyParam = "test1", applyParamValue = "true")
public void testMethod_1() {
    System.out.println("testMethod_1...");
}

@Discard(applyParam = "test2", applyParamValue = "true")
public void testMethod_2() {
    System.out.println("testMethod_2...");
}
```

使用`gradle assembleDebug -Ptest1=true -Ptest2=false`来构建时，`testMethod_1()`方法会被discard，而`testMethod_2()`不会被discard。构建完毕反编译class结果如下：

```java
@Discard(applyParam = "test1", applyParamValue = "true")
public void testMethod_1() {
}

@Discard(applyParam = "test2", applyParamValue = "true")
public void testMethod_2() {
    System.out.println("testMethod_2...");
}
```

#### 1.2.2.3 `srcCode`

替换方法的方法体，如果不设置，默认discard方法实现：

- 返回类型为`void`: discard后方法体为`{}`
- 返回类型为原始数据类型：discard后方法返回默认值，比如`{ return 0; }`
- 返回类型为类对象时： discard后方法返回为`{ return null; }`

可以如下填写具体的方法体代码块：

```java
@Discard(srcCode = "{super.onCreate($1); System.out.println(\"this: \" + $0);}")
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        usernameEt = (EditText) findViewById(R.id.activity_main_username_et);
        passwordEt = (EditText) findViewById(R.id.activity_main_password_et);
        setTestAccount();
    }
```

discard之后的class反编译代码如下：

```java
@Discard(
        srcCode = "{super.onCreate($1); System.out.println(\"this: \" + $0);}"
    )
    protected void onCreate(Bundle var1) {
        super.onCreate(var1);
        System.out.println("this: " + this);
    }
```

方法的`$0`表示当前对象`this`，方法参数依次为`$1, $2, $3...`，[详细文档参考这里](http://jboss-javassist.github.io/javassist/tutorial/tutorial2.html#alter)

#### 1.2.2.4 `makeClassNames`

可以在这里指定具体的类名，在discard时对未在classPath的类进行make。**不常用，可以省略。**

#### 1.2.2.5 `enable`

表示改方法或者类的discard是否开启，默认为`true`，比较典型的场景为，在类上面增加`@Discard`对该类所有方法进行discard，但是需要某个方法不discard，这时可以使用`@Discard(enable = false)`来对方法进行排除在`discard`范围外。


## 使用方式

> 源码依赖，目前不支持远程依赖，稍后更新远程依赖的支持

### 1. 修改`build.gradle`

```groovy
// 使用插件
apply plugin: 'com.wangjie.plg.discardfile'

// 配置需要修改的类所属在那些包下
discard {
    includePackagePath 'com.wangjie.plg.discardfile.sample.ui', 'com.wangjie.plg.discardfile.sample.include'
    excludePackagePath 'com.wangjie.plg.discardfile.sample.exclude'/*, 'com.wangjie.plg.discardfile.sample.ui.MainActivity'*/
}
```

### 2. 设置`@Discard`注解

在需要清空的类上添加`@Discard`注解，`applyParamValue = "true"`表示只有在`Release`版本下，才会执行Discard。

```java
@Discard(applyParamValue = "false")
public class IncludeClassC {
    /**
     * 因为IncludeClassC类增加了`@Discard`注解，所以该方法也会被discard。
     */
    public void onIncludeMethodC() {
        System.out.println("onIncludeMethodC...");
    }

    /**
     * 替换该方法的实现为：{System.out.println("onIncludeMethodC_2... injected!");}
     */
    @Discard(srcCode = "{System.out.println(\"onIncludeMethodC_2... injected!\");}")
    public void onIncludeMethodC_2() {
        System.out.println("onIncludeMethodC_2...");
    }

    /**
     * 替换该方法永远返回true
     */
    @Discard(srcCode = "{return true;}")
    public boolean onIncludeMethodC_3() {
        System.out.println("onIncludeMethodC_3...");
        return false;
    }

    /**
     * 因为IncludeClassC类增加了`@Discard`注解，所以该方法也会被discard。
     */
    public int onIncludeMethodC_4() {
        System.out.println("onIncludeMethodC_4...");
        return 100;
    }

    /**
     * 由于使用了`@Discard`注解进行显式地声明禁用了本地的discard，所以该方法不会被discard
     */
    @Discard(enable = false)
    public String onIncludeMethodC_5() {
        System.out.println("onIncludeMethodC_5...");
        return "hello world";
    }

    /**
     * 替换该方法永远返回"hello world"字符串
     */
    @Discard(srcCode = "{return \"hello world injected!\";}")
    public String onIncludeMethodC_6() {
        System.out.println("onIncludeMethodC_6...");
        return "hello world";
    }
}
```

### 3. build运行

**在`Release`环境下**编译完成之后，该类的`class`文件将会根据配置的`@Discard`注解被自动修改成如下：

```
build/intermediates/classes/../release/.../IncludeClassC.class
```

```java
@Discard(
    applyParamValue = "false"
)
public class IncludeClassC {
    public IncludeClassC() {
    }

    public void onIncludeMethodC() {
    }

    @Discard(
        srcCode = "{System.out.println(\"onIncludeMethodC_2... injected!\");}"
    )
    public void onIncludeMethodC_2() {
        System.out.println("onIncludeMethodC_2... injected!");
    }

    @Discard(
        srcCode = "{return true;}"
    )
    public boolean onIncludeMethodC_3() {
        return true;
    }

    public int onIncludeMethodC_4() {
        return 0;
    }

    @Discard(
        enable = false
    )
    public String onIncludeMethodC_5() {
        System.out.println("onIncludeMethodC_5...");
        return "hello world";
    }

    @Discard(
        srcCode = "{return \"hello world injected!\";}"
    )
    public String onIncludeMethodC_6() {
        return "hello world injected!";
    }
}
```

License
=======


```
Copyright 2017 Wang Jie

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing blacklist and
limitations under the License.
```


