---
layout: post
title:  "浅析C++与Lua数据交互层"
date:   2019-02-24 18:13:00 +0800
categories: tech
---

## 背景

​	C++作为服务器端开发的主流语言，相比其他语言有明显的性能优势，但是它的技术门槛比较高，不支持热更新，无法进行迅速的需求迭代和线上问题的响应。另外，C++内存管理的特点，技术人员对指针的使用不当，可能会造成宕机。因此，现在越来越多的技术团队会选择在C++中嵌入脚本语言，C++提供底层支持，脚本语言进行业务逻辑的开发。这样的好处，一方面将开发人员进行划分，C++代码的维护只由核心人员进行。业务层的脚本代码开发，相对比较简单，可以交由经验比较浅的新人，甚至外包给其他团队；另外一方面，脚本语言的灵活性，可热更新，又弥补了C++的不足。

​	Lua作为一门非常轻量的脚本语言，被大量的游戏团队使用。我们就来说一说，C++是如何与Lua进行数据交互的。

​	已有的类似系统有LuaBinder、LuaTinker、CppToLua等。

## 基本原理

​	Lua提供了虚拟栈的概念，C/C++可以通过push/pop从栈上逐一存取基本的数据类型，也可以通过pushcfunction/pushcclosure，将函数注册给Lua，供其调用。但对于C++来说，数据结构比较复杂，有类和对象的概念，类有成员变量和成员函数，类之间又有继承的关系。Lua的基本接口并无法支撑这么复杂的数据交互。我们要实现的数据交互层，就是为C++提供一套注册接口，方面将C++中的类、对象类型的变量、类的成员变量和成员函数注册到Lua中去，让Lua像访问原生数据一样访问它们。

​       如下，在C++中定义了如下类A，并利用数据交互层提供的接口，将A类的成员变量和成员函数注册给Lua。然后，将A类型的对象ca注册到Lua的全局变量中，为了进行区分，命名为la。在Lua代码里，就可以通过la.x访问对象ca.x的变量，通过la:funcA(p1,p2,…)调用ca.funcA的函数了。

```c++
Class A
{
public:
    A();
    ~A();
    
    TypeX x;
    TypeY funcA(Param1 p1, Param2 p2, ...);
};

...
registerClass<A>(L, "A"); // L 为lua_State*类型
registerClassFunction<A>(L, "funcA", A::funcA);
registerClassMemberVariable<A>(L, "x", &A::x);

A ca;
registerVariable(L, "la", &ca);
```

​	这背后的发生了什么？先来看一下`regiserClass`的实现，在Lua的全局环境中注册了名为name的表，并定义几个元方法，我们称这种表为类表。

```c++
template<typename T>
registerClass(lua_State* L, const char* name)
{
    ClassName<T>::name(name); //注册类名
    
    lua_newtable(L);
    
    lua_pushstring(L, "__name");
    lua_pushstring(L, name);
    lua_rawset(L, -3);
    
    lua_pushstring(L, "__index");
    lua_pushcclosure(L, meta_get, 0);
    lua_rawset(L, -3);
    
    lua_pushstring(L, "__newindex");
    lua_pushcclosure(L, meta_set, 0);
    lua_rawset(L, -3);
    
    lua_pushstring(L, "__gc");
    lua_pushcclosure(L, destroyer<T>, 0);
    lua_rawset(L, -3);
    
    lua_setglobal(L, name);
}
```

当访问la.x时，会把la当作table，寻找table中key为"x"的值。la.x当作左值时会触发 `__newindex`元方法，当作右值时会触发 `__index`元方法。当la中没有找到key为"x"的值，就会去la的元表中进行查找。因此，需要将la的元表设置为A的类表。

​	除了上面通过注册全局变量，Lua层还有另外两个途径访问C++类的对象：通过C++类的构造函数直接构造对象、通过其他对象的成员变量或者成员函数的返回值。三种途径的关键一步，都是将userdata的元表关联到注册给Lua的类表中。	

### 一层数据封装

​	Lua中的基本数据类型有number、string、table、function、userdata、thread。la本质上就是userdata，指向内存数据的一个指针，简单理解，可以认为是对象实例ca的地址，实际上，进行了一层封装。分别定义了ValueToUser<T>、PtrToUser<T>、RefToUser<T>三种类型，来处理值类型、指针类型和引用类型，它们都继承自UserBase。这样就可以将任意类型的值、指针、引用压入Lua虚拟栈中。注意到ValueToUser需要负责类型的创建和删除，PtrToUser和RefToUser只是持有对象的地址。

