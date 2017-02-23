---
layout: post
title: 使用模板和宏静态检查类成员是否存在
date: 2017-02-22
tags: [C++, 模板]
---

早在C++03的时候就用模板的方式实现过类成员函数的静态检查；这两天一个小兄弟又问到我同样问题，使用C++11撸了一个通用版本可以检查变量，函数和类型定义，使用GCC和CLANG编译都没问题，但在VS2015上却出现了迷之错误，前后折腾了半天确定应该是其BUG，重新选定方案撸了另一个版本，全平台兼容，[点此下载](/assets/use_template_and_macro_to_check_for_member_of_class_is_exist/has.cpp)。

### 需求 ###

还是从需求出发：

> 1. 简单易用，大概形式应该这样：HAS_MEMBER(type, name)
> 2. 要同时支持成员变量，成员函数和类型定义，如: HAS_MEMBER(std::vector\<int\>, size), HAS_MEMBER(std::vector\<int\>, value_type)
> 3. 引入函数定义支持重载函数，函数定义为了书写方便采用引用方式，如: HAS_MEMBER(std::vector\<int\>, push_back, void(const int&))；对于非重载函数也可检验其是否符合函数定义，如: HAS_MEMBER(std::vector\<int\>, size, size_t())为false (正确应为: size_t() const)

测试先行：

> ```c++
> HAS_MEMBER_DEF(push_back);
> HAS_MEMBER_DEF(size);
> HAS_MEMBER_DEF(value_type);
> HAS_MEMBER_DEF(first);
> HAS_MEMBER_DEF(x);
>
> int main() {
>   static_assert(HAS_MEMBER(std::vector<int>, push_back) == false, "重载函数");
>   static_assert(HAS_MEMBER(std::vector<int>, push_back, void(const int &)), "重载函数指定函数类型");
>   static_assert(HAS_MEMBER(std::vector<int>, size), "非重载函数");
>   static_assert(HAS_MEMBER(std::vector<int>, size, size_t()) == false, "非重载函数带错误函数类型");
>   static_assert(HAS_MEMBER(std::vector<int>, size, size_t()const), "非重载函数带正确函数类型");
>   static_assert(HAS_MEMBER(std::vector<int>, value_type), "类型定义");
>
>   using IntIntPairType = std::pair<int, int>;  // 解决宏分不清 "," 的问题
>   static_assert(HAS_MEMBER(IntIntPairType, first), "成员函数");
>   static_assert(HAS_MEMBER(IntIntPairType, x) == false, "不存在");
> }
> ```

### 初始版本 ###

这个实现在CLANG和GCC编译器下都OK，但在VS2015下会出现迷之错误，通过IDE智能提示查看到的值都是对的，但一编译就有问题，感觉应该是其decltype关键字实现上有BUG:

> ```c++
> namespace detail {
> // 引入C++17中的std::void_t实现
> template <class...> struct _Param_tester { using type = void; };
> template <class... _Types>
> using void_t = typename _Param_tester<_Types...>::type;
> }
>
> #define HAS_MEMBER_DEF(name)                                                   \
>   namespace detail {                                                           \
>   template <class T, class F = void, class V = void>                           \
>   struct HasMember_##name : std::false_type {};                                \
>   template <class T, class F>                                                  \
>   struct HasMember_##name<T, F, void_t<decltype(std::mem_fn<F>(&T::name))>>    \
>       : std::true_type {};                                                     \
>   template <class T>                                                           \
>   struct HasMember_##name<T, void, void_t<decltype(&T::name)>>                 \
>       : std::true_type {};                                                     \
>   template <class T>                                                           \
>   struct HasMember_##name<T, void, void_t<typename T::name>>                   \
>       : std::true_type {};                                                     \
>   }
>
> #define HAS_MEMBER(type, name, ...)                                            \
>   detail::HasMember_##name<type, ##__VA_ARGS__, void>::value
> ```

### 终极实现 ###

> ```c++
> #define HAS_MEMBER_DEF(name)                                                   \
>   namespace detail {                                                           \
>   template <class T, class F = void> class HasMember_##name {                  \
>     template <class _Ux>                                                       \
>     static std::true_type                                                      \
>     Check(int,                                                                 \
>           typename std::enable_if<std::is_void<F>::value,                      \
>                                   decltype(&_Ux::name) *>::type = nullptr);    \
>     template <class _Ux>                                                       \
>     static std::true_type                                                      \
>     Check(int, decltype(std::mem_fn<F>(&_Ux::name)) * = nullptr);              \
>     template <class _Ux>                                                       \
>     static std::true_type Check(int, typename _Ux::name * = nullptr);          \
>     template <class _Ux> static std::false_type Check(...);                    \
>                                                                                \
>   public:                                                                      \
>     static constexpr bool value = decltype(Check<T>(0))::value;                \
>   };                                                                           \
>   }
>
> #define HAS_MEMBER(type, name, ...)                                            \
>   detail::HasMember_##name<type, ##__VA_ARGS__>::value
> ```

