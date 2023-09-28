---
title: Gradle DSL 解密
date: 2023-04-13 21:14:51
layout:     post
subtitle:  ""
author:     "李文坤"
catalog:  true
tags:
    - Gradle
---

# 前言
gradle 的强大之处不在于它本身提供了多少功能，而是它的扩展性强，几乎任务都可以通过插件的形式集成到构建流程中，而且 gradle 借助 groovy 强大的 dsl 定制能力，能够让我们很方便的完成构建脚本的编写。也正是由于 dsl 的可定制化能力强，gradle 脚本的语法经常让我们琢磨不透，无法从传统编程语言的角度去理解它。这篇文章将会从源码探究 gralde 对脚本 dsl 的定制，旨在帮助大家消除心中的各种疑惑，更好地理解和掌握 gradle 脚本。

Gradle 会利用 Groovy 编译器将 build.gradle 脚本文件编译成一个叫做 _BuildScript_ 的类，它继承自 ProjectScript，而 ProjectScript 又继承自 Sript，Sript 类是 Groovy 对脚本的建模。Script 有一个方法叫做 run()，脚本中的所有内容都会运行在这个方法里面。Sript 包含两类对象，一是 MetaClass，二是 Bindings。MetaClass 可以认为是对 Script 这个类本身的建模，当我们在脚本中进行各种方法调用和属性访问时，会先通过 MetaClass 来完成，如果 MetaClass 完成不了，那就通过 Bindings 来完成。既然 MetaClass 是对 Script 对象本身的建模，那么通过 MetaClass 中进行方法调用和属性访问就相当于通过反射对 Script 类进行方法调用和属性访问。我们看看 Script 都提供了哪些方法可供使用：

主要是 println 和 evaluate，这也就是为什么我们可以直接不通过任何对象，直接调用 println 和 evaluate 的原因，其实这两个对象最终都调到了 Script 类本身的这两个方法上了。

那么 Bindings 是什么呢？可以认为是让开发者对脚本进行定制化的接口。毕竟 Script 类自己提供的全局函数和变量很有限，如果我们希望脚本能够提供更多的全局函数和变量，就可以通过注入自定义的 Bindings 来实现。

但是 Gradle 貌似并没有使用 Groovy 提供的这套机制来对脚本进行定制化开发。而是使用了另辟蹊径，利用 Groovy 可手动指定脚本的基类来实现脚本能力的扩展。Groovy 默认将脚本编译成 Script 的直接子类，但也提供方法，让开发者可以指定一个 Script 的子类，让其作为编译后的脚本类的基类。如前文所述，Gradle 指定的这个子类就是 ProjectScript 类，而脚本本身叫做 _BuildScript_。

所以我们知道，Gradle 所有的秘密应该都可以从 ProjectScript 中找到。我们看看 ProjectScript 类的定义：
```java
public abstract class ProjectScript extends PluginsAwareScript {

    ......

    @Override
    public void apply(Closure closure) {
        getScriptTarget().apply(closure);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void apply(Map options) {
        getScriptTarget().apply(options);
    }

    @Override
    public void buildscript(Closure configureClosure) {
        getScriptTarget().buildscript(configureClosure);
    }

    @Override
    public LoggingManager getLogging() {
        return getScriptTarget().getLogging();
    }

    @Override
    public Logger getLogger() {
        return getScriptTarget().getLogger();
    }


    @Override
    public ProjectInternal getScriptTarget() {
        return (ProjectInternal) super.getScriptTarget();
    }

}
```
ProjectScript 中定义了 apply() getLogger() buildScript() 等常见方法，这也就是我们为什么可以在脚本中使用 apply{} logger.log() buildScript{} 的原因。

ProjectScript 其实也很简单，它并不是直接继承自 Groovy 的 Script 类的，它的继承链如下：

ProjectScript -> PluginsAwareScript -> DefaultScript -> BasicScript -> org.gradle.groovy.scripts.Script -> groovy.lang.Script

PluginsAwareScript 和 org.gradle.groovy.scripts.Script 很简单，没什么可讲的。重点是 DefaultScript 和 BasicScript 这两个类，前者用来扩充 Gradle 的领域相关能力，而后者用来支持这种扩充能力本身。我们先从 BasicScript 类看：

