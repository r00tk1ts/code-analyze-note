C++ json库在设计上所要解决的核心问题就在于：如何提供优雅直观的接口去像JS,Python那样操作json对象。在此基础上，再将性能发挥到极致。

Json本身是一个字符串，它有着特定的制式，为了能够灵活操作，库设计者需要去把它解析成一个对象，使用设计好的接口去便捷高效操作它。

C++发展了这么多年，诞生了许多知名的json库，这其中，热度最高的便是：[nlohmann/json](https://github.com/sponsors/nlohmann)

本文旨在对该开源仓库进行深度剖析，源码之前，了无秘密。

# Json类的核心设计

用于表达json对象的`class`是`basic_json`类模板，它的定义有5k行代码，看着相当唬人。但实际上，只要我们掌握阅读一个类的方法，了解它的能力，就会发现拆解源代码的实现工作量并不大。

## Json的类型表达

拆解类模板设计的第一步就是看它的模板头声明与类继承关系。

Json的解析无非就是对几种可枚举的类型进行Key/Value的处理：
- 对象
- 数组
- 字符串
- 布尔
- 有符号整型数
- 无符号整型数
- 浮点数
- 二进制bytes

在C++中，每种类型都很容易联想到对应的基础类型或是标准容器。比如对象适合用`std::map`，数组适合用`std::vector`，字符串适合用`std::string`。其他几个也各有合适的基础类型。但直接就绑定使用这些类型未免过于僵硬，库的设计往往都要考虑灵活的延展性，因此，`basic_json`在设计上，允许通过模板参数来指定每一种类型，这在模板元编程中，术语叫做strategy。

```cpp
template<template<typename U, typename V, typename... Args> class ObjectType =  
         std::map,  
         template<typename U, typename... Args> class ArrayType = std::vector,  
         class StringType = std::string, class BooleanType = bool,  
         class NumberIntegerType = std::int64_t,  
         class NumberUnsignedType = std::uint64_t,  
         class NumberFloatType = double,  
         template<typename U> class AllocatorType = std::allocator,  
         template<typename T, typename SFINAE = void> class JSONSerializer =  
         adl_serializer,  
         class BinaryType = std::vector<std::uint8_t>, 
         class CustomBaseClass = void>  
class basic_json 
	: public ::nlohmann::detail::json_base_class<CustomBaseClass> {
	......
};
```

基础类型的指派是一目了然的，这里重点说下`ObjectType`和`ArrayType`，这两个都是模板模板参数，对于前者来说，由于我们只关心对象的KV类型，所以模板参数的前两个`U`和`V`要固定下来，为了能够支持更灵活的类型，额外再提供一个可变模板参数`Args`，后者亦然。`std::map`和`std::vector`满足二者的模板参数列表，它们分别被指定为默认值。

除了类型相关的模板参数以外，这里还指定了默认的内存分配器类型为`std::allocator`，默认的序列化类型为库实现的`adl_serializer`，默认的二进制类型`BinaryType`为`std::vector<std::uint8_t>`，默认的`CustomBaseClass`为`void`。`CustomBaseClass`见名知意，设计的本意是让`basic_json`可以继承用户自定义的某个基类，可能是想继承到某些能力，或是和其他库设计的某些类混用，适用多态的能力。由于默认情况下模板参数指定的是`void`，而在C++语法规则里是不能去继承`void`类型的，因此这里使用type trait技巧包装了一下：

```cpp
struct json_default_base {};  
  
template<class T>  
using json_base_class = typename std::conditional <  
                        std::is_same<T, void>::value,  
                        json_default_base,  
                        T  
                        >::type;
```

实际上直接写作`class CustomBaseClass = detail::json_default_base`达成的效果完全一样，只是`void`更能表达设计的意图。

## 核心类型定义

类模板在设计时，不论是strategy/policy型模板参数，还是直接引用的外部类型，直接使用都较为臃肿。惯例上，会直接（别名）或间接（Type Trait）定义出核心类内类型，一方面可以简化模板实现代码的编写，另一方面也可以屏蔽外部的变化并约束引用的类型。

而想要了解哪些类内定义最为核心，可以从先寻找`basic_json`内部的成员变量出发。`basic_json`类内实际上只有一个成员变量：

```cpp
class basic_json{
	struct data  
	{  
	    /// the type of the current element  
	    value_t m_type = value_t::null;  
	  
	    /// the value of the current element  
	    json_value m_value = {};  
	  
	    data(const value_t v)  
	        : m_type(v), m_value(v)  
	    {  
	    }  
	  
	    data(size_type cnt, const basic_json& val)  
	        : m_type(value_t::array)  
	    {  
	        m_value.array = create<array_t>(cnt, val);  
	    }  
		// 允许默认构造和移动构造，但禁止拷贝
	    data() noexcept = default;  
	    data(data&&) noexcept = default;  
	    data(const data&) noexcept = delete;  
	    data& operator=(data&&) noexcept = delete;  
	    data& operator=(const data&) noexcept = delete;  
	  
	    ~data() noexcept  
	    {  
	        m_value.destroy(m_type);  
	    }  
	};  

	data m_data = {};
};
```
内部类型`data`包裹了整个`basic_json`最为核心的两个类型设计：`value_t`和`json_value`，前者用于表示该`basic_json`对象具体是哪种类型，后者则存储对应的值。

### `m_type`

显然，对于`value_t`来说，在设计上应该是枚举，我们来看看`value_t`的定义：

```cpp
class basic_json {
public:  
  using value_t = detail::value_t;
};

enum class value_t : std::uint8_t  
{  
    null,             ///< null value  
    object,           ///< object (unordered set of name/value pairs)  
    array,            ///< array (ordered collection of values)  
    string,           ///< string value  
    boolean,          ///< boolean value  
    number_integer,   ///< number value (signed integer)  
    number_unsigned,  ///< number value (unsigned integer)  
    number_float,     ///< number value (floating-point)  
    binary,           ///< binary array (ordered collection of bytes)  
    discarded         ///< discarded by the parser callback function  
};
```
果然与我们的猜想如出一辙。

### `m_value`

再来看看`json_value`:

```cpp
class basic_json {
  private:
	union json_value  
	{  
	    /// object (stored with pointer to save storage)  
	    object_t* object;  
	    /// array (stored with pointer to save storage)  
	    array_t* array;  
	    /// string (stored with pointer to save storage)  
	    string_t* string;  
	    /// binary (stored with pointer to save storage)  
	    binary_t* binary;  
	    /// boolean  
	    boolean_t boolean;  
	    /// number (integer)  
	    number_integer_t number_integer;  
	    /// number (unsigned integer)  
	    number_unsigned_t number_unsigned;  
	    /// number (floating-point)  
	    number_float_t number_float;
	    ......
	};
  public:
	// 借助ObjectType这个类模板模板参数，将object_t完整定义出来
	// json对象的key是StringType，value则是basic_json本身
	// ObjectType默认实参是std::map
	// 后面的Compare和Allocator参数则完全照搬std::map的默认模板参数
	using object_t = ObjectType<StringType,  
	  basic_json,  
	  std::less<StringType>,  
	  AllocatorType<std::pair<const StringType,  
	  basic_json>>>;	
	// 如法炮制出数组类型，默认实参是std::vector，element类型为basic_json本身
	using array_t = ArrayType<basic_json, AllocatorType<basic_json>>;
	using string_t = StringType;
	using boolean_t = BooleanType;
	using number_integer_t = NumberIntegerType;
	using number_unsigned_t = NumberUnsignedType;
	using number_float_t = NumberFloatType;
	using binary_t = nlohmann::byte_container_with_subtype<BinaryType>;
};```
因为一个`basic_json`对象在同一时刻只能是上述类型的某一种，一整个json字符串应该是自顶向下，由多个类型各异的`basic_json`对象嵌套而成，因此，`json_value`在设计上使用`union`是自然而然的。每种类型的定义通过模板参数来实现，其中`binary_t`较为复杂，它单独做了一层封装，我们暂且按下不表，留待后续讨论。

在`data`中，`m_type`用于表示该`basic_json`对象具体是哪种类型，`m_value`则存储对应的具体值。

我们还观察到`data`有个很特别的构造器：

```cpp
data(const value_t v)  : m_type(v), m_value(v)  {} 
```
在成员初始化列表中，`m_type`的初始化是显而易见的，`m_value`则有着特定的初始化构造逻辑：

```cpp
json_value(value_t t)  
{  
    switch (t)  
    {  
        case value_t::object:  
        {  
            object = create<object_t>();  
            break;        }  
  
        case value_t::array:  
        {  
            array = create<array_t>();  
            break;        }  
  
        case value_t::string:  
        {  
            string = create<string_t>("");  
            break;        }  
  
        case value_t::binary:  
        {  
            binary = create<binary_t>();  
            break;        }  
  
        case value_t::boolean:  
        {  
            boolean = static_cast<boolean_t>(false);  
            break;        }  
  
        case value_t::number_integer:  
        {  
            number_integer = static_cast<number_integer_t>(0);  
            break;        }  
  
        case value_t::number_unsigned:  
        {  
            number_unsigned = static_cast<number_unsigned_t>(0);  
            break;        }  
  
        case value_t::number_float:  
        {  
            number_float = static_cast<number_float_t>(0.0);  
            break;        }  
  
        case value_t::null:  
        {  
            object = nullptr;   
            break;  
        }  
  
        case value_t::discarded:  
        default:  
        {  
            object = nullptr;    
            if (JSON_HEDLEY_UNLIKELY(t == value_t::null))  
            {  
                JSON_THROW(other_error::create(500, "961c151d2e87f2686a955a9be24d316f1362bf21 3.11.3", nullptr));
            }  
            break;  
        }  
    }
}

template<typename T, typename... Args>  
static T* create(Args&& ... args)  
{  
    AllocatorType<T> alloc;  
    using AllocatorTraits = std::allocator_traits<AllocatorType<T>>;  
  
    auto deleter = [&](T * obj)  
    {  
        AllocatorTraits::deallocate(alloc, obj, 1);  
    };  
    std::unique_ptr<T, decltype(deleter)> obj(AllocatorTraits::allocate(alloc, 1), deleter);  
    AllocatorTraits::construct(alloc, obj.get(), std::forward<Args>(args)...);  
    JSON_ASSERT(obj != nullptr);  
    return obj.release();  
}
```

对于上述`data`构造函数来说，构造的`m_value`就是特定类型的默认零值。对于指针型成员，这里在类内函数模板`create`来创建对象，由于`basic_json`可以支持用户去指定`AllocatorType`，故这里也要沿用模板参数所指定的内存分配器。

当然，`json_value`还应支持各种特定类型作为参数的构造器，诸如：

```cpp
/// constructor for strings  
json_value(const string_t& value) : string(create<string_t>(value)) {}  
  
/// constructor for rvalue strings  
json_value(string_t&& value) : string(create<string_t>(std::move(value))) {}

/// constructor for objects  
json_value(const object_t& value) : object(create<object_t>(value)) {}  
  
/// constructor for rvalue objects  
json_value(object_t&& value) : object(create<object_t>(std::move(value))) {}

......
```

按照惯例，设计出接受常量左值引用和右值引用的两个版本，各个类型俱是如此，不多赘述。

此外，`json_value`的构造器既然负责了指针指向对象的构造，那么自然也要有个对称的释放接口，即`destroy`:

```cpp
void destroy(value_t t)  
{  
	// 卫语句
    if (  
        (t == value_t::object && object == nullptr) ||  
        (t == value_t::array && array == nullptr) ||  
        (t == value_t::string && string == nullptr) ||  
        (t == value_t::binary && binary == nullptr)  
    )  
    {  
        //not initialized (e.g. due to exception in the ctor)  
        return;  
    }  
    // 数组和对象型会有嵌套，我们需要自底向上release
    // 这里用BFS处理，使用stack来存储子节点，LIFO，倒序析构
    if (t == value_t::array || t == value_t::object)  
    {  
        // flatten the current json_value to a heap-allocated stack  
        std::vector<basic_json> stack;  
  
        // move the top-level items to stack  
        if (t == value_t::array)  
        {  
            stack.reserve(array->size());  
            std::move(array->begin(), array->end(), std::back_inserter(stack));  
        }  
        else  
        {  
            stack.reserve(object->size());  
            for (auto&& it : *object)  
            {  
                stack.push_back(std::move(it.second));  
            }  
        }  
		// stack内需要释放的obj全部move给局部变量current_item
		// current_item仅能活过一次循环，每次循环退出前触发basic_json析构
		// 进而触发data的析构，执行m_value.destroy(m_type)
        while (!stack.empty())  
        {  
            // move the last item to local variable to be processed  
            basic_json current_item(std::move(stack.back()));  
            stack.pop_back();  
  
            // if current_item is array/object, move  
            // its children to the stack to be processed later            if (current_item.is_array())  
            {  
                std::move(current_item.m_data.m_value.array->begin(), current_item.m_data.m_value.array->end(), std::back_inserter(stack));  
  
                current_item.m_data.m_value.array->clear();  
            }  
            else if (current_item.is_object())  
            {  
                for (auto&& it : *current_item.m_data.m_value.object)  
                {  
                    stack.push_back(std::move(it.second));  
                }  
  
                current_item.m_data.m_value.object->clear();  
            }  
  
            // it's now safe that current_item get destructed  
            // since it doesn't have any children        }  
    }  
	// 走到这里，object或array型json的所有子对象就全部释放干净了
	// 现在来分解他自己
    switch (t)  
    {  
        case value_t::object:  
        {  
            AllocatorType<object_t> alloc;  
            std::allocator_traits<decltype(alloc)>::destroy(alloc, object);  
            std::allocator_traits<decltype(alloc)>::deallocate(alloc, object, 1);  
            break;        }  
  
        case value_t::array:  
        {  
            AllocatorType<array_t> alloc;  
            std::allocator_traits<decltype(alloc)>::destroy(alloc, array);  
            std::allocator_traits<decltype(alloc)>::deallocate(alloc, array, 1);  
            break;        }  
  
        case value_t::string:  
        {  
            AllocatorType<string_t> alloc;  
            std::allocator_traits<decltype(alloc)>::destroy(alloc, string);  
            std::allocator_traits<decltype(alloc)>::deallocate(alloc, string, 1);  
            break;        }  
  
        case value_t::binary:  
        {  
            AllocatorType<binary_t> alloc;  
            std::allocator_traits<decltype(alloc)>::destroy(alloc, binary);  
            std::allocator_traits<decltype(alloc)>::deallocate(alloc, binary, 1);  
            break;        }  
  
        case value_t::null:  
        case value_t::boolean:  
        case value_t::number_integer:  
        case value_t::number_unsigned:  
        case value_t::number_float:  
        case value_t::discarded:  
        default:  
        {  
            break;  
        }  
    }  
}
```

实际上整个`destroy`的设计就是一个BFS，底层的对象先释放，顶层的后释放，防止关联丢失。

## 序列化与反序列化

对于json库来说，序列化与反序列化是最核心的功能：
- 反序列化：将一个json字符串解析成一棵嵌套的`basic_json`对象树。
- 序列化：将嵌套的`basic_json`对象树转储到字符串json字面量。

这里的反序列化是通过静态成员函数模板`basic_json::parse`来实现，而序列化则是通过`basic_json`的`dump`成员函数来生成一个json `string`：

```cpp
// parse explicitly
auto j3 = json::parse(R"({"happy": true, "pi": 3.141})");

// explicit conversion to string
std::string s = j.dump();    // {"happy":true,"pi":3.141}

// serialization with pretty printing
// pass in the amount of spaces to indent
std::cout << j.dump(4) << std::endl;
// {
//     "happy": true,
//     "pi": 3.141
// }
```

dump还支持一个`indent`参数来控制输出json的结构化，提高可读性。

### `parse`

`parse`的实现实际上有2个重载的成员函数模板，前者是最为泛用的版本，后者则主要兼容STL风格，这二者实际上并无本质区别。

```cpp
template<typename InputType>  
static basic_json parse(InputType&& i,  
                        const parser_callback_t cb = nullptr,  
                        const bool allow_exceptions = true,  
                        const bool ignore_comments = false)  
{  
    basic_json result;  
    parser(detail::input_adapter(std::forward<InputType>(i)), cb, allow_exceptions, ignore_comments).parse(true, result);  
    return result;  
}

// 实际上与前者并无本质区别，只是构造InputAdapterType的入参不同
template<typename IteratorType>  
static basic_json parse(IteratorType first,  
                        IteratorType last,  
                        const parser_callback_t cb = nullptr,  
                        const bool allow_exceptions = true,  
                        const bool ignore_comments = false)  
{  
    basic_json result;  
    parser(detail::input_adapter(std::move(first), std::move(last)), cb, allow_exceptions, ignore_comments).parse(true, result);  
    return result;  
}
```

实际的工作都由内部的`parser`来完成，不急，我们先来看看[接口文档](https://json.nlohmann.me/api/basic_json/parse/)：

`InputType`

A compatible input, for instance:

- an `std::istream` object
- a `FILE` pointer (must not be null)
- a C-style array of characters
- a pointer to a null-terminated string of single byte characters
- a `std::string`
- an object `obj` for which `begin(obj)` and `end(obj)` produces a valid pair of iterators.

`cb` (in)

a parser callback function of type [`parser_callback_t`](https://json.nlohmann.me/api/basic_json/parser_callback_t/) which is used to control the deserialization by filtering unwanted values (optional)

`allow_exceptions` (in)

whether to throw exceptions in case of a parse error (optional, `true` by default)

`ignore_comments` (in)

whether comments should be ignored and treated like whitespace (`true`) or yield a parse error (`false`); (optional, `false` by default)

模板参数`InputType&& i`非常灵活，按照类别拆分，大体可以分成三类：
1. 文件类：同时支持C++标准库的`std::istream`对象和传统C标准库的`FILE`指针
2. 字符串类：
	1. 如C++标准库的`std::string`（也是最常用的）。
	2. 也可以是C风格字符数组，如`char[]`。
	3. 还可以是C风格字符串(以`\0`结尾，一般写成`char* str="{}";`)。
3. 一种对象：它需要支持`begin(obj)`和`end(obj)`操作，并能够生成一组有效的迭代器。

回调对象`cb`则与控制反序列化时不需要的值的过滤有关，它默认是`nullptr`，即无需过滤。`allow_exceptions`默认为`true`，表示在解析json失败时是否要抛出异常。`ignore_comments`则表示是否要忽略注释，如果忽略的话就将其视为空格，否则就抛出一个解析错误，它默认为`false`。

#### `InputAdapterType`

我们先来拆解最内层：`parser`的首个参数。

为了能够支持上述三大类模板参数，`parser`抽象出了一个适配器：`InputAdapterType`作为它的首个入参：

```cpp
template<typename InputAdapterType>  
static ::nlohmann::detail::parser<basic_json, InputAdapterType> parser(  
    InputAdapterType adapter,  
    detail::parser_callback_t<basic_json>cb = nullptr,  
    const bool allow_exceptions = true,  
    const bool ignore_comments = false  
)  
{  
    return ::nlohmann::detail::parser<basic_json, InputAdapterType>(std::move(adapter),  
            std::move(cb), allow_exceptions, ignore_comments);  
}
```

显然，`detail::input_adapter`必定设计成了重载函数，对于不同的`InputType`它会生成各异的`InputAdapterType`，每种`InputAdapterType`都得具有两个核心特性：拥有一个`char_type`的内部类型定义，以及一个具有`int get_character()`签名的成员函数。

我们且来看看`input_adapter`对于不同的`InputType`会生成哪些产物：

##### `FILE*`

```cpp
// 对于FILE指针
inline file_input_adapter input_adapter(std::FILE* file)  
{  
	// 构造一个file_input_adapter对象
    return file_input_adapter(file);  
}  

class file_input_adapter  
{  
  public:  
	// 用于parser内对InputAdapterType做type trait
    using char_type = char;  
  
    explicit file_input_adapter(std::FILE* f) noexcept  
        : m_file(f)  
    {  
        assert(m_file != nullptr);  
    }  
  
    // make class move-only  
    file_input_adapter(const file_input_adapter&) = delete;  
    file_input_adapter(file_input_adapter&&) noexcept = default;  
    file_input_adapter& operator=(const file_input_adapter&) = delete;  
    file_input_adapter& operator=(file_input_adapter&&) = delete;  
    ~file_input_adapter() = default;  

	// 提供get_character接口，每次返回文件的下一个字符
	// 对于fgetc来说，当读到末尾时会返回EOF
	// EOF在C标准中定义，一般是个常量-1，即int 0xFFFFFFFF
    std::char_traits<char>::int_type get_character() noexcept  
    {  
        return std::fgetc(m_file);  
    }  
  
  private:  
    /// the file pointer to read from  
    std::FILE* m_file;  
};
```

返回的对象简明扼要，C标准库的文件操作确实优雅。
##### `std::istream`

```cpp
// 对于std::istream
inline input_stream_adapter input_adapter(std::istream& stream)  
{  
	// 构造一个input_stream_adapter对象
    return input_stream_adapter(stream);  
}  
  
inline input_stream_adapter input_adapter(std::istream&& stream)  
{  
    return input_stream_adapter(stream);  
}

class input_stream_adapter  
{  
  public:  
	// 用于parser内对InputAdapterType做type trait
    using char_type = char;  
  
    ~input_stream_adapter()  
    {  
        // clear stream flags; we use underlying streambuf I/O, do not  
        // maintain ifstream flags, except eof        if (is != nullptr)  
        {  
            is->clear(is->rdstate() & std::ios::eofbit);  
        }  
    }  
  
    explicit input_stream_adapter(std::istream& i)  
        : is(&i), sb(i.rdbuf())  
    {}  
  
    // delete because of pointer members  
    input_stream_adapter(const input_stream_adapter&) = delete;  
    input_stream_adapter& operator=(input_stream_adapter&) = delete;  
    input_stream_adapter& operator=(input_stream_adapter&&) = delete;  
  
    input_stream_adapter(input_stream_adapter&& rhs) noexcept  
        : is(rhs.is), sb(rhs.sb)  
    {  
        rhs.is = nullptr;  
        rhs.sb = nullptr;  
    }  
  
    // std::istream/std::streambuf use 
    // std::char_traits<char>::to_int_type, to ensure that 
    // std::char_traits<char>::eof() and the character 0xFF   
    // do not end up as the same value, e.g. 0xFFFFFFFF. 
    std::char_traits<char>::int_type get_character()  
    {  
        auto res = sb->sbumpc();  
        // set eof manually, as we don't use the istream interface.  
        if (JSON_HEDLEY_UNLIKELY(res == std::char_traits<char>::eof()))  
        {  
            is->clear(is->rdstate() | std::ios::eofbit);  
        }  
        return res;  
    }  
  
  private:  
    /// the associated input stream  
    std::istream* is = nullptr;  
    std::streambuf* sb = nullptr;  
};
```

`get_character`在直觉上应该设计成一个返回`char`型的接口，但为了兼容C的`fgetc`式设计，统一转成了`std::char_traits<char>::int_type`(实际上，这只是其中的一个原因，另一个重要的原因在于兼容宽字符集)，如此，就可以区分`0xFF`这个有效的`char`值和`EOF`了，而对于`std::istream`也是一样。

`std::istream`的设计确实挺逆天的，这里不详细解读了，避免喧宾夺主。

##### `字符数组`与C风格字符串

无论对字符数组或是C风格字符串来说，往往都是使用`char`作为基本型，但考虑到宽字符集等扩展性，`input_adapter`在设计上还是保留了一定的扩展能力。
```cpp
// 受限于模板推导的退化，原生数组的形参需要声明成T(&)[N]形式
template<typename T, std::size_t N>  
auto input_adapter(T (&array)[N]) -> decltype(input_adapter(array, array + N)) 
{  
	// 构造一个input_adapter对象
    return input_adapter(array, array + N);  
}

// 第二个模板参数利用SFINAE对CharT的类型进行了限制
// CharT需要满足如下条件：指针型；不能是数组；底层类型是integral且size为1
// 实际上基本限定了CharT为char，除非你自己diy一个和char类似的类型
template < typename CharT,  
           typename std::enable_if <  
               std::is_pointer<CharT>::value&&  
               !std::is_array<CharT>::value&&  
               std::is_integral<typename std::remove_pointer<CharT>::type>::value&&  
               sizeof(typename std::remove_pointer<CharT>::type) == 1,  
               int >::type = 0 >  
contiguous_bytes_input_adapter input_adapter(CharT b)  
{  
    auto length = std::strlen(reinterpret_cast<const char*>(b));  
    const auto* ptr = reinterpret_cast<const char*>(b);  
    return input_adapter(ptr, ptr + length);  
}

using contiguous_bytes_input_adapter = decltype(input_adapter(std::declval<const char*>(), std::declval<const char*>()));
```

二者构造的都是`input_adapter`对象：

```cpp
// 对于字符串数组来说，IteratorType的类型取决于T，它可以是char，也可以是其他诸如wchar_t等宽字符类型（面向UTF）
// 对于C风格字符串来说，IteratorType就是char*
template<typename IteratorType>  
typename iterator_input_adapter_factory<IteratorType>::adapter_type input_adapter(IteratorType first, IteratorType last)  
{  
	// 经由工厂类型的create接口构造出相应的adapter_type对象
    using factory_type = iterator_input_adapter_factory<IteratorType>;  
    return factory_type::create(first, last);  
}

// 主模板，对单字符类型(如char)适用
template<typename IteratorType, typename Enable = void>  
struct iterator_input_adapter_factory  
{  
    using iterator_type = IteratorType;  
    using char_type = typename std::iterator_traits<iterator_type>::value_type;  
    using adapter_type = iterator_input_adapter<iterator_type>;  
    
	// 最终构造的是一个iterator_input_adapter模板类对象
    static adapter_type create(IteratorType first, IteratorType last)  
    {  
        return adapter_type(std::move(first), std::move(last));  
    }  
};

template<typename T>  
struct is_iterator_of_multibyte  
{  
    using value_type = typename std::iterator_traits<T>::value_type;  
    enum    {  
        value = sizeof(value_type) > 1  
    };  
};  

// 模板偏特化，对于宽字符类型适用
template<typename IteratorType>  
struct iterator_input_adapter_factory<IteratorType, enable_if_t<is_iterator_of_multibyte<IteratorType>::value>>  
{  
    using iterator_type = IteratorType;  
    using char_type = typename std::iterator_traits<iterator_type>::value_type;  
    using base_adapter_type = iterator_input_adapter<iterator_type>;  
    using adapter_type = wide_string_input_adapter<base_adapter_type, char_type>;  
  
    static adapter_type create(IteratorType first, IteratorType last)  
    {  
        return adapter_type(base_adapter_type(std::move(first), std::move(last)));  
    }  
};
```

这里的`enable_if_t`是对标准库模板的别名简化：

```cpp
// 别名模板简化，因为用途上只是为了利用SFINAE机制，对T是什么类型并不关心
// 故可以随意指定成任意类型，这里使用了void
template<bool B, typename T = void>  
using enable_if_t = typename std::enable_if<B, T>::type;
```

我们重点看一下对单字符适用的主模板的`create`生成的是一个怎样的`adapter_type`对象，先看一下`adapter_type`的类型定义：

```cpp
// 使用单字符char举例来说，此处的IteratorType是char*
template<typename IteratorType>  
class iterator_input_adapter  
{  
  public:  
	// 通过iterator_traits得到char
	// 同样用于parser内对InputAdapterType做type trait
    using char_type = typename std::iterator_traits<IteratorType>::value_type;  
	
	// 提供接受一对首位迭代器的构造器
	// 这里的设计和STL的iterator设计理念一致，它本身向下兼容了raw pointer型
    iterator_input_adapter(IteratorType first, IteratorType last)  
        : current(std::move(first)), end(std::move(last))  
    {}  

	// 和input_stream_adapter和file_input_adapter类似
	// 也要提供一个get_character接口用于读取内容
    typename char_traits<char_type>::int_type get_character()  
    {  
        if (JSON_HEDLEY_LIKELY(current != end))  
        {  
            auto result = char_traits<char_type>::to_int_type(*current);  
            std::advance(current, 1);  
            return result;  
        }  
        return char_traits<char_type>::eof();  
    }  
  
  private:  
    IteratorType current;  
    IteratorType end;  
};
```

对于当前主流的编码来说，无论是ASCII还是UTF-8，使用`char`直截了当。但是对于其他
宽字符集，诸如UTF-16或是UTF-32编码，在处理上就显得尤为低效了。作者在设计时充分考虑了这两种情景，对于定长的UTF-16和UTF-32编码（即`sizeof(CharType)`分别为`2`和`4`的情形），则是使用`wide_string_input_adapter`对`iterator_input_adapter`又包了一层，`char_type`依然是`char`，但对于`get_character`来说，它会hook底层`iterator_input_adapter`的同名方法，并在hook逻辑中将UTF-16/32转码成UTF-8，这一工作是在`wide_string_input_helper`中完成的，如此，就达成了："Convert wide character types into individual bytes."的效果。

这一部分的处理并非核心设计，且一旦展开篇幅较大，故有兴趣的读者可以自行走读。

##### std::string

对于标准库的`std::string`来说，适用的重载版本如下：

```cpp
template<typename ContainerType>  
typename container_input_adapter_factory_impl::container_input_adapter_factory<ContainerType>::adapter_type 
input_adapter(const ContainerType& container)  
{  
    return container_input_adapter_factory_impl::container_input_adapter_factory<ContainerType>::create(container);  
}

namespace container_input_adapter_factory_impl  
{  
	// 独立命名空间内使用using导入，移除后续在使用时的std::限定，可以激活ADL
	// 对于用户在独立命名空间中定义的ContainerType，依然可以找到其定义
	using std::begin;  
	using std::end;  
	  
	template<typename ContainerType, typename Enable = void>  
	struct container_input_adapter_factory {};  
	
	// 对于标准库的容器来说，都是支持begin和end操作的
	template<typename ContainerType>  
	struct container_input_adapter_factory< ContainerType,  
	       void_t<decltype(begin(std::declval<ContainerType>()), end(std::declval<ContainerType>()))>> {  
	       
	    using adapter_type = decltype(input_adapter(begin(std::declval<ContainerType>()), end(std::declval<ContainerType>())));  
	    
		// 这里这是简单的元函数转发，使用的依然是接受首尾迭代器的版本
		// 相比于字符数组和C风格字符串，仅仅是IteratorType有所差别
		// std::string::iterator => char*
	    static adapter_type create(const ContainerType& container) {  
			return input_adapter(begin(container), end(container));  
		}  
	};  
}  // namespace container_input_adapter_factory_impl
```

这里的`void_t`也是一种常见SFINAE技巧：
```cpp
template<typename ...Ts> struct make_void  
{  
    using type = void;  
};  
template<typename ...Ts> using void_t = typename make_void<Ts...>::type;
```

只是为了在模板参数推导时，能够产生一个合法的类型，至于这个类型实际上是用不到的。

从设计上也能看出，`ContainerType`相当宽泛，它不一定非要是`std::string`，比如我们用个`std::vector<char>`去装载json字符串也一样可以解析，因为实际上只是在obj上通过`std::begin`和`std::end`嫁接一对首尾迭代器。

> 实际上，仅仅能begin和end还不够，还得满足如下条件：它的迭代器得至少是一个[LegacyInputIterator](https://en.cppreference.com/w/cpp/named_req/InputIterator "cpp/named req/InputIterator").才行。

### parser

到此，我们已经构造好了`InputAdapterType`，它可以是一个`iterator_input_adapter`,`input_stream_adapter`或是`file_input_adapter`等等对象，它们都有两个共有的特性：

- 内部定义了一个`char_type`类型
- 拥有一个`get_character()`成员函数，它每次返回单个数据

进入parser内部：

```cpp
template<typename InputAdapterType>  
static ::nlohmann::detail::parser<basic_json, InputAdapterType> parser(  
    InputAdapterType adapter,  
    detail::parser_callback_t<basic_json>cb = nullptr,  
    const bool allow_exceptions = true,  
    const bool ignore_comments = false  
)  
{  
    return ::nlohmann::detail::parser<basic_json, InputAdapterType>(std::move(adapter),  
            std::move(cb), allow_exceptions, ignore_comments);  
}
```

类静态函数直接调用parser类的构造器，构造一个parser对象，来看看对应的构造器：

```cpp
template<typename BasicJsonType, typename InputAdapterType>  
class parser {
	using number_integer_t = typename BasicJsonType::number_integer_t;  
	using number_unsigned_t = typename BasicJsonType::number_unsigned_t;  
	using number_float_t = typename BasicJsonType::number_float_t;  
	using string_t = typename BasicJsonType::string_t;  
	// 定义词法分析器类型
	using lexer_t = lexer<BasicJsonType, InputAdapterType>;  
	using token_type = typename lexer_t::token_type;
public:
	// 显式构造器
	explicit parser(InputAdapterType&& adapter,  
					const parser_callback_t<BasicJsonType> cb = nullptr,  
					const bool allow_exceptions_ = true,  
					const bool skip_comments = false)  
		: callback(cb)  
		, m_lexer(std::move(adapter), skip_comments)  
		, allow_exceptions(allow_exceptions_)  
	{  
		// read first token  
		get_token();  
	}
	......
private:
	/// get next token from lexer  
	token_type get_token()  
	{  
	    return last_token = m_lexer.scan();  
	}
private:
	/// callback function  
	const parser_callback_t<BasicJsonType> callback = nullptr;  
	/// the type of the last read token  
	token_type last_token = token_type::uninitialized;  
	/// the lexer  
	lexer_t m_lexer;  
	/// whether to throw exceptions in case of errors  
	const bool allow_exceptions = true;
};
```

去掉包装，核心的实现就在于`lexer`这个词法分析器了。

#### `lexer`

`lexer`的构造器只是对入参的转储，同时设置为仅允许移动，不允许拷贝。

```cpp
template<typename BasicJsonType, typename InputAdapterType>  
class lexer : public lexer_base<BasicJsonType> {
	using number_integer_t = typename BasicJsonType::number_integer_t;  
	using number_unsigned_t = typename BasicJsonType::number_unsigned_t;  
	using number_float_t = typename BasicJsonType::number_float_t;  
	using string_t = typename BasicJsonType::string_t;  
	using char_type = typename InputAdapterType::char_type;  
	using char_int_type = typename char_traits<char_type>::int_type;
	// 基类定义的枚举，用于表示当前token的类型
	using token_type = typename lexer_base<BasicJsonType>::token_type;
	
  public:
	explicit lexer(InputAdapterType&& adapter, bool ignore_comments_ = false) noexcept  
	    : ia(std::move(adapter))  
	    , ignore_comments(ignore_comments_)  
	    , decimal_point_char(static_cast<char_int_type>(get_decimal_point()))  
	{}
	......
  private:  
	/// input adapter  
	InputAdapterType ia;  
	
	/// whether comments should be ignored (true) or signaled as errors (false)  
	const bool ignore_comments = false;  
	
	/// the current character  
	char_int_type current = char_traits<char_type>::eof();  
	
	/// whether the next get() call should just return current  
	bool next_unget = false;  
	
	/// the start position of the current token  
	position_t position {};  
	
	/// raw input token string (for error messages)  
	std::vector<char_type> token_string {};  
	
	/// buffer for variable-length tokens (numbers, strings)  
	string_t token_buffer {};  
	
	/// a description of occurred lexer errors  
	const char* error_message = "";  
	
	// number values  
	number_integer_t value_integer = 0;  
	number_unsigned_t value_unsigned = 0;  
	number_float_t value_float = 0;  
	
	/// the decimal point  
	const char_int_type decimal_point_char = '.';
};
```

这里的`token_type`用来表示词法解析时每个token的类型，比如它是一个冒号分隔符，还是某种字面量，比如布尔型的`true`，或是浮点数值等等。

在解析过程中，成员变量`current`和`position`会自左向右持续更新，前者表示单一字符，其类型就是adapter的`get_character()`方法返回的类型，即`int`，后者则记录历史scan的状态，同时可以用于表达当前token的起始位置。结构如下，一目了然：

```cpp
/// struct to capture the start position of the current token  
struct position_t  
{  
    /// the total number of characters read  
    std::size_t chars_read_total = 0;  
    /// the number of characters read in the current line  
    std::size_t chars_read_current_line = 0;  
    /// the number of lines read  
    std::size_t lines_read = 0;  
  
    /// conversion to size_t to preserve SAX interface  
    constexpr operator size_t() const  
    {  
        return chars_read_total;  
    }  
};
```

##### `scan`

我们直接看`scan`方法，它自左向右横向遍历，每次找到并返回下一个有意义的token：

```cpp
// 成功扫描到有效token则返回对应的枚举型，失败则统一返回parse_error
token_type scan()  
{  
    // 首次调用scan接口时，跳过BOM头
    if (position.chars_read_total == 0 && !skip_bom())  
    {  
        error_message = "invalid BOM; must be 0xEF 0xBB 0xBF if given";  
        return token_type::parse_error;  
    }  
  
    // read next character and ignore whitespace  
    // do while跳过所有的\t,\n,\r和空格
    skip_whitespace();  
  
    // ignore comments  
    while (ignore_comments && current == '/')  
    {  
	    // 跳过注释部分，支持/**/的多行注释和//单行注释
	    // scan_comment的实现就是个状态机
        if (!scan_comment())  
        {  
            return token_type::parse_error;  
        }  
  
        // skip following whitespace  
        skip_whitespace();  
    }  
	// 走到这里，才算遍历到一个有意义的字符
    switch (current)  
    {  
        // structural characters  
        // 遇到这几个字符，可以直接定性并返回
        case '[':  
            return token_type::begin_array;  
        case ']':  
            return token_type::end_array;  
        case '{':  
            return token_type::begin_object;  
        case '}':  
            return token_type::end_object;  
        case ':':  
            return token_type::name_separator;  
        case ',':  
            return token_type::value_separator;  
  
        // literals  
        // 单独处理的几个特殊的字面量：true,false,null
        // 它们分别返回literal_true,literal_false,literal_null
        case 't':  
        {  
            std::array<char_type, 4> true_literal = {{static_cast<char_type>('t'), static_cast<char_type>('r'), static_cast<char_type>('u'), static_cast<char_type>('e')}};  
            return scan_literal(true_literal.data(), true_literal.size(), token_type::literal_true);  
        }  
        case 'f':  
        {  
            std::array<char_type, 5> false_literal = {{static_cast<char_type>('f'), static_cast<char_type>('a'), static_cast<char_type>('l'), static_cast<char_type>('s'), static_cast<char_type>('e')}};  
            return scan_literal(false_literal.data(), false_literal.size(), token_type::literal_false);  
        }  
        case 'n':  
        {  
            std::array<char_type, 4> null_literal = {{static_cast<char_type>('n'), static_cast<char_type>('u'), static_cast<char_type>('l'), static_cast<char_type>('l')}};  
            return scan_literal(null_literal.data(), null_literal.size(), token_type::literal_null);  
        }  
  
        // string 
        // 匹配到左双引号，此时需扫描一个字符串 
        // 字符串的处理相对复杂，它需要解析各种转义字符
        // 也是用状态机来实现，最终匹配到一个右双引号结束，返回value_string
        case '\"':  
            return scan_string();  
  
        // number  
        // 对于数字型，只可能是以下字符作为起始，扫描整个数字
        case '-':  
        case '0':  
        case '1':  
        case '2':  
        case '3':  
        case '4':  
        case '5':  
        case '6':  
        case '7':  
        case '8':  
        case '9':  
            return scan_number();  
  
        // end of input (the null byte is needed when parsing from  
        // string literals)  
        // 扫描到字符串终止符，返回end_of_input
        case '\0':  
        case char_traits<char_type>::eof():  
            return token_type::end_of_input;  
  
        // error  
        default:  
            error_message = "invalid literal";  
            return token_type::parse_error;  
    }  
}
```

扫描过程中的核心操作在于`get`和`uget`接口，前者通过adapter的`get_character()`方法读取下一个字符，后者则反向操作：

```cpp
char_int_type get()  
{  
    ++position.chars_read_total;  
    ++position.chars_read_current_line;  

	// next_unget成员用于支持uget接口，adapter实际上是没有去支持uget的
	// 因此，能够被unget的字符实际上只能有一个，即缓存在current成员本身
	if (next_unget)  
    {  
        // just reset the next_unget variable and work with current  
        next_unget = false;  
    }  
    else  
    {  
	    // 未unget时，下一个字符应该从adapter的get_character接口去获取
        current = ia.get_character();  
    }  
  
    if (JSON_HEDLEY_LIKELY(current != char_traits<char_type>::eof()))  
    {  
        token_string.push_back(char_traits<char_type>::to_char_type(current));  
    }  

	// 处理换行符，chars_read_current_line需归0
    if (current == '\n')  
    {  
        ++position.lines_read;  
        position.chars_read_current_line = 0;  
    }  
  
    return current;  
}

void unget()  
{  
	// 通过next_unget标记位和对chars_read_total的--操作，来模拟一次回退
    next_unget = true;  
  
    --position.chars_read_total;  
  
    // 回退状态记录
    if (position.chars_read_current_line == 0)  
    {  
        if (position.lines_read > 0)  
        {  
            --position.lines_read;  
        }  
    }  
    else  
    {  
        --position.chars_read_current_line;  
    }  
  
    if (JSON_HEDLEY_LIKELY(current != char_traits<char_type>::eof()))  
    {  
        JSON_ASSERT(!token_string.empty());  
        token_string.pop_back();  
    }  
}
```

词法分析器在实现上就是自左向右的状态机，当识别到不同的字符时，进行各自的后续处理。这里对比较典型的`scan_number`做一下展开：

```cpp
token_type scan_number()    
{  
	// reset token_buffer to store the number's bytes  
	reset();  

	// the type of the parsed number; initially set to unsigned; will be  
	// changed if minus sign, decimal point or exponent is read       
	token_type number_type = token_type::value_unsigned;  

	// state (init): we just found out we need to scan a number  
	switch (current)  
	{  
		// 实现上使用了大量goto，主要是便于处理状态机
		case '-':  
		{  
			add(current);  
			goto scan_number_minus;  
		}  

		case '0':  
		{  
			add(current);  
			goto scan_number_zero;  
		}  

		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_any1;  
		}  

		// all other characters are rejected outside scan_number()  
		default:            // LCOV_EXCL_LINE  
			JSON_ASSERT(false); // NOLINT(cert-dcl03-c,hicpp-static-assert,misc-static-assert) LCOV_EXCL_LINE  
	}  

scan_number_minus:  
	// state: we just parsed a leading minus sign  
	// 以'-'开头的肯定是有符号数了
	number_type = token_type::value_integer;  
	switch (get())  
	{  
		case '0':  
		{  
			add(current);  
			goto scan_number_zero;  
		}  

		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_any1;  
		}  

		default:  
		{  
			error_message = "invalid number; expected digit after '-'";  
			return token_type::parse_error;  
		}  
	}  

scan_number_zero:  
	// state: we just parse a zero (maybe with a leading minus sign)  
	switch (get())  
	{  
		// 如果0的后面跟着一个'.'，那就是一个浮点数
		case '.':  
		{  
			add(decimal_point_char);  
			goto scan_number_decimal1;  
		}  

		case 'e':  
		case 'E':  
		{  
			add(current);  
			goto scan_number_exponent;  
		}  

		default:  
			goto scan_number_done;  
	}  

scan_number_any1:  
	// state: we just parsed a number 0-9 (maybe with a leading minus sign)  
	switch (get())  
	{  
		case '0':  
		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_any1;  
		}  

		case '.':  
		{  
			add(decimal_point_char);  
			goto scan_number_decimal1;  
		}  

		case 'e':  
		case 'E':  
		{  
			add(current);  
			goto scan_number_exponent;  
		}  

		default:  
			goto scan_number_done;  
	}  

scan_number_decimal1:  
	// state: we just parsed a decimal point  
	number_type = token_type::value_float;  
	switch (get())  
	{  
		// 小数点后跟着的至少得有一个有效数字
		case '0':  
		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_decimal2;  
		}  

		default:  
		{  
			error_message = "invalid number; expected digit after '.'";  
			return token_type::parse_error;  
		}  
	}  

scan_number_decimal2:  
	// we just parsed at least one number after a decimal point  
	switch (get())  
	{  
		// 在一个有效数字之后，不仅可以继续追加数字，还可以有'e','E'作为指数标记
		case '0':  
		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			// 继续向后直到穷尽
			goto scan_number_decimal2;  
		}  

		case 'e':  
		case 'E':  
		{  
			add(current);  
			// 处理指数
			goto scan_number_exponent;  
		}  

		default:  
			goto scan_number_done;  
	}  

scan_number_exponent:  
	// we just parsed an exponent  
	number_type = token_type::value_float;  
	switch (get())  
	{  
		case '+':  
		case '-':  
		{  
			add(current);  
			goto scan_number_sign;  
		}  

		case '0':  
		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_any2;  
		}  

		default:  
		{  
			error_message =  
				"invalid number; expected '+', '-', or digit after exponent";  
			return token_type::parse_error;  
		}  
	}  

// 下面的两个标签与上面的设计思路完全相同
scan_number_sign:  
	// we just parsed an exponent sign  
	switch (get())  
	{  
		case '0':  
		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_any2;  
		}  

		default:  
		{  
			error_message = "invalid number; expected digit after exponent sign";  
			return token_type::parse_error;  
		}  
	}  

scan_number_any2:  
	// we just parsed a number after the exponent or exponent sign  
	switch (get())  
	{  
		case '0':  
		case '1':  
		case '2':  
		case '3':  
		case '4':  
		case '5':  
		case '6':  
		case '7':  
		case '8':  
		case '9':  
		{  
			add(current);  
			goto scan_number_any2;  
		}  

		default:  
			goto scan_number_done;  
	}  

// 在穷尽所有数后会走到这里
scan_number_done:  
	// unget the character after the number (we only read it to know that  
	// we are done scanning a number)        unget();  

	char* endptr = nullptr; // NOLINT(cppcoreguidelines-pro-type-vararg,hicpp-vararg)  
	errno = 0;  

	// try to parse integers first and fall back to floats  
	// 处理无符号和有符号整型数，看是否能成功解析
	if (number_type == token_type::value_unsigned)  
	{  
		// 默认解析成unsigned long long
		const auto x = std::strtoull(token_buffer.data(), &endptr, 10);  

		// we checked the number format before  
		JSON_ASSERT(endptr == token_buffer.data() + token_buffer.size());  

		if (errno == 0)  
		{  
			// 类型强转到模板参数指定的整型数后，看是否有narrow
			value_unsigned = static_cast<number_unsigned_t>(x);  
			if (value_unsigned == x)  
			{  
				return token_type::value_unsigned;  
			}  
		}  
	}  
	else if (number_type == token_type::value_integer)  
	{  
		const auto x = std::strtoll(token_buffer.data(), &endptr, 10);  

		// we checked the number format before  
		JSON_ASSERT(endptr == token_buffer.data() + token_buffer.size());  

		if (errno == 0)  
		{  
			value_integer = static_cast<number_integer_t>(x);  
			if (value_integer == x)  
			{  
				return token_type::value_integer;  
			}  
		}  
	}  

	// this code is reached if we parse a floating-point number or if an  
	// integer conversion above failed        
	// 浮点数解析，strtof根据value_float的类型不同会选择不同的重载函数
	// value_float的类型是number_float_t，亦由basic_json的模板参数指定
	// 默认是double，相当于：
	//   value_float = std::strtold(token_buffer.data(), endptr);
	strtof(value_float, token_buffer.data(), &endptr);  

	// we checked the number format before  
	JSON_ASSERT(endptr == token_buffer.data() + token_buffer.size());  

	return token_type::value_float;  
}
```

#### `parse`

回到一开始的`static parse`方法，构造好了`parser`对象后，下一步就是调用`parser::parse`方法来解析完整的json字符串了：

```cpp
template<typename InputType>  
static basic_json parse(InputType&& i,  
                        const parser_callback_t cb = nullptr,  
                        const bool allow_exceptions = true,  
                        const bool ignore_comments = false)  
{  
    basic_json result;  
    parser(detail::input_adapter(std::forward<InputType>(i)), cb, allow_exceptions, ignore_comments).parse(true, result);  
    return result;  
}

template<typename BasicJsonType, typename InputAdapterType>  
class parser {
  public:
    ......
	// result是一个OUT型参数，它就是最终生成的basic_json对象
	void parse(const bool strict, BasicJsonType& result)  
	{  
	    if (callback)  
	    {  
			......
	    }  
	    else  
	    {  
		    // 先忽略有callback的情况，直接走到这里
	        json_sax_dom_parser<BasicJsonType> sdp(result, allow_exceptions);  
	        // 下面的函数就是解析整个json的核心实现
	        sax_parse_internal(&sdp);  
	        
	        // in strict mode, input must be completely read  
	        if (strict && (get_token() != token_type::end_of_input))  
	        {  
	            sdp.parse_error(m_lexer.get_position(),  
	                            m_lexer.get_token_string(),  
	                            parse_error::create(101, m_lexer.get_position(), exception_message(token_type::end_of_input, "value"), nullptr));  
	        }  
	  
	        // in case of an error, return discarded value  
	        if (sdp.is_errored())  
	        {  
	            result = value_t::discarded;  
	            return;        
	        }  
	    }  
	  
	    result.assert_invariant();  
	}
	......
};
```

在解析过程中，这里有一个非常重要的结构：`json_sax_dom_parser`。我们需要先对该结构做一下拆解。

##### `json_sax_dom_parser`

先来看一下JSON SAX的描述：

> JSON SAX（Simple API for XML）是一种用于处理JSON数据的事件驱动的解析器API。它通过事件回调函数提供了对JSON数据的访问。这些回调函数在解析过程中被触发，以便在遇到JSON数据的不同部分时执行特定的操作。这种事件驱动的方法允许开发人员在解析过程中有选择地处理JSON数据，而不是一次性将整个JSON数据加载到内存中。
> 
> JSON SAX API通常用于以下场景：
> 	1. 处理大型JSON文件：当JSON文件非常大时，使用DOM解析器可能会导致内存不足。在这种情况下，可以使用SAX解析器逐个处理JSON数据的元素，从而减少内存使用。
> 	2. 提取特定信息：当只需要JSON数据中的某些信息时，可以使用SAX解析器仅解析所需的部分，从而提高解析速度。
> 	3. 转换数据：可以使用SAX解析器将JSON数据转换为其他格式，例如XML或CSV。

简单来说，JSON SAX是一种用于处理JSON数据的事件驱动的解析器API，它允许开发人员在解析过程中有选择地处理JSON数据，从而提高内存使用效率和解析速度。

我们来直接看看该库是怎么设计的：

```cpp
template<typename BasicJsonType>  
class json_sax_dom_parser  
{  
  public:  
    using number_integer_t = typename BasicJsonType::number_integer_t;  
    using number_unsigned_t = typename BasicJsonType::number_unsigned_t;  
    using number_float_t = typename BasicJsonType::number_float_t;  
    using string_t = typename BasicJsonType::string_t;  
    using binary_t = typename BasicJsonType::binary_t;  

	// 引用型成员root就是最终basic_json树的根
	// 构造器初始化列表中予以关联，即外部的result对象
    explicit json_sax_dom_parser(BasicJsonType& r, const bool allow_exceptions_ = true)  
        : root(r), allow_exceptions(allow_exceptions_)  
    {}  
  
    // make class move-only  
    json_sax_dom_parser(const json_sax_dom_parser&) = delete;  
    json_sax_dom_parser(json_sax_dom_parser&&) = default; 
    json_sax_dom_parser& operator=(const json_sax_dom_parser&) = delete;  
    json_sax_dom_parser& operator=(json_sax_dom_parser&&) = default;  
    ~json_sax_dom_parser() = default;  

	// 提供下面这一组SAX DOM接口，对不同的类型做不同的处理
	// 核心实现封装在了内部函数模板handle_value
    bool null()  
    {  
        handle_value(nullptr);  
        return true;    
    }  
  
    bool boolean(bool val)  
    {  
        handle_value(val);  
        return true;    
    }  
  
    bool number_integer(number_integer_t val)  
    {  
        handle_value(val);  
        return true;       
    }  
  
    bool number_unsigned(number_unsigned_t val)  
    {  
        handle_value(val);  
        return true;    
    }  
  
    bool number_float(number_float_t val, const string_t& /*unused*/)  
    {  
        handle_value(val);  
        return true;   
    }  
  
    bool string(string_t& val)  
    {  
        handle_value(val);  
        return true;       
    }  
  
    bool binary(binary_t& val)  
    {  
        handle_value(std::move(val));  
        return true;    
    }  
  
    bool start_object(std::size_t len)  
    {  
        ref_stack.push_back(handle_value(BasicJsonType::value_t::object));  
  
        if (JSON_HEDLEY_UNLIKELY(len != static_cast<std::size_t>(-1) && len > ref_stack.back()->max_size()))  
        {  
            JSON_THROW(out_of_range::create(408, concat("excessive object size: ", std::to_string(len)), ref_stack.back()));  
        }  
  
        return true;  
    }  
  
    bool key(string_t& val)  
    {  
        JSON_ASSERT(!ref_stack.empty());  
        JSON_ASSERT(ref_stack.back()->is_object());  
  
        // add null at given key and store the reference for later  
        object_element = &(ref_stack.back()->m_data.m_value.object->operator[](val));  
        return true;    }  
  
    bool end_object()  
    {  
        JSON_ASSERT(!ref_stack.empty());  
        JSON_ASSERT(ref_stack.back()->is_object());  
  
        ref_stack.back()->set_parents();  
        ref_stack.pop_back();  
        return true;    }  
  
    bool start_array(std::size_t len)  
    {  
        ref_stack.push_back(handle_value(BasicJsonType::value_t::array));  
  
        if (JSON_HEDLEY_UNLIKELY(len != static_cast<std::size_t>(-1) && len > ref_stack.back()->max_size()))  
        {  
            JSON_THROW(out_of_range::create(408, concat("excessive array size: ", std::to_string(len)), ref_stack.back()));  
        }  
  
        return true;  
    }  
  
    bool end_array()  
    {  
        JSON_ASSERT(!ref_stack.empty());  
        JSON_ASSERT(ref_stack.back()->is_array());  
  
        ref_stack.back()->set_parents();  
        ref_stack.pop_back();  
        return true;    }  
  
    template<class Exception>  
    bool parse_error(std::size_t /*unused*/, const std::string& /*unused*/,  
                     const Exception& ex)  
    {  
        errored = true;  
        static_cast<void>(ex);  
        if (allow_exceptions)  
        {  
            JSON_THROW(ex);  
        }  
        return false;  
    }  
  
    constexpr bool is_errored() const  
    {  
        return errored;  
    }  
  
  private:  
	// 核心值处理函数
    template<typename Value>  
    BasicJsonType* handle_value(Value&& v)  
    {  
	    // 如果ref stack为空，此时要处理的value就是根
        if (ref_stack.empty())  
        {  
            root = BasicJsonType(std::forward<Value>(v));  
            return &root;  
        }  
		// 否则，ref stack包含的值就是一个数组或对象
        JSON_ASSERT(ref_stack.back()->is_array() || ref_stack.back()->is_object());  
	
        if (ref_stack.back()->is_array())  
        {  
            ref_stack.back()->m_data.m_value.array->emplace_back(std::forward<Value>(v));  
            return &(ref_stack.back()->m_data.m_value.array->back());  
        }  
  
        JSON_ASSERT(ref_stack.back()->is_object());  
        JSON_ASSERT(object_element);  
        *object_element = BasicJsonType(std::forward<Value>(v));  
        return object_element;  
    }  
  
    /// the parsed JSON value  
    BasicJsonType& root;  
    /// stack to model hierarchy of values  
    // 整个parser的设计核心就是这个堆栈，它用来配合外部的解析函数去check
    // json字符串的对称性
    std::vector<BasicJsonType*> ref_stack {};  
    /// helper to hold the reference for the next object element  
    BasicJsonType* object_element = nullptr;  
    /// whether a syntax error occurred  
    bool errored = false;  
    /// whether to throw exceptions in case of errors  
    const bool allow_exceptions = true;  
};
```

##### `sex_parse_internal`

在初步了解了`json_sax_dom_parser`提供的接口能力后，我们再看核心函数`sex_parse_internal`的实现：

```cpp
// 参数sax就是上面构造好的json_sax_dom_parser<basic_json>对象
// json_sax_dom_parser只是提供SAX DOM标准接口，本身没有做解析工作
// 真正的解析工作是由parser的成员函数sax_parse_internal来完成的
template<typename SAX>   
bool sax_parse_internal(SAX* sax)  
{  
    // stack to remember the hierarchy of structured values we are parsing  
    // true = array; false = object    
    std::vector<bool> states;  
    // value to avoid a goto (see comment where set to true)  
    bool skip_to_state_evaluation = false;  
  
    while (true)  
    {  
        if (!skip_to_state_evaluation)  
        {  
            // invariant: get_token() was called before each iteration
            // 构造parser对象时会立即执行一次get_token，所以初始时last_token是有值的  
            switch (last_token)  
            {  
                case token_type::begin_object:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->start_object(static_cast<std::size_t>(-1))))  
                    {  
                        return false;  
                    }  
  
                    // closing } -> we are done  
                    if (get_token() == token_type::end_object)  
                    {  
                        if (JSON_HEDLEY_UNLIKELY(!sax->end_object()))  
                        {  
                            return false;  
                        }  
                        break;  
                    }  
  
                    // parse key  
                    if (JSON_HEDLEY_UNLIKELY(last_token != token_type::value_string))  
                    {  
                        return sax->parse_error(m_lexer.get_position(),  
                                                m_lexer.get_token_string(),  
                                                parse_error::create(101, m_lexer.get_position(), exception_message(token_type::value_string, "object key"), nullptr));  
                    }  
                    if (JSON_HEDLEY_UNLIKELY(!sax->key(m_lexer.get_string())))  
                    {  
                        return false;  
                    }  
  
                    // parse separator (:)  
                    if (JSON_HEDLEY_UNLIKELY(get_token() != token_type::name_separator))  
                    {  
                        return sax->parse_error(m_lexer.get_position(),  
                                                m_lexer.get_token_string(),  
                                                parse_error::create(101, m_lexer.get_position(), exception_message(token_type::name_separator, "object separator"), nullptr));  
                    }  
  
                    // remember we are now inside an object  
                    states.push_back(false);  
  
                    // parse values  
                    get_token();  
                    continue;                }  
  
                case token_type::begin_array:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->start_array(static_cast<std::size_t>(-1))))  
                    {  
                        return false;  
                    }  
  
                    // closing ] -> we are done  
                    if (get_token() == token_type::end_array)  
                    {  
                        if (JSON_HEDLEY_UNLIKELY(!sax->end_array()))  
                        {  
                            return false;  
                        }  
                        break;  
                    }  
  
                    // remember we are now inside an array  
                    states.push_back(true);  
  
                    // parse values (no need to call get_token)  
                    continue;  
                }  
  
                case token_type::value_float:  
                {  
                    const auto res = m_lexer.get_number_float();  
  
                    if (JSON_HEDLEY_UNLIKELY(!std::isfinite(res)))  
                    {  
                        return sax->parse_error(m_lexer.get_position(),  
                                                m_lexer.get_token_string(),  
                                                out_of_range::create(406, concat("number overflow parsing '", m_lexer.get_token_string(), '\''), nullptr));  
                    }  
  
                    if (JSON_HEDLEY_UNLIKELY(!sax->number_float(res, m_lexer.get_string())))  
                    {  
                        return false;  
                    }  
  
                    break;  
                }  
  
                case token_type::literal_false:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->boolean(false)))  
                    {  
                        return false;  
                    }  
                    break;  
                }  
  
                case token_type::literal_null:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->null()))  
                    {  
                        return false;  
                    }  
                    break;  
                }  
  
                case token_type::literal_true:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->boolean(true)))  
                    {  
                        return false;  
                    }  
                    break;  
                }  
  
                case token_type::value_integer:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->number_integer(m_lexer.get_number_integer())))  
                    {  
                        return false;  
                    }  
                    break;  
                }  
  
                case token_type::value_string:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->string(m_lexer.get_string())))  
                    {  
                        return false;  
                    }  
                    break;  
                }  
  
                case token_type::value_unsigned:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(!sax->number_unsigned(m_lexer.get_number_unsigned())))  
                    {  
                        return false;  
                    }  
                    break;  
                }  
  
                case token_type::parse_error:  
                {  
                    // using "uninitialized" to avoid "expected" message  
                    return sax->parse_error(m_lexer.get_position(),  
                                            m_lexer.get_token_string(),  
                                            parse_error::create(101, m_lexer.get_position(), exception_message(token_type::uninitialized, "value"), nullptr));  
                }  
                case token_type::end_of_input:  
                {  
                    if (JSON_HEDLEY_UNLIKELY(m_lexer.get_position().chars_read_total == 1))  
                    {  
                        return sax->parse_error(m_lexer.get_position(),  
                                                m_lexer.get_token_string(),  
                                                parse_error::create(101, m_lexer.get_position(),  
                                                        "attempting to parse an empty input; check that your input string or stream contains the expected JSON", nullptr));  
                    }  
  
                    return sax->parse_error(m_lexer.get_position(),  
                                            m_lexer.get_token_string(),  
                                            parse_error::create(101, m_lexer.get_position(), exception_message(token_type::literal_or_value, "value"), nullptr));  
                }  
                case token_type::uninitialized:  
                case token_type::end_array:  
                case token_type::end_object:  
                case token_type::name_separator:  
                case token_type::value_separator:  
                case token_type::literal_or_value:  
                default: // the last token was unexpected  
                {  
                    return sax->parse_error(m_lexer.get_position(),  
                                            m_lexer.get_token_string(),  
                                            parse_error::create(101, m_lexer.get_position(), exception_message(token_type::literal_or_value, "value"), nullptr));  
                }  
            }  
        }  
        else  
        {  
            skip_to_state_evaluation = false;  
        }  
  
        // we reached this line after we successfully parsed a value  
        if (states.empty())  
        {  
            // empty stack: we reached the end of the hierarchy: done  
            return true;  
        }  
  
        if (states.back())  // array  
        {  
            // comma -> next value  
            if (get_token() == token_type::value_separator)  
            {  
                // parse a new value  
                get_token();  
                continue;            }  
  
            // closing ]  
            if (JSON_HEDLEY_LIKELY(last_token == token_type::end_array))  
            {  
                if (JSON_HEDLEY_UNLIKELY(!sax->end_array()))  
                {  
                    return false;  
                }  
  
                // We are done with this array. Before we can parse a  
                // new value, we need to evaluate the new state first.                // By setting skip_to_state_evaluation to false, we                // are effectively jumping to the beginning of this if.                JSON_ASSERT(!states.empty());  
                states.pop_back();  
                skip_to_state_evaluation = true;  
                continue;            }  
  
            return sax->parse_error(m_lexer.get_position(),  
                                    m_lexer.get_token_string(),  
                                    parse_error::create(101, m_lexer.get_position(), exception_message(token_type::end_array, "array"), nullptr));  
        }  
  
        // states.back() is false -> object  
  
        // comma -> next value        if (get_token() == token_type::value_separator)  
        {  
            // parse key  
            if (JSON_HEDLEY_UNLIKELY(get_token() != token_type::value_string))  
            {  
                return sax->parse_error(m_lexer.get_position(),  
                                        m_lexer.get_token_string(),  
                                        parse_error::create(101, m_lexer.get_position(), exception_message(token_type::value_string, "object key"), nullptr));  
            }  
  
            if (JSON_HEDLEY_UNLIKELY(!sax->key(m_lexer.get_string())))  
            {  
                return false;  
            }  
  
            // parse separator (:)  
            if (JSON_HEDLEY_UNLIKELY(get_token() != token_type::name_separator))  
            {  
                return sax->parse_error(m_lexer.get_position(),  
                                        m_lexer.get_token_string(),  
                                        parse_error::create(101, m_lexer.get_position(), exception_message(token_type::name_separator, "object separator"), nullptr));  
            }  
  
            // parse values  
            get_token();  
            continue;        }  
  
        // closing }  
        if (JSON_HEDLEY_LIKELY(last_token == token_type::end_object))  
        {  
            if (JSON_HEDLEY_UNLIKELY(!sax->end_object()))  
            {  
                return false;  
            }  
  
            // We are done with this object. Before we can parse a  
            // new value, we need to evaluate the new state first.            // By setting skip_to_state_evaluation to false, we            // are effectively jumping to the beginning of this if.            JSON_ASSERT(!states.empty());  
            states.pop_back();  
            skip_to_state_evaluation = true;  
            continue;        }  
  
        return sax->parse_error(m_lexer.get_position(),  
                                m_lexer.get_token_string(),  
                                parse_error::create(101, m_lexer.get_position(), exception_message(token_type::end_object, "object"), nullptr));  
    }  
}
```

整个设计上是一个状态机，代码虽长，但可以分两段来看。第一段针对`skip_to_state_evaluation`的if-else处理部分，整体是对token类型按照json的设计制式来逐步处理。我们先分类讨论下几种情况：

**对象型**

```cpp
case token_type::begin_object:  
{  
	
    if (JSON_HEDLEY_UNLIKELY(!sax->start_object(static_cast<std::size_t>(-1))))  
    {  
        return false;  
    }  

	......

// --------------------------------------------------------------------
bool start_object(std::size_t len)  
{  
	// json_sax_dom_parser会使用handle_value构造一个对象型basic_json
	// 置入ref_stack并返回
	ref_stack.push_back(handle_value(BasicJsonType::value_t::object));  

    if (JSON_HEDLEY_UNLIKELY(len != static_cast<std::size_t>(-1) && len > ref_stack.back()->max_size()))  
    {  
        JSON_THROW(out_of_range::create(408, concat("excessive object size: ", std::to_string(len)), ref_stack.back()));  
    }  
  
    return true;  
}
// --------------------------------------------------------------------
    // closing } -> we are done  
    // 如果在{的后面紧跟着的是一个}，那就是个空对象
    if (get_token() == token_type::end_object)  
    {  
        if (JSON_HEDLEY_UNLIKELY(!sax->end_object()))  
        {  
            return false;  
        }  
        break;  
    }  
// --------------------------------------------------------------------
bool end_object()  
{  
	// 调用end_object必定意味着前面已经有一个start_object调用过了
	// 此时ref_stack必不能为空，且因对称性，最后一个成员必定是一个对象型
	// 否则就是json串不符合规矩
    JSON_ASSERT(!ref_stack.empty());  
    JSON_ASSERT(ref_stack.back()->is_object());  

	// 诊断debug用，记录该对象本身的父子对象关系，先忽略
    ref_stack.back()->set_parents();  
    // 此时这个对象就处理完了，该出栈了
    ref_stack.pop_back();  
    return true;
}
// --------------------------------------------------------------------
    // parse key  
    // 一个{后面紧跟着的应该是个字符串key，如果不是就说明json串有误
    if (JSON_HEDLEY_UNLIKELY(last_token != token_type::value_string))  
    {  
        return sax->parse_error(m_lexer.get_position(),  
                                m_lexer.get_token_string(),  
                                parse_error::create(101, m_lexer.get_position(), exception_message(token_type::value_string, "object key"), nullptr));  
    }  
    // 调用key接口处理key
    if (JSON_HEDLEY_UNLIKELY(!sax->key(m_lexer.get_string())))  
    {  
        return false;  
    }  
// --------------------------------------------------------------------
bool key(string_t& val)  
{  
    JSON_ASSERT(!ref_stack.empty());  
    JSON_ASSERT(ref_stack.back()->is_object());  
  
    // add null at given key and store the reference for later  
    // 将临时对象指针object_element指向obj型basic_json的value对象
    // 此处通过其operator[]寻址运算符，默认情况下m_value.object是个std::map
    object_element = &(ref_stack.back()->m_data.m_value.object->operator[](val));  
    return true;
}
// --------------------------------------------------------------------
    // parse separator (:)  
    // key之后紧跟着的得是个冒号分隔符
    if (JSON_HEDLEY_UNLIKELY(get_token() != token_type::name_separator))  
    {  
        return sax->parse_error(m_lexer.get_position(),  
                                m_lexer.get_token_string(),  
                                parse_error::create(101, m_lexer.get_position(), exception_message(token_type::name_separator, "object separator"), nullptr));  
    }  

    // remember we are now inside an object  
    // 每次处理一个对象型，就记录一个false到states
    states.push_back(false);  
	
    // parse values  
    // 处理value，value可能有复杂的嵌套
    // 所以先get_token()读入value，然后重新走switch-case状态机，所以continue
    get_token();  
    continue;
}
```

**数组型**

```cpp
case token_type::begin_array:  
{  
	// 调用start_array去记录一个数组
    if (JSON_HEDLEY_UNLIKELY(!sax->start_array(static_cast<std::size_t>(-1))))  
    {  
        return false;  
    }  
// --------------------------------------------------------------------
bool start_array(std::size_t len)  
{  
	// 生成一个array型basic_json对象，置入ref stack
    ref_stack.push_back(handle_value(BasicJsonType::value_t::array));  
  
    if (JSON_HEDLEY_UNLIKELY(len != static_cast<std::size_t>(-1) && len > ref_stack.back()->max_size()))  
    {  
        JSON_THROW(out_of_range::create(408, concat("excessive array size: ", std::to_string(len)), ref_stack.back()));  
    }  
  
    return true;  
}
// --------------------------------------------------------------------
    // closing ] -> we are done  
    // [之后可能紧跟着]，做一下处理
    if (get_token() == token_type::end_array)  
    {  
        if (JSON_HEDLEY_UNLIKELY(!sax->end_array()))  
        {  
            return false;  
        }  
        break;  
    }  
	// 记录一下，当前在处理一个array，使用true表示array
    // remember we are now inside an array  
    states.push_back(true);  
  
    // parse values (no need to call get_token)  
    // 数组是没有key的，只有value，所以和obj类似，也是通过continue
    // 回到switch-case状态机重新处理
    continue;  
}
```