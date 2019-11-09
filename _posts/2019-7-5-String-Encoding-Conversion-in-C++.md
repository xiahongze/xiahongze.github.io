---
layout: post
title:  String Encoding and Decoding in C++
date:   2019-07-05 12:00:00 +1100
categories: tech c++
comments: true
---

This is an old note written awhile back this year. Hope it would be helpful to people
who are fiddling with C++ encoding.

## String encode and decode in C++

Two options:

- use libiconv, directly work with standard char array (in both C/C++)
- use \<codecvt\> if you are working with C++11 and above

### libiconv

```cpp
#include <iconv.h>
#include <cstring>
#include <iomanip>
#include <iostream>
#include <string>
using namespace std;

int codeConvert(char *from_charset, char *to_charset, char *inbuf, size_t inlen, char *outbuf, size_t outlen) {
    char **pin = &inbuf;
    char **pout = &outbuf;
    iconv_t cd = iconv_open(to_charset, from_charset);  // conversion descripto
    if (cd == 0) {                                      // if no such cd exists
        cout << "no set conversion from " << from_charset << " to " << to_charset << endl;
        return -1;
    }
    memset(outbuf, 0, outlen);
    size_t errnum = iconv(cd, pin, &inlen, pout, &outlen);
    iconv_close(cd);
    return errnum;
}

string codeConvertString(const string &inStr, const string &fromCode, const string &toCode) {
    cout << "converting `" << inStr << "` from " << fromCode << " to " << toCode << endl;
    char *inbuf = (char *)inStr.c_str();
    int inlen = strlen(inbuf);
    int outlen = 3 * inlen;  // in case unicode 3 times than ascii, could be other bigger numbers
    char outbuf[outlen];
    int err = codeConvert((char *)fromCode.c_str(), (char *)toCode.c_str(), inbuf, inlen, outbuf, outlen);
    if (err == 0) {
        return string(outbuf);
    } else {
        cout << "failed to convert with error code " << err << endl;
        return inStr;
    }
}

void displayHexString(const string &string) {
    for (auto &&c : string) {
        cout << hex << showbase << setw(2) << nouppercase << (unsigned int)(unsigned char)c << ' ';
    }
    cout << '\n';
}

int main() {
    string s("ÐÂ²»ÁËÇé");
    displayHexString(s);
    string ss = codeConvertString(s, "utf-8", "latin1");
    displayHexString(ss);
    string sss = codeConvertString(ss, "gbk", "utf-8");
    displayHexString(sss);
    cout << sss << endl;
}
```

> g++ -std=c++11 test.cpp -o test -liconv
> ./test

    0xc3 0x90 0xc3 0x82 0xc2 0xb2 0xc2 0xbb 0xc3 0x81 0xc3 0x8b 0xc3 0x87 0xc3 0xa9 
    converting: `ÐÂ²»ÁËÇé` from utf-8 to latin1
    0xd0 0xc2 0xb2 0xbb 0xc1 0xcb 0xc7 0xe9 
    converting: `�²�����` from gbk to utf-8
    0xe6 0x96 0xb0 0xe4 0xb8 0x8d 0xe4 0xba 0x86 0xe6 0x83 0x85 
    新不了情


### codecvt