```java
public abstract class BasicScript extends org.gradle.groovy.scripts.Script implements org.gradle.api.Script, DynamicObjectAware, GradleScript {
    ......

    private Object target;
    private ScriptDynamicObject dynamicObject = new ScriptDynamicObject(this);

    @Override
    public void init(Object target, ServiceRegistry services) {
        ......
        setScriptTarget(target);
    }

    public Object getScriptTarget() {
        return target;
    }

    private void setScriptTarget(Object target) {
        this.target = target;
        this.dynamicObject.setTarget(target);
    }

    public PrintStream getOut() {
        return System.out;
    }

    @Override
    public Object getProperty(String property) {
        return dynamicObject.getProperty(property);
    }

    @Override
    public void setProperty(String property, Object newValue) {
        dynamicObject.setProperty(property, newValue);
    }

    public Map<String, ?> getProperties() {
        return dynamicObject.getProperties();
    }

    public boolean hasProperty(String property) {
        return dynamicObject.hasProperty(property);
    }

    @Override
    public Object invokeMethod(String name, Object args) {
        return dynamicObject.invokeMethod(name, (Object[]) args);
    }

    @Override
    public DynamicObject getAsDynamicObject() {
        return dynamicObject;
    }
}
```

脚本中的所有的方法调用都将会委托给 dynamicObject，它是 ScriptDynamicObject 对象。Gradle 经常出现 DynamicObject，它到底是个什么角色呢？

Gradle 脚本中的所有对象都将继承自 GroovyObject， Script 对象也不例外。GroovyObject 这个类是 Groovy 语言动态性的基础。Groovy 会把脚本内的所有方法调用和属性访问都编译成对 GroovyObject 的 getProperty setProperty invokeMethod 方法的调用。这就相当于 Groovy 对所有方法调用和属性访问都做了拦截，我们知道拦截的好处之一就是可以做访问控制，而这种访问控制让 Groovy 的动态性成为可能。举个例子，动态语言的特点之一是可运行时修改类的结构，假如我定义了一个 Groovy 对象 A 这个类是个空类，现有以下代码：

```java
class A {}
A a = new A()
a.b = new Object()
```
如果这段代码发生在 Java 中，肯定会报错，找不到属性 b。但是在 Groovy 中就不一样了。上面这段代码在逻辑上会被编译成这个样子:

```java
class A implements GroovyObject {
}
A a = new A()
a.setProperty("b", new Object())
```

当然，最后会通过反射尝试访问 A 的 b 属性，但是很显然，这会失败。但你可以这么做：

```java
class A {

    Map<String, Object> dynamicProperties = new HashMap<>()

    @Override
    void setProperty(String propertyName, Object newValue) {
        try {
            super.setProperty(propertyName, newValue)
        } catch (ignored) {
            dynamicProperties.put(propertyName, newValue)
        }
    }

    @Override
    Object getProperty(String propertyName) {
        try {
            return super.getProperty(propertyName)
        } catch (ignored) {
            return dynamicProperties.get(propertyName)
        }
    }
}
A a = new A()
a.b = new Object()
```

通过重写 setProperty getProperty 方法，让 Groovy 具备了动态添加属性的能力。有了对方法调用和属性访问的完全掌控，Groovy 就可以为所欲为，偷梁换柱，移花接木，釜底抽薪，暗度陈仓......

Groovy 的动态性也是 Gradle 实现自己 dsl 的基础。Gradle 将 Groovy 对象用来支持的动态性的一组方法桥接到了 DynamicObject 上。我们知道，桥接模式的优点是，让类的某些行为可以朝着独立于类本身的方向泛化，这样脚本类的子类可以专注于脚本本身的能力的扩充，比如给脚本提供各种和 Gradle 本身业务无关的全局函数和属性，这些能力比较固定，适合通过继承的形式进行扩充。而 dsl 的建设对动态性比较高，依赖于运行时的状态，所以 Gradle 利用桥接模式把脚本的动态能力桥接到 DynamicObject 上，为实现 Gradle 的 dsl 打下基础。实际上 ScriptDynamicObject 担任了这样的角色：

