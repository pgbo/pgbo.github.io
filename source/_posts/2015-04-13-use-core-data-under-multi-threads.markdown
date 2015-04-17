---
layout: post
title: 在多线程下使用Core Data
date: 2015-04-13 00:05:09 +0800
comments: true
categories: [iOS, Core Data, 多线程, 并发]
---
###背景
在开发iOS应用过程中，不免会和数据的操作打交道，为了方便和快速，你可以选择基于sqlite的第三方库如`FMDB`。当然你也可能像我一样，花点时间了解一下Core Data，自己动手解决数据库操作的问题，其中一个稍微复杂的问题就是多线程下数据操作的问题。

话说事情的发展是这样的，起初你写了一个聊天应用，用到了Core Data来存取聊天数据，那时还很简单，数据的存取操作都基于一个application级别的唯一的`NSManagedObjectContext`，这时候一切都还很美好，没发现什么问题；可是随着业务的增多，你的应用开始发现越来越多的地方要对某类数据进行存取，聊天模块要查询聊天信息列表，同时在垃圾过滤模块要删除垃圾信息，更惊恐的是，同时在新消息提醒模块还在连续地插入新聊天信息，这时候你的应用在听到第一声新消息提醒声的时候却奔溃了。怎么了呢？你陷入深深的思考之中...
###哪里出问题了？
你的数据存取都基于一个`NSManagedObjectContext`，你并没有对它进行多线程环境的改装。
###要怎么做？
简单一句话，就是在多线程下正确地使用Core Data。
###实现方法
- **Thread Confinement** 

在 iOS 5之前，官方推荐的是使用Thread Confinement，也就是传统的多线程`NSManagedObjectContext`。每个线程使用独立的`MOC(managed object context)`，然后共享一个`PSC(persistent store coordinator)`。同时在线程之间传递数据时，要传递线程安全的`objectID`，而不是不安全的`NSMangedObject`。线程之间通过监听`NSManagedObjectContextDidSaveNotification`通知然后调用`mergeChangesFromContextDidSaveNotification:`方法来保持数据的一致性。一张图解释这种方式：

![Thread Confinement](/images/core_data/Thread confinement.png "Thread Confinement")

- **Queue-based concurrency**

iOS5之后为了更方便地处理Core Data多线程下并发问题，通过构造方法`initWithConcurrencyType:`可以构造3种类型的MOC:
1.	`Confinement (NSConfinementConcurrencyType)`. 默认类型，主要是为了兼容iOS5之前的`Confinement`方式，这也是iOS5之后不推荐的方法。官方说法是，只有在`MOC`的`parent store`是一个` persistent store coordinator `时才可以使用这种类型的`MOC`。
2.	`Private queue (NSPrivateQueueConcurrencyType)`. 该类型的`MOC`创建和管理一个`private queue`，通过这种类型的`MOC`操作数据，就能实现数据的多线程操作。
3.	`Main queue (NSMainQueueConcurrencyType)`. 该类型的`MOC`和主线程关联了，它和`NSPrivateQueueConcurrencyType`的`MOC`类似，只是关联的queue变成了主线程，它设立的目的也是为了`MOC`能和界面打交道，比如`NSFetchedResultsViewController`.

`MOC`可以指定`parent`，`MOC`的`CUD`操作在`save`之后就会冒泡到`parent`。一张图解释`Parent/Child Context`模型:

![Parent/child Context](/images/core_data/parent-child.png "Parent/child Context")


###那么问题来了，如何保证线程安全？
引入了多线程，就要面对这个头痛的问题。`queue-based concurrency`的`MOC`通过使用`performBlock:`和`performBlockAndWait:`来保证在他们的代码块里的`NSManagedObject`线程安全。为了更好的让大家了解如何安全的使用Core Data，下面举一些例子。

+ 用例1，插入一条数据：
``` objective-c
// 在某上下文中

// 选择正确的MOC
NSManagedObjectContext *moc = nil;
if ([ThreadUtil currentInMainThread]) {
moc = mainMOC;
} else {
moc = privateMOC;
}

// NSManagedObject错误的创建位置, 这里创建的NSManagedObject不安全
// NSManagedObject *moj = [NSMangedObject createXXXX];

[moc performBlockAndWait:^{
// NSManagedObject正确的创建位置
NSManagedObject *moj = [NSMangedObject createXXXX];
[privateMOC save:&error];
}];
```

+ 用例2，查询某数据列表：
``` objective-c
// 在某上下文中

NSFetchRequest *req = [NSFetchRequest createRequestXXX];
NSArray *results = nil;
// 选择正确的MOC
if ([ThreadUtil currentInMainThread]) {
[mainMOC performBlockAndWait:^{
results = [mainMOC excuteXXX];
}];          
} else {
[privateMOC performBlockAndWait:^{
results = [privateMOC excuteXXX]; 
}];
}
```

用例3，删除某些数据：
``` objective-c
// 在某上下文中

// 选择正确的MOC
NSManagedObjectContext *moc = nil;
if ([ThreadUtil currentInMainThread]) {
moc = mainMOC;
} else {
moc = privateMOC;
}

// 先查询出数据对应数据
NSFetchRequest *req = [NSFetchRequest createRequestXXX];
NSArray *results = nil;
[moc performBlockAndWait:^{
results = [privateMOC excuteXXX]; 
// 在mod的block里执行删除，不要跑到block外去删除，否则会造成NSManagedObject不安全
for (NSManagedObject *moj in results) {
[privateMOC deleteObject:moj];
}
}];
// 删除NSManagedObject错误的位置，在这里NSMangedObject会不安全
//for (NSManagedObject *moj in results) {
//    [moj.managedContext deleteObject:moj];
//}
```

用例4，更新某条数据：
``` objective-c
// 上下文
// 先根据条件查询相应数据
NSFetchRequest *req = [NSFetchRequest createRequestXXX];
NSArray *results = nil;
// 选择正确的MOC
NSManagedObjectContext *moc = nil;
if ([ThreadUtil currentInMainThread]) {
moc = mainMOC;
} else {
moc = privateMOC;
}
[moc performBlockAndWait:^{
results = [moc excuteXXX];
// 正确地更新位置：在block里执行更新操作 
for (NSManagedObject *moj in results) {
moj.name = @“小明”;
moj.age = 22; ……
// 提交会将变动冒泡到moc的parentContext
[moc save:&err];
}
}];

// 错误地更新位置，不在block里执行会引起NSManagedObject的不安全 
//for (NSManagedObject *moj in results) {
//     moj.name = @“小明”;
//     moj.age = 22; ……
//     [moc save:&err];
//}
```

###有没有形成可复用的框架？

基于`queue-based concurrency`方式，有人提出`Asyschronized saving`模型，其精髓是把`Main MOC`的`parent`设置为一个`private MOC`, 然后把`persistent store coordinator`设置给这个最顶层的`private MOC`，在这个模型中，底层的`MOC`只要`save`，就会向它的`parent MOC`提交变化，比如界面上的创建和修改数据操作可以异步提交给`main moc`的`parent moc`, 该`moc`可以进行异步保存，__所以界面不会卡顿__。我自己基于这个`Asyschronized saving`模型，写了一些可复用的代码，能有效预防多线程下Core Data使用不安全的问题，同时在并发下对于数据的增删改查有不错的效果，具体可以在github上看代码和介绍文档[LvMultiThreadCoreDataDao](https://github.com/pgbo/LvMultiThreadCoreDataDao "LvMultiThreadCoreDataDao")