```c++
struct UserBase
{
    UserBase(void* ptr): m_p(ptr) {}
    virtual ~UserBase();
    void* m_p;
};

template<typename T>
struct ValueToUser: UserBase
{
	template<typename ... Args>
	ValueToUser(Args... args) : UserBase(new T(std::forward<Args>(args)...)) {}
	
	~ValueToUser() { delete ((T*)m_p);}
};

template<typename T>
struct PtrToUser: UserBase
{
	PtrToUser(T* t): UserBase((void*)t) {}
};

template<typename T>
struct RefToUser: UserBase
{
	RefToUser(T& t): UserBase(&t) {}
};

```

注意到registerClass中将destroyer<T>函数注册为__gc元方法，当Lua产生垃圾回收时，就会调用该方法：

```C++
template<typame T>
int destroyer(lua_State* L)
{
    ((UserBase*)lua_touserdata(L, -1))->~UserBase();
    return 0; //告诉Lua该函数没有返回值
}
```

### 两个抽象操作

定义两个操作：pushToLuaStack<T>可以将任意类型的数据压入Lua虚拟栈，getFromLuaStack<T>则可以从Lua虚拟栈中取出任意类型的数据。

```
pushToLuaStack<T>
				|
                | EnumToLua<T> ---> 将number类型压入栈
                | ObjectToLua<T> ---> | ValueToLua<T> ---> 将ValueToUser<T>类型入栈
                                      | PtrToLua<T>   ---> 将PtrToUser<T>类型入栈
                                      | RefToLua<T>   ---> 将RefToUser<T>类型入栈 
                                      & 将T的类表作为刚压入栈的userdata的元表
```

```
getFromLuaStack<T>
                |
                | LuaToEnum<T> ---> 从栈上取出number类型变量
				| LuaToObject<T> ----> & 1. user = LuaUserDataToType<UserBase*>::convert()
						               & 2. VoidToType<T>(user->m_p)
						                           |
						                           | T如果是指针类型，VoidToPtr
						                           | T如果是引用类型，VoidToRef
						                           | T如果是值类型，  VoidToValue
```

### 注册成员变量

​	`registerClassMemberVariable<A>(L, "x", &A::x)`在类表中注册了key为"x"的键值对，值是一个void* 类型的userdata，实际是一个VariableBase类型的指针，利用运行时多态和模板，是可以达到反射的目的的。把任意类的任意成员变量，都抽象成VariableBase类，它只有两个虚函数接口，get用来把成员变量的值压入栈中，set用来取出栈上的值并赋给成员变量。VariableBase根据T和V派生出MemberVariable<T, V>类。MemberVariable<T, V>记录了成员变量在类中的偏移地址，同时具现了get和set这两个接口，因为有类型信息，可以知道从栈上取出是什么类型的值。

```C++
struct VariableBase
{
    virtual void get(lua_State* L) = 0;
    virtual void set(lua_State* L) = 0;
};

template<typename T, typename V>
MemberVariable: VariableBase
{
	V T::*_var;
	
	void get(lua_State* L)
    {
    	pushToLuaStack<V>(L, getFromLuaStack<T*>(L, 1)->*(_var));  //从栈底取出对象地址偏移到成员变量所在地址，将成员变量值压入栈中
    }
    
    void set(lua_State* L)
    {
    	getFromLuaStack<T*>(L, 1)->*(_var) = getFromLuaStack<V>(L, 3);
    }
};

int meta_get(lua_State* L)
{
    lua_getmetatable(L, 1); // 将栈底的元表压栈，例子中就是la的类表A
    lua_pusvalue(L, 2);     // 将栈底上面的元素，就是key复制压入栈；例子中就是"x"
    lua_rawget(L, -2);      // 得到A[x]的值
    
    if(lua_isuserdata(L, -1))
    {
        LuaUserDataToType<VariableBase*>::convert(L, -1)->get(L); //将栈底的userdata转
        lua_remove(L, -2); // 将类表移出栈
    }
    ...
}

int meta_set(lua_State* L)
{
    lua_getmetatable(L, 1); // 将栈底的元表压栈，例子中就是la的类表A
    lua_pusvalue(L, 2);     // 将栈底上面的元素，就是key复制压入栈；例子中就是"x"
    lua_rawget(L, -2);      // 得到A[x]的值
    
    if(lua_isuserdata(L, -1))
    {
        LuaUserDataToType<VariableBase*>::convert(L, -1)->set(L);
    }
    ...
    lua_settop(L, 3);
}
```

