---
title: "How `Cedar` implement this syntax without punctuation?"
date: 2022-04-17T20:12:00+08:00
draft: false
---

A few days ago, I searched for a Unit Test framework for Objc&Swift. Occasionally, I found `Cedar`. I know that all UT frameworks are much the same, but `Cedar`'s syntax attracted me.

The sample code is like below.
<!--more-->
```objc
describe(@"Example specs on NSString", ^{
    it(@"lowercaseString returns a new string with everything in lower case", ^{
        [@"FOOBar" lowercaseString] should equal(@"foobar");
    });

    it(@"length returns the number of characters in the string", ^{
        [@"internationalization" length] should equal(20);
    });

    describe(@"isEqualToString:", ^{
        it(@"should return true if the strings are the same", ^{
            [@"someString" isEqualToString:@"someString"] should be_truthy;
        });

        it(@"should return false if the strings are not the same", ^{
            [@"someString" isEqualToString:@"anotherString"] should be_falsy;
        });
    });
});
```

It is simple, importantly, there is no punctuation in assert syntax.
The most interesting part is `should` and `should_not`. 

The lib is written in C++. Though I am not good at C++,  a programmer's intuition and experience help me finally figure out how it implements this syntax.

The first thing that comes to my mind is `operation overload`. And after digging into the source code confirmed this.

`should` is a macro, `Cedar` overloads the operation comma `,`.
let's see how `should` is defined.
```objc
#ifndef CEDAR_MATCHERS_DISALLOW_SHOULD
    #define should ,(ActualValueMarker){__FILE__, __LINE__},false,
    #define should_not ,(ActualValueMarker){__FILE__, __LINE__},true,
#endif
```

But there are so many commas in there to distinguish which is which. 
Let's do it step by step. Use this official sample code as an example.
```objc
#import <Cedar/Cedar.h>

using namespace Cedar::Matchers;
using namespace Cedar::Doubles;

SPEC_BEGIN(iOSFrameworkSpecs)

describe(@"Cedar-iOS", ^{
    __block NSObject *object;

    beforeEach(^{
        object = [[NSObject alloc] init];
    });

    it(@"should allow assertions", ^{
        object should_not be_nil;
        object should_not be_same_instance_as([[NSObject alloc] init]);
    });
});

SPEC_END

```

Notice the line
```objc
        object should_not be_nil; // expr 1
```

If we look up the definition of `should_not`, the sentence can be converted to
```objc
        object ,(ActualValueMarker){__FILE__, __LINE__},true, be_nil; // expr 2
```

It can be confused at the first glimpse. We must split it into several parts.
Start from the easy part `(ActualValueMarker){__FILE__, __LINE__}`. `ActualValueMarker` is a struct. Assume that the comma in `{}` is just a comma (not operator overload).
```C++
    struct ActualValueMarker {
        const char *fileName;
        int lineNumber;
    };
```

So the line above initialize a struct using the current **file** and **line** info. And other parts are `object`, `true` and `be_nil`. 

We can infer that the first comma must be overloaded such that the syntax is C++ compatible.
Three comma operator overloading functions are right below the struct definition.
```cpp
	// overload 1
    template<typename T>
    const ActualValue<T> operator,(const T & actualValue, const ActualValueMarker & marker) {
        return ActualValue<T>(marker.fileName, marker.lineNumber, actualValue);
    }

	// overload 2
    template<typename T>
    const ActualValueMatchProxy<T> operator,(const ActualValue<T> & actualValue, bool negate) {
        return negate ? actualValue.to_not : actualValue.to;
    }

	// overload 3
    template<typename T, typename MatcherType>
    void operator,(const ActualValueMatchProxy<T> & matchProxy, const MatcherType & matcher) {
        matchProxy(matcher);
    }
```

According to the function signature and real parameters, the first function is a match. `T` can be any type, the second parameter is the type of `ActualValueMarker`.  So `expr 2`
```objc
        object ,(ActualValueMarker){__FILE__, __LINE__},true, be_nil;
```
will be
```objc
        ActualValue<NSObject>(__FILE__, __LINE__, object),true, be_nil; // expr 3
```

And, obviously, that second comma of `expr 2` is also an overloading operator, cause it matches the second function's signature.
The `expr 3` can be changed to the form of 
```objc
        (ActualValue<NSObject>(__FILE__, __LINE__, object)).to_not, be_nil; // expr 4
```

`be_nil` is of type `BeNil`.
```cpp
namespace Cedar { namespace Matchers {
	using CedarBeNil = Cedar::Matchers::Private::BeNil;
	static const CedarBeNil be_nil = CedarBeNil();
}}
```

Let's dig into the `overload 3` function signature. 
`to_not` if of type `ActualValueMatchProxy<T> to_not;`, matches the first parameter's type. The `MatcherType` is a generic type of any type.
`expr 4` can be
```objc
        (ActualValue<NSObject>(__FILE__, __LINE__, object)).to_not(be_nil); // expr 5
```

When I search `to_not`, I found it's not a function at all, but it can be called like it is.
```objc
		ActualValueMatchProxy<T> to_not;
```

Actually, the magic part is `()` operator overload.
In `ActualValueMatchProxy` class public functions, there is an operator overload.
```cpp
    template<typename T> template<typename MatcherType>
    void ActualValueMatchProxy<T>::operator()(const MatcherType & matcher) const {
        if (negate_) {
            actualValue_.execute_negative_match(matcher);
        } else {
            actualValue_.execute_positive_match(matcher);
        }
    }
```

So `expr 5` -> `expr6`:
```objc
        (ActualValue<NSObject>(__FILE__, __LINE__, object)).execute_negative_match(be_nil); // expr 6
```

`execute_negative_match` implementation is like
```objc
    template<typename T> template<typename MatcherType>
    void ActualValue<T>::execute_negative_match(const MatcherType & matcher) const {
        if (matcher.matches(value_)) {
            CDR_fail(fileName_.c_str(), lineNumber_, matcher.negative_failure_message_for(value_));
        }
    }
```

The key point is `matcher.matches()` function. (`value_` is `object`). It is, in fact, `be_nil.matches()` function.
```cpp
    template<typename U>
    bool BeNil::matches(const U & actualValue) const {
        // ARC bug: http://lists.apple.com/archives/objc-language/2012/Feb/msg00078.html
        // Also blocks aren't considered a pointer type under both ARC and MRC
        if (strcmp(@encode(U), @encode(id)) == 0 || strcmp(@encode(U), @encode(void(^)(void))) == 0) {
            void *ptrOfPtr = (void *)&actualValue;
            void *ptr = *(reinterpret_cast<void **>(ptrOfPtr));
            return !ptr;
        }

        [[CDRSpecFailure specFailureWithReason:@"Attempt to compare non-pointer type to nil"] raise];
        return NO;
    }
```

Until now, the `magic` is clear totally. It is just a pointer of pointer compares nil.

# Conclusion
After digging into the source code and trying to understand the C++ code operator overload feature, I finally find out how the library implements the without-punctuation syntax.
Though the syntax is a little fancy, the complexation and extra cost of understanding it introduced are not worth it. From the point of perfect, `should_not` should be `should not`, `be_falsy` should be `be falsy`. Then we will define a new trampoline class to handle this which will make it harder to maintain and fit all the bad cases.
Operator overload is a wonderful feature but should be used in the proper situation.