With *c++1x*, a code converting library was implemented in the standard [localization library](https://en.cppreference.com/w/cpp/locale). 
However, its usage is pretty daunting as one has to deal with different string types and char types.

Chars:

Apart from the standard **C** char, there are four more types of chars in c++, namely char8_t, char16_t, char32_t, and wchar_t.

Strings:

- std:string: your normal/daily string if you are using UNIX. Think about it as a wrapper class over the classic char/byte array. basic_string<char>.
- std:wstring: wide encoding string. basic_string<wchar_t>. Maybe preferred in Windows.
- of course you can define string types for the other types. Or you could just use a container to store your list/array of xchars.

Concepts:

- locale: human conventions of writing of characters, times and monetary symbols, etc.
- facet: c++ class, conversion rules given a locale and source and destination character types.
- mbstate_t: a trivial non-array type that can represent any of the conversion states that can occur in an implementation-defined set of supported multibyte character encoding rules.

Using a defined facet, one could convert two char types given a locale setting.
With std::codecvt, one can choose from the [six defined conversions](https://en.cppreference.com/w/cpp/locale/codecvt).
We will only cover the conversion between wchar_t and char, thus string and wstring in this article.

```cpp
#include <codecvt>
#include <iomanip>
#include <iostream>
using namespace std;

wstring decodingStringToWstring(const string& external, const string& localeName) {
    locale loc(localeName);
    auto& f = use_facet<codecvt<wchar_t, char, mbstate_t>>(loc);
    mbstate_t mb = mbstate_t();  // initial shift state
    wstring internal(external.size(), '\0');
    const char* from_next;
    wchar_t* to_next;
    auto result = f.in(mb, &external[0], &external[external.size()], from_next, &internal[0],
                       &internal[internal.size()], to_next);
    switch (result) {
        case codecvt_base::result::ok:
            internal.resize(to_next - &internal[0]);
            wcout << L"The string in wide encoding: " << internal << endl;
            return internal;
        case codecvt_base::result::partial:
            cout << "partial conversion\n";
            return L"";
        case codecvt_base::result::error:
            cout << "conversion error\n";
            return L"";
        case codecvt_base::result::noconv:
            cout << "no conversion done\n";
            return L"";
    }
}

string encodingWstringToString(const wstring& internal, const string& localeName) {
    locale loc(localeName);
    auto& f = std::use_facet<std::codecvt<wchar_t, char, std::mbstate_t>>(loc);
    // note that the following can be done with wstring_convert is deprecated in c++17
    std::mbstate_t mb{};  // initial shift state
    std::string external(internal.size() * f.max_length(), '\0');
    const wchar_t* from_next;
    char* to_next;
    auto result = f.out(mb, &internal[0], &internal[internal.size()], from_next, &external[0],
                        &external[external.size()], to_next);
    switch (result) {
        case codecvt_base::result::ok:
            external.resize(to_next - &external[0]);
            std::cout << "The string in narrow multibyte encoding: " << external << endl;
            return external;
        case codecvt_base::result::partial:
            cout << "partial conversion\n";
            return "";
        case codecvt_base::result::error:
            cout << "conversion error\n";
            return "";
        case codecvt_base::result::noconv:
            cout << "no conversion done\n";
            return "";
    }
}

void displayHexString(const string& string) {
    for (auto&& c : string) {
        cout << hex << showbase << setw(2) << (int)c << ' ';
    }
    cout << '\n';
}

void displayHexWString(const wstring& string) {
    for (auto&& c : string) {
        cout << hex << showbase << setw(2) << (int)c << ' ';
    }
    cout << "size=" << string.size() << '\n';
}

int main() {
    string fromLocale("zh_CN.GBK"), toLocale("en_US.UTF-8");
    string external("\xCC\xCC");  // "烫"的GBK码
    cout << "display gbk str: " << external << endl;
    displayHexString(external);
    wstring internal = decodingStringToWstring(external, fromLocale);
    wcout << L"display gbk wstr: " << internal << endl;
    displayHexWString(internal);
    external = encodingWstringToString(internal, toLocale);
    cout << "display utf8 str: " << external << endl;
    displayHexString(external);
}
```

> g++ -std=c++11 test.cpp -o test
> ./test

    display gbk str: ??
    0xcc 0xcc 
    The string in wide encoding: ?
    display gbk wstr: ?
    0x70eb size=0x1
    The string in narrow multibyte encoding: 烫
    display utf8 str: 烫
    0xe7 0x83 0xab 

## Caveats

- Somehow codecvt did not work properly in macOS (clang-1001.0.46.4). Good thing is that libiconv works.
- Linux works. Tested with g++ (Raspbian 6.3.0-18+rpi1+deb9u1) 6.3.0 20170516
- Make sure you know your available locales from `locale -a`. If you misspell the locale name, you get an error. Uncomment the locale your need from `/etc/locale.gen` and generate it with `locale-gen`.
- std::use_facet returns a reference to the facet. The reference returned by this function is valid as long as any std::locale object exists that implements Facet.