### 更多实现 ###

C++17引入的[std::is_detected](http://en.cppreference.com/w/cpp/experimental/is_detected){:target="_blank"}在大多数情况下，提供了解决这类问题的一个标准方式：

> ```c++
> template<typename T>
> using toString_t = decltype( std::declval<T&>().toString() );
>
> template<typename T>
> constexpr bool has_toString = std::is_detected_v<T, toString_t>;
> ```

此外[Boost.TTI](http://www.boost.org/doc/libs/1_55_0/libs/tti/doc/html/index.html){:target="_blank"}实现方式和我上面的类似：

> ```c++
> #include <boost/tti/has_member_function.hpp>
> BOOST_TTI_HAS_MEMBER_FUNCTION(toString)
> ...
> constexpr bool foo = has_member_function_toString<T, std::string>::value;
> ```

### 使用 ###

在C++14中(如果不使用template lambda可以兼容C++11，只是写起来要麻烦些)，可以这样用：

> ```c++
> template <bool C, class Ft, class Ff, class... Args>
> typename std::enable_if<C, size_t>::type Exec(Ft &&ft, Ff &&ff, Args &&... args) {
>   return std::forward<Ft>(ft)(std::forward<Args>(args)...);
> }
> template <bool C, class Ft, class Ff, class... Args>
> typename std::enable_if<!C, size_t>::type Exec(Ft &&ft, Ff &&ff, Args &&... args) {
>   return std::forward<Ff>(ff)(std::forward<Args>(args)...);
> }
> template <class T> auto Size(const T &obj) {
>   // 使用了template lamabda需要使用C++14选项编译
>   return Exec<HAS_MEMBER(T, size)>(
>     [](const auto &obj) { return obj.size(); },
>     [](const auto &obj) { return -1; }, obj);
> }
>
> int main() {
>   std::vector<int> a;
>   std::pair<int, int> b;
>   assert(Size(a) == 0);
>   assert(Size(b) == size_t(-1));
> }
> ```

配合C++17的[if constexpr](http://www.oschina.net/translate/final-features-of-c17){:target="_blank"}特性，我们可以更优雅：

> ```c++
> template<class T>
> size_t Size(const T& obj)
> {
>     if constexpr (HAS_MEMBER(T, size))
>         return obj.size();
>     else
>         return -1;
> }
> ```

### 后续 ###

其实早在远古时代，MSVC就提供了非标准扩展[__if_exists](https://msdn.microsoft.com/en-us/library/x7wy9xh3.aspx){:target="_blank"}关键字来实现上述的功能，不仅仅能检查类成员，还可以检查全局函数，全局变量，局部变量，但我对非标的东西一直持排斥态度；于是实现了一个全局检测版本，在GCC，CLANG上不能编译，但在VS2015上能完美运行：

> ```c++
> #define HAS_DEF(name)                                                          \
>   namespace detail {                                                           \
>   template <class F = void> class Has_##name {                                 \
>     template <class _Ux>                                                       \
>     static std::true_type                                                      \
>     Check(int, typename std::enable_if<std::is_void<F>::value,                 \
>                                        decltype(&name) *>::type = nullptr);    \
>     template <class _Ux>                                                       \
>     static std::true_type                                                      \
>     Check(int, decltype(std::bind<void, typename std::decay<F>::type>(         \
>                    &name)) * = nullptr);                                       \
>     template <class _Ux> static std::false_type Check(...);                    \
>                                                                                \
>   public:                                                                      \
>     static constexpr bool value = decltype(Check<void>(0))::value;             \
>   };                                                                           \
>   }
> #define HAS(name, ...) detail::Has_##name<##__VA_ARGS__>::value
>
> #if defined(_MSC_VER) && (_MSC_VER >= 1900)
> void Print() {}
> void Print(int) {}
> void X() {}
> int aaa;
>
> HAS_DEF(Print)
> HAS_DEF(Print1)
> HAS_DEF(X)
> HAS_DEF(aaa)
> HAS_DEF(bbb)
> #endif
>
> int main() {
> #if defined(_MSC_VER) && (_MSC_VER >= 1900)
>   static_assert(HAS(Print) == false, "重载函数");
>   static_assert(HAS(Print, void(int)), "重载函数指定函数类型");
>   static_assert(HAS(X), "非重载函数");
>   static_assert(HAS(X, void(int)) == false, "非重载函数带错误函数类型");
>   static_assert(HAS(X, void()), "非重载函数带正确函数类型");
>   static_assert(HAS(aaa), "变量");
>   static_assert(HAS(bbb) == false, "不存在");
> #endif
> }
> ```

<br/>

> [原始链接]({{page.url}}) 版权声明：自由转载-非商用-非衍生-保持署名 \| [Creative Commons BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)