### 注册成员函数

​	`registerClassFunction<A>(L, "funcA", A::funcA)`做了什么？在类表中注册了key为"funcA"的键值对，值是一个闭包。`lua_pushcclosure(L，func, n)`函数允许将一个函数func压入栈的同时，指定它之前栈上的n个元素作为upvalue，这就是闭包的概念。这里的闭包是模版类MemberFunctionDelegate<Ret, T, …Args>的静态函数——call函数，upvalue是A::funcA的函数指针。模板类通过模板参数，将所属类T，返回值类型Ret，以及函数的参数列表的类型都记录下来。当la:funcA调用时，实际调用的是`MemberFunctionDelegate<Ret, T, …Args>::call`函数，Lua会将la(指向ca的指针)和函数的参数依次压入栈中。call的实现，首先从upvalue中取出成员函数指针，再从栈低到栈顶依次取出，ca的地址和A::funcA的各个参数，有了这些，就能真正地调用ca:funcA(p1,p2,…)，并将函数的返回值压入栈。

```C++
template<typename RVal, typename T, typename ... Args>
struct MemberFunctionDelegate
{
    static int call(lua_State* L)
    {
        typedef RVal(T::*MemFunc)(Args ...);
        MemFunc fun = getUpValue<MemFunc>(L); // 从upvalue中取出成员函数指针
        
        int index = 1;
        T* t = getFromLuaStack<T*>(L, index++); //从栈底取出对象地址
        pushToLuaStack(L, (t->*fun)(getFromLuaStack<Args>(L, index++)...));
        
        return 1;
    }
};

template<typename RVal, typename T, typename ... Args>
pushFunctionDelegate(lua_State* L, RVal(T::*func)(Args...))
{
    lua_pushcclosure(L, MemberFunctionDelegate<RVal, T, Args...>::call, 1);
}

template<typename T, typename F>
void registerClassFunction(lua_State* L, const char* name, F func)
{
    push_meta(L, ClassName<T>::name());  // 将T的类表压入栈
    if(lua_istable(L, -1))
    {
        lua_pushstring(L, name);
        new(lua_newuserdata(L, sizeof(F))) F(func); //成员函数指针作为upvalue
        pushFunctionDelegate(L, func); 
        lua_rawset(L, -3);
    }
    lua_pop(L, 1);
}
```

### 注册构造函数

当我们在Lua中将表当作构造表达式使用时，例如t = T(…)，就会调用T的__call元方法。通过下面代码可以看到，注册的构造函数，其实创建了一个ValueToUser<T>类型的userdata，并将它的元表设为T的类表。

```c++
template<typename T, typename ... Args>
int constructor(lua_State* L)
{
    int index = 2;
    new(lua_newuserdata(L, sizeof(ValueToUser<T>))) ValueToUser<T>(getFromLuaStack<Args>(L, index++)...);
    push_meta(L, ClassName<typename ClassType<T>::Type>::name());
    lua_setmetatable(L, -2);
    return 1;
}
...
template<typename T, typename F>
void addClassConstructor(lua_State* L, F func)
{
    push_meta(L, ClassName<T>::name());
    if(lua_istable(L, -1))
    {
        lua_newtable(L);
        lua_pushstring(L, "__call");
        lua_pushcclosure(L, func, 0);
        lua_rawset(L, -3);
        lua_setmetatable(L, -2);
    }
    lua_pop(L, 1);
}

// 注册构造函数示例
addClassConstrutor<A>(L, constructor<A, int>);

// Lua中创建A对象示例
la = A(10);

```

### 注册继承关系

C++中定义如下两个类A和B，B继承自A。

```c++
Class A
{
public:
    A(int a): a(a);
    ~A();
    
    int a;
    TypeY funcA(Param1 p1, Param2 p2, ...);
};

Class B : public A
{
public:
    B(int b) : A(b+1);
    ~B();
    
    int b;
    TypeW funcB(Param p1, Param2 p2, ...);
};

```

通过数据交互层提供的接口，不仅可以注册类的成员变量和成员函数，还可以将继承关系继承到Lua中。