```java
private static final class ScriptDynamicObject extends AbstractDynamicObject {

        private final Binding binding;
        private final DynamicObject scriptObject;
        private DynamicObject dynamicTarget;

        ScriptDynamicObject(BasicScript script) {
            this.binding = script.getBinding();
            scriptObject = new BeanDynamicObject(script).withNotImplementsMissing();
            dynamicTarget = scriptObject;
        }

        public void setTarget(Object target) {
            dynamicTarget = DynamicObjectUtil.asDynamicObject(target);
        }

        @Override
        public Map<String, ?> getProperties() {
            return dynamicTarget.getProperties();
        }

        @Override
        public boolean hasMethod(String name, Object... arguments) {
            return scriptObject.hasMethod(name, arguments) || dynamicTarget.hasMethod(name, arguments);
        }

        @Override
        public boolean hasProperty(String name) {
            return binding.hasVariable(name) || scriptObject.hasProperty(name) || dynamicTarget.hasProperty(name);
        }

        @Override
        public DynamicInvokeResult tryInvokeMethod(String name, Object... arguments) {
            DynamicInvokeResult result = scriptObject.tryInvokeMethod(name, arguments);
            if (result.isFound()) {
                return result;
            }
            return dynamicTarget.tryInvokeMethod(name, arguments);
        }

        @Override
        public DynamicInvokeResult tryGetProperty(String property) {
            if (binding.hasVariable(property)) {
                return DynamicInvokeResult.found(binding.getVariable(property));
            }
            DynamicInvokeResult result = scriptObject.tryGetProperty(property);
            if (result.isFound()) {
                return result;
            }
            return dynamicTarget.tryGetProperty(property);
        }

        @Override
        public DynamicInvokeResult trySetProperty(String property, Object newValue) {
            return dynamicTarget.trySetProperty(property, newValue);
        }
        ......
    }
```
ScriptDynamicObject 中有三个重要的对象，bindings，scriptObject，dynamicTarget。bindings 前面已经讲过，scriptObject 是另一个 DynamicObject，这个类由 DefaultScript 包装而来，叫做 BeanDynamicObject，这个类可以认为是对 DefaultScript 的 MetaClass 对象的封装，对这个对象的方法和属性的访问其实最终都会通过脚本类的 MetaClass 来完成的。而 dynamicTarget 默认情况下就是 scriptObject，可以通过 setTarget 方法覆盖默认值，这个对象也同样是个 DynamicObject。可见，ScriptDynamicObject 其实是个包工头而已，它自己几乎没干啥，只是将所有的方法调用和属性访问都转包给 bindings，scriptObject，dynamicTarget 这三个对象了，前两个对象只是继承 Groovy 的动态性方案，毕竟 Gradle 是在 Groovy 平台玩，你可以扩展自己的玩法，但得保留别人的玩法，这样才能算得上是 Groovy 的超集。这三个对象我们主要关注 scriptObject 和 dynamicTarget。

因为 Gradle 的能力主要集中在 DefaultScipt 中，所以可以认为 scriptObject 代表着 DefaultScipt，DefaultScipt 这个类虽然很大，但职责简单，它仅仅是给脚本提供了很多工具方法，方便我们编写脚本。而后者 dynamicTarget 是通过 BasicScript#init() 注入进来的，这个对象到底是何方神圣呢？我们可以在 init 方法里进行断点，然后以调试模式运行 gradle 脚本中任何一个任务，就能通过调用栈追溯到这个对象最终来自于 DefaultProject#getAsDynamicObject()。我们知道，一个 build.gradle 脚本对应着一个 project，我们在脚本中的所有配置都是对这个 project 进行的，而DefaultProject 就是对 project 的建模。

现在，我们可以把焦点放在到 DefaultProject 上了。因为 DefaultScipt 提供的是领域无关的能力，所以我们可以断定，Gralde 脚本中的各种领域相关能力，都是 DefaultProject 实现的。我们从 getAsDynamicObject 方法为入口，对其一探究竟：

