---
title: 理解 Objective-C 中的指定构造方法
date: 2016-08-02 21:11:49
categories: ObjectiveC
tags: [Objective-C, Initializer]
---

所有对象在被使用前都要先进行初始化。对于最简单的情况，我们只需要使用 `init` 方法进行初始化就可以了。然而，大多数情况下，对象的初始化都需要我们提供额外的信息，并且有时创建实例的方法不止一种，这时我们就需要提供多个构造方法。

对于需要多个构造方法的情况，我们需要确保有一个`指定构造方法(designated initializer)`，指定构造方法用于为类提供必要的信息使其能完成初始化工作，所有其它的构造方法都要调用指定构造方法。这些其它的构造方法，我们通常称为“便利构造方法”，它们只能直接或者间接地调用指定构造方法，而不能直接对象进行初始化。

<!-- more -->

## 一个实例

假设我们有一个 Human 类，这个类有一个初始化方法：

```
@interface Human: NSObject
@property (strong, nonatomic) NSString *name;

- (instancetype)initWithName:(NSString *)name;
@end
```

实现：
```
@implement Human

- (instancetype)initWithName:(NSString *)name {
    if (self = [super init]) {
        _name = name;
    }
    return self;
}

@end
```

在这个类里面，`initWithName:`就是它的指定构造方法，正常情况下，我们都应该调用它来对 Human 对象进行初始化。

但是如果有人偏偏要用 `init` 来进行初始化呢？这种情况下，name 会被自动初始化成 nil，但这可能不是我们想要的结果，所以我们可以对 `init` 进行覆写，然后在覆写的方法里面调用自身的指定构造方法：

```
- (instancetype)init {
    return [self initWithName:@"Nerd"];
}
```

现在创建一个 Programmer 类并继承自 Human：

```
@interface Programmer: Human

@property (readonly) NSArray *skills;

- (instancetype)initWithName:(NSString *)name skill:(NSString *)skill;

@end

@implement Programmer {
    NSArray *_skills;
}

- (instancetype)initWithName:(NSString *)name skill:(NSString *)skill {
    if (self = [super initWithName:name]) {
        _skills = [NSMutableArray array];
        [_skills addObject:skill];
    }
    return self;
}
@end
```

在子类的构造方法里面，只需要处理子类新引用的属性，对于父类的初始化，只需要调用父类的指定构造方法就可以了。所以 *在子类的指定构造方法里面，一定要调用父类的指定构造方法*，这样才能保证所有必要的属性都被正确地初始化。

跟刚刚一样，这里也存在一个问题，我们有可能不经意间使用下面的代码对 Programmer 对象进行初始化：

```
Programmer *p = [Programmer alloc] initWithName:@"Dennis Ritchie"];
```

使用这种方式初始化出来的 Programmer 对象将对它掌握的技能(skill)一无所知。

同样的，我们可以覆写父类的指定构造方法来解决这个问题：

```
- (instancetype)initWithName:(NSString *)name {
    return [self initWithName:name skill:@"Objective-C"];
}
```

你可能会想说，那我们是不是还需要实现 `init` 方法，答案是“不需要”。

当对 Programmer 对象调用 `init` 构造方法的时候，因为它没有实现这个方法，所以会变成调用其父类，即 Human 的 `init` 方法。而 Human 的 `init` 方法里面调用的是 `[self initWithName:]`，这时的 self 是一个 Programmer 对象，所以它会调用 Programmer 的 `initWithName:` 方法，这个方法最终会调用到 Programmer 的指定构造方法，也即 `initWithName:skill:`。

也就是说，只要我们在继承链当中的每个类都实现了指定构造方法，并保证其它的构造方法都调用到了指定构造方法，就可以保证整个初始化过程的正确性。

## 规则

以上的内容可以总结成几条规则：

- 当一个类有多个构造方法的时候，要保证只有一个能真正对对象进行初始化，即指定构造方法。而其它的便利构造方法都要直接或者间接地调用指定构造方法。
- 指定构造方法需要先调用父类的指定构造方法，然后再对自身类的属性进行初始化。
- 如果子类的指定构造方法与父类不同，则该子类需要覆写父类的指定构造方法，并在该实现里面调用自身的指定构造方法。
- 如果一个类有多个构造方法，需要在头文件中指定哪个是指定构造方法。

## NS_DESIGNATED_INITIALIZER

对于上面提到的规则中的最后一条，以前都是在头文件中使用注释来说明哪个构造方法是指定构造方法。这种方法的缺点在于注释有可能过时，有时更新代码后却没有更新注释，这都有可能给后继的开发造成困扰。

自从 Xcode 6 开始，有了一个新的宏 -- `NS_DESIGNATED_INITIALIZER`。只要在头文件中使用该宏表明哪个构造方法是指定构造方法，之后如果我们在实现构造方法的时候违反了上面的规则，编译器就会给出警告。


```
// UIView
- (instancetype)initWithFrame:(CGRect)frame NS_DESIGNATED_INITIALIZER;

// NSDate
- (instancetype)initWithTimeIntervalSinceReferenceDate:(NSTimeInterval)ti NS_DESIGNATED_INITIALIZER;

// NSSet
- (instancetype)initWithObjects:(const ObjectType [])objects count:(NSUInteger)cnt NS_DESIGNATED_INITIALIZER;
```

## 结论

指定构造方法在 Objective-C 当中已经存在很久了，但是它真正开始迅速受到关注还利益于 Swift。

在 Swift 当中，所有的 init 方法都是指定构造方法，其它的便利构造方法都需要在 init 方法前加上 convince 关键字。

```
class BankAccount {
    var name: String
    var balance: Int

    convenience init(name: String) {
        self.init(name: name, balance: 0)
    }

    convenience init(balance: Int) {
        self.init(name: NSLocalizedString("Anonymous", nil), balance: balance)
    }

    init(name: String, balance: Int) {
        self.name = name
        self.balance = balance
    }
}
```

由于 Swift 中对初始化过程的检查更加严格了，一个类中的所有属性都必须被初始化，只要违反了前面小节中提到的规则中的任意一条，都会导致无法通过编译。

当我们开始用 NS_DESIGNATED_INITIALIZER 对 Objective-C 的指定构造方法进行标记，或者在 Swift 当中使用 convince 关键字的时候，都能更好地在代码中表达我们的意图，并且让编译器来帮助我们减少一些不必要的 BUG。

## 参考

- [Objective-C's Designated Secret](http://timekl.com/blog/2014/12/09/objective-cs-designated-secret/)
- [Object Initialization](https://developer.apple.com/library/ios/documentation/General/Conceptual/CocoaEncyclopedia/Initialization/Initialization.html)
