---
title: "C++20 Format Library for Custom Types"
date: 2022-02-08T21:06:32+08:00
draft: true
slug: "cpp-format-custom-types"
description: "Support the C++20 format library for your custom types"
categories:
    - Blog
    - Programing Languages
tags:
    - C++20
    - Ranges
    - Coroutines
    - Language
---

For a long time before C++20's born, first-party support for string formatting is indeed a mess for C++.  `printf` family inherited from C is known for security concerns and a lack of support for custom format strings and custom types, and the `<iostream>` library is criticized for ugly grammars and performance issues.  C++20 brings us `<format>` a better way to address string formatting.  

## `<format>` Basics

`format` library is based on the popular placeholder-based syntax for string formatting used by various languages like Python and C#. It provide a type-safe way with great extensibility.  

Function `std::format` is an important part of the library. Here is a simple example:   

```C++
cout << std::format("{1} {2} {0}", "world", "hello", 1);
```
Which gives the ouput  
> hello 1 world

Also, the function `std::format_to` used with iterators can put formatted strings into containers and streams. For example,  

``` C++
std::ofstream file{ "format.txt" };
std::format_to(std::ostream_iterator<char>(file), "hello, {}!", "world");
```

will gives a file with content "hello, world!".  
Here, we work with simple types like integers and strings. The post will discuss how to cope with custom types which is commonly seen in any C++ program.  

## Formatters

In fact, to support `std::format`, the traditional way for `<iostream>` library of defining `operator<<` is still viable, but it is less configurable and also introduce the same performance overhead of the `<iostream>` library. The newer and better way is to write a custom formatter that specialize the `std::formatter` [template](https://en.cppreference.com/w/cpp/utility/format/formatter). Formatters have been provided for built-in types and some library components,  as listed in the document in **Standard specializations for basic types and string types**:  
```C++
template<> struct formatter<char, char>;
template<> struct formatter<char, wchar_t>;
template<> struct formatter<wchar_t, wchar_t>;

template<> struct formatter<CharT*, CharT>;
template<> struct formatter<const CharT*, CharT>;
template<std::size_t N> struct formatter<const CharT[N], CharT>;
template<class Traits, class Alloc>
  struct formatter<std::basic_string<CharT, Traits, Alloc>, CharT>;
template<class Traits>
  struct formatter<std::basic_string_view<CharT, Traits>, CharT>;

template<> struct formatter<ArithmeticT, CharT>;

template<> struct formatter<std::nullptr_t, CharT>;
template<> struct formatter<void*, CharT>;
template<> struct formatter<const void*, CharT>;
```

To support one for our custom type, say the struct `product`, 

```C++
struct product
{
	string brand;
	int value;
	string name;
};
```

two functions, `parse` and `format` should be provided to support parsing the format specifications in the placeholder and to format the given value, repectively.  
Typically, `format` should be always provided, but `parse` can be simply inherited from `std::formatter` if no special format specifications is required for the custom type. Here, we support 2 format specifications, `d`,`b`, to control the detailed or breif output, respectively, for our `product` struct. So we implement the formatter as follows:

```C++
template<>
struct std::formatter<product>
{
	bool detailed{ false };

	constexpr auto parse(std::format_parse_context& context)
	{
		auto it = context.begin(), end = context.end();
		if (it != end && (*it == 'b' || *it == 'd')) detailed = (*it++) == 'd';

		if (it != end && *it != '}') throw format_error("Invalid format");

		return it;
	}

	template<class FormatContext>
	auto format(
		const product& p,
		FormatContext& context)
	{
		if (detailed)
		{
			return format_to(context.out(), "product({},{},{})", p.brand, p.name, p.value);
		}
		else
		{
			return format_to(context.out(), "product {}", p.name);
		}
	}
};
```
`parse` is resposible to parse the format specification and store the settings in data members for `format` method to use. It should return the iterator past the end of the parsed range according to the requirement. `format` should put the result in `context.out()`. So the `std::format_to` function mentioned above is used.  

With the custom formatter, we can `format` the `product` struct freely. The following code  

```C++
product p{ "kanon",5000,"d550" };

cout << std::format("{:d}", p) << endl;
cout << std::format("{:b}", p) << endl;
```

Will print  

> product(kanon,d550,5000)
> product d550

and  

```C++ 
cout << std::format("{:c}", p) << endl;
```

Will raise an exception because `c` is not a format specification that we support.  

In my work, I use `<format>` with a library [magic_enum](https://github.com/Neargye/magic_enum) to support a elegant way for logging enums.   

## `<format>` in the Future

As is said in Microsoft's blog [<format> in Visual Studio 2019 version 16.10](https://devblogs.microsoft.com/cppblog/format-in-visual-studio-2019-version-16-10/), C++23 will likely add compile time format checking to format literals, which will make the new library even faster and avoid runtime exceptions. Also, `std::print` may come to the library to directly print the format result.  