```java
public abstract class DefaultProject extends AbstractPluginAware implements ProjectInternal, DynamicObjectAware {
    ......
    private final ExtensibleDynamicObject extensibleDynamicObject;
    ......
    public DefaultProject(...) {
        ......

        services = serviceRegistryFactory.createFor(this);
        taskContainer = services.get(TaskContainerInternal.class);

        extensibleDynamicObject = new ExtensibleDynamicObject(this, Project.class, services.get(InstantiatorFactory.class).decorateLenient(services));
        if (parent != null) {
            extensibleDynamicObject.setParent(parent.getInheritedScope());
        }
        extensibleDynamicObject.addObject(taskContainer.getTasksAsDynamicObject(), ExtensibleDynamicObject.Location.AfterConvention);
        ......
    }
    @Override
    public DynamicObject getAsDynamicObject() {
        return extensibleDynamicObject;
    }
    ......
}
```

略去无关代码后，我们一眼就能看出 ExtensibleDynamicObject 是主角。我们看看 ExtensibleDynamicObject 中的逻辑：

```java
public class ExtensibleDynamicObject extends MixInClosurePropertiesAsMethodsDynamicObject implements org.gradle.api.internal.HasConvention {
    ......
    public ExtensibleDynamicObject(Object delegate, Class<?> publicType, InstanceGenerator instanceGenerator) {
        this(delegate, createDynamicObject(delegate, publicType), new DefaultConvention(instanceGenerator));
    }

    public ExtensibleDynamicObject(Object delegate, AbstractDynamicObject dynamicDelegate, InstanceGenerator instanceGenerator) {
        this(delegate, dynamicDelegate, new DefaultConvention(instanceGenerator));
    }

    public ExtensibleDynamicObject(Object delegate, AbstractDynamicObject dynamicDelegate, Convention convention) {
        this.dynamicDelegate = dynamicDelegate;
        this.convention = convention;
        this.extraPropertiesDynamicObject = new ExtraPropertiesDynamicObjectAdapter(delegate.getClass(), convention.getExtraProperties());

        updateDelegates();
    }

    private void updateDelegates() {
        DynamicObject[] delegates = new DynamicObject[6];
        delegates[0] = dynamicDelegate;
        delegates[1] = extraPropertiesDynamicObject;
        int idx = 2;
        if (beforeConvention != null) {
            delegates[idx++] = beforeConvention;
        }
        if (convention != null) {
            delegates[idx++] = convention.getExtensionsAsDynamicObject();
        }
        if (afterConvention != null) {
            delegates[idx++] = afterConvention;
        }
        boolean addedParent = false;
        if (parent != null) {
            addedParent = true;
            delegates[idx++] = parent;
        }
        DynamicObject[] objects = new DynamicObject[idx];
        System.arraycopy(delegates, 0, objects, 0, idx);
        setObjects(objects);

        if (addedParent) {
            idx--;
            objects = new DynamicObject[idx];
            System.arraycopy(delegates, 0, objects, 0, idx);
            setObjectsForUpdate(objects);
        }
    }

    public void addObject(DynamicObject object, Location location) {
        switch (location) {
            case BeforeConvention:
                beforeConvention = object;
                break;
            case AfterConvention:
                afterConvention = object;
        }
        updateDelegates();
    }
    ......
```
ExtensibleDynamicObject 也是个不干活的，它是一个 DynamicObject 容器，它把 DefaultProject 封装成 BeanDyanmicObject，把 convention.getExtraProperties() 封装成 ExtraPropertiesDynamicObjectAdapter，还通过 convention.getExtensionsAsDynamicObject() 获取了 
ExtensionsDynamicObject。在 DefaultProject 的构造函数中我们看到，它还通过 addObject 获取了 taskContainer.getTasksAsDynamicObject()。所以，脚本中所有对 ExtensibleDynamicObject 的方法调用和属性访问，都又会转包给这些对象。

现在我们看看 getExtraProperties()：

```java
public interface ExtensionContainer {
    ......
    /**
     * The extra properties extension in this extension container.
     *
     * This extension is always present in the container, with the name “ext”.
     *
     * @return The extra properties extension in this extension container.
     */
    ExtraPropertiesExtension getExtraProperties();
}
```