```c++
registerClass<A>(L, "A"); // L 为lua_State*类型
registerClassFunction<A>(L, "funcA", A::funcA);
registerClassMemberVariable<A>(L, "x", &A::x);

registerClass<B>(L, "B");
registerClassFunction<B>(L, "funcB", B::funcB);
registerClassMemberVariable<B>(L, "z", &B::z);
inherientClass<B, A>(L);

A ca;
registerVariable(L, "la", &ca);
```

于是在Lua脚本中，通过实例lb不仅可以访问类B的成员变量和函数，可以访问其父类的。

```lua
lb = B(3);

print(lb.b);
lb:funcB(...);

print(lb.a);
lb:funcA(...);
```

这种继承关系，是如何注册到Lua中去的？

看看inherientClass函数的实现：本质上就是将一个类表的__parent指向了父类表。

```c++
template<typename T, typename P>
void inherientClass(lua_State* L)
{
    push_meta(L, ClassName<T>::name());
    if(lua_istable(L, -1))
    {
        lua_pushstring(L, "__parent");     // 通过__parent元方法，奖励两个类表之间的“继承”关系
        push_meta(L, ClassName<P>::name());
        lua_rawset(L, -3);
    }
    lua_pop(L, 1);
}
```



我们再来看一下完整的meta_get和meta_set函数实现，当在当前类表中查找不到键值对时，就会调用invoke_parent函数，递归地从继承关系链中，向上搜寻父类表或者祖先类表中是否存在键值对。

```c++
static void invoke_parent(lua_State* L)
{
    lua_pushstring(L, "__parent");  // 取出继承关系中的父类表
    lua_rawget(L, -2);
    
    if(lua_istable(L, -1))   // 如果存在父类表
    {
        lua_pushvalue(L, 2); //将栈底上面的元素，即要找的key复制入栈
        lua_rawget(L, -2);   // 尝试从父类表中找到key对应的value是否存在
        
        if(!lua_isnil(L, -1))
        {
            lua_remove(L, -2); //如果找到，将父类表移除
        }
        else
        {
            lua_remove(L, -1); //将栈顶的nil移除
            invoke_parent(L);  //递归调用在父类的父类中查找
            lua_remove(L, -2); //将父类表移除
        }
    }
}
int meta_get(lua_State* L)
{
    lua_getmetatable(L, 1); //将栈底元素的元表入栈
    lua_pushvalue(L, 2);    // 复制key并压入栈
    lua_rawget(L, -3);      // 获取元表[key]的值
    
    if(lua_isuserdata(L, -1)) //如果是userdata
    {
        LuaUserDateToType<VariableBase*>::convert(L, -1)->get(L); //将成员变量的值压入栈
        lua_remove(L, -2);  // 将userdata移除
    }
    else if(lua_isnil(L, -1))
    {
        lua_remove(L, -1); //将nil值从栈顶移除
        invoke_parent(L);  // 从父元表中查找
        
        if(lua_isuserdata(L, -1)) 
        {
            LuaUserDataToType<VariableBase*>::convert(L, -1)->get(L);
            lua_remove(L, -2);
        }
        else if(lua_isnil(L, -1))
        {
            lua_remove(L, -1);
            lua_pushfstring("Fogot registering '%s' class variable?", lua_tostring(L, 2));
            lua_error(L);
        }
    }
    
    lua_remove(L, -2); // 将元表移除
        
    return 1; // 提示栈上有数据返回
}

int meta_set(lua_State* L)
{
    lua_getmetatable(L, 1); //将栈底元素的元表入栈
    lua_pushvalue(L, 2);    // 复制key并压入栈
    lua_rawget(L, -3);      // 获取元表[key]的值
    
    if(lua_isuserdata(L, -1)) //如果是userdata
    {
        LuaUserDateToType<VariableBase*>::convert(L, -1)->set(L); //成员变量赋值
    }
    else if(lua_isnil(L, -1))
    {
        lua_remove(L, -1); //将nil值从栈顶移除
        invoke_parent(L);  // 从父元表中查找
        
        if(lua_isuserdata(L, -1)) 
        {
            LuaUserDataToType<VariableBase*>::convert(L, -1)->set(L);
        }
        else if(lua_isnil(L, -1))
        {
            lua_remove(L, -1);
            lua_pushfstring("Fogot registering '%s' class variable?", lua_tostring(L, 2));
            lua_error(L);
        }
    }
    
    lua_settop(L, 3); 
    
    return 0;
}
```