# Ext DSL 的支持

很显然，ExtraPropertiesExtension 代表 gradle 中的 ext 扩展，通常我们利用这个扩展设置一些属性。ExtraPropertiesDynamicObjectAdapter 对它进行了封装，然后将属性暴露给脚本，我们在脚本中对 ext 中属性的访问都将透过 ExtraPropertiesDynamicObjectAdapter 来进行，这就是我们为什么可以在脚本中直接使用 ext 块中定义的属性的原因。

我们现在看看 convention.getExtensionsAsDynamicObject() 返回的 ExtensionsDynamicObject。

```java
private class ExtensionsDynamicObject extends AbstractDynamicObject {
        @Override
        public String getDisplayName() {
            return "extensions";
        }

        @Override
        public boolean hasProperty(String name) {
            if (extensionsStorage.hasExtension(name)) {
                return true;
            }
            ......
            return false;
        }

        @Override
        public Map<String, Object> getProperties() {
            Map<String, Object> properties = new HashMap<String, Object>();
            ......
            properties.putAll(extensionsStorage.getAsMap());
            return properties;
        }

        @Override
        public DynamicInvokeResult tryGetProperty(String name) {
            Object extension = extensionsStorage.findByName(name);
            if (extension != null) {
                return DynamicInvokeResult.found(extension);
            }
            ......
            return DynamicInvokeResult.notFound();
        }

        public Object propertyMissing(String name) {
            return getProperty(name);
        }

        @Override
        public DynamicInvokeResult trySetProperty(String name, Object value) {
            checkExtensionIsNotReassigned(name);
            if (plugins == null) {
                return DynamicInvokeResult.notFound();
            }
            ......
            return DynamicInvokeResult.notFound();
        }

        public void propertyMissing(String name, Object value) {
            setProperty(name, value);
        }

        @Override
        public DynamicInvokeResult tryInvokeMethod(String name, Object... args) {
            if (isConfigureExtensionMethod(name, args)) {
                return DynamicInvokeResult.found(configureExtension(name, args));
            }
            ......
            return DynamicInvokeResult.notFound();
        }

        public Object methodMissing(String name, Object args) {
            return invokeMethod(name, (Object[]) args);
        }

        @Override
        public boolean hasMethod(String name, Object... args) {
            if (isConfigureExtensionMethod(name, args)) {
                return true;
            }
            ......
            return false;
        }
        ......

        private Object configureExtension(String name, Object[] args) {
            Closure closure = (Closure) args[0];
            Action<Object> action = ConfigureUtil.configureUsing(closure);
            return extensionsStorage.configureExtension(name, action);
        }
}
```

# 扩展相关 DSL 的支持

可以看到，当访问属性时，这个对象会寻找对应的扩展，然后将扩展返回，所以我们可以在脚本中直接通过扩展名获取扩展，比如 java.sourceCompatibility；当调用方法时，会使用 configureExtension 对扩展进行配置，这里会假设 args 只包含一个 Closure 类型的参数，ExtensionsStorage#configureExtension 方法的逻辑是：先根据 name 找到对应的扩展，然后对扩展用前面的 Closure 类型的参数进行配置，这就是为什么我们可以使用 java {} 这种方式配置 java 扩展的原因。当我们使用 java {} 语法配置 java 扩展时，此语法结构会被 Groovy 会编译成对脚本对象 DefaultScript#invokeMethod() 方法的调用。调用参数为：invokeMethod(name: "java", args: closure)。前面分析过了，DefaultScript 最终会把调用转发给 ScriptDynamicObject，而 ScriptDynamicObject 又会把调用转发给 ExtensibleDynamicObject，而 ExtensibleDynamicObject 是一个 DynamicObject 容器，它会继续把调用转发给容器中的 DynamicObject，当调用转发到 ExtensionsDynamicObject 时，会根据 name 获取到 java 扩展，然后对 java 扩展进行配置。Gradle 会把扩展对象作为 closure 的 delegate，这样一来，closure 内就可以访问到扩展的属性和方法了。

# Plugin 相关 DSL 的支持


# Task 相关 DSL 的支持

