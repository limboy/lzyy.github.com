---
layout: post
title: 说说iOS的多线程Core Data
category: tech
tag: 技术
---

Core Data是iOS中很重要的一个部分，可以理解为基于SQLite(当然也可以是其他的Storage，如In-memory，只是SQLite比较常见)的一个ORM实现，所以有关系数据库的特性，又不用写SQL。顺便吐一下槽，官方说法是使用Core Data能减少50%-70%的代码量，但相信用过的人应该都心里明白，Core Data使用起来还是比较麻烦的，这也是为什么有不少的第三方类库来代替/二次包装Core Data。

稍微复杂的应用就有可能出现同时处理多份数据的情况，这就需要用到多线程Core Data。在 iOS 5之前，官方推荐的是使用「Thread Confinement」，就是每个线程使用独立的MOC(managed object context)，然后共享一个PSC(persistent store coordinator)。同时在线程之间传递数据时，要传递objectID，而不是object，因为前者是线程安全的，后者不是。

如果A线程里，对PSC执行了CUD(create, update, delete)操作，其他线程如何感知呢？这就需要通过监听事件来实现。比如在线程A中监听「NSManagedObjectContextDidSaveNotification」事件，如果线程B中执行了CUD操作，线程A就能感知到，并触发响应的action，虽然可以通过noti userinfo来获取managed objects，但因为它们是关联到另一个MOC，所以无法直接操作它们，解决方法就是调用「mergeChangesFromContextDidSaveNotification:」方法。

用一张图来形容的话，大体就是这样：

![iOS multi thread Core Data before iOS 5](/image/multi_thread_core_data.png)

{% highlight objective-c %}
- (void)_setupCoreDataStack
{
     // setup managed object model
     NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"Database" withExtension:@"momd"];
     _managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
 
     // setup persistent store coordinator
     NSURL *storeURL = [NSURL fileURLWithPath:[[NSString cachesPath] stringByAppendingPathComponent:@"Database.db"]];
 
     NSError *error = nil;
     _persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:_managedObjectModel];
 
     if (![_persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:nil error:&amp;error]) {
   	     // handle error
   }
 
     // create MOC
     _managedObjectContext = [[NSManagedObjectContext alloc] init];
     [_managedObjectContext setPersistentStoreCoordinator:_persistentStoreCoordinator];
 
     // subscribe to change notifications
     [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(_mocDidSaveNotification:) name:NSManagedObjectContextDidSaveNotification object:nil];
}
{% endhighlight %}

再来看看Notification Handler，主要作用就是合并新的变化。

{% highlight objective-c %}
- (void)_mocDidSaveNotification:(NSNotification *)notification
{
     NSManagedObjectContext *savedContext = [notification object];
 
     // ignore change notifications for the main MOC
     if (_managedObjectContext == savedContext) {
          return;
     }
 
     dispatch_sync(dispatch_get_main_queue(), ^{
      [_managedObjectContext mergeChangesFromContextDidSaveNotification:notification];
     });
}
{% endhighlight %}

这种方式实现起来和维护起来都有点麻烦，所以iOS 5中就出现了更加方便和灵活的实现，也就是「Nested MOC」。

{% highlight objective-c %}
[[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
{% endhighlight %}

可以看到在初始化时可以选择ConcurrencyType，可选的有3个：

### NSConfinementConcurrencyType

这个是默认项，每个线程一个独立的Context，主要是为了兼容之前的设计。

### NSPrivateQueueConcurrencyType

创建一个private queue(使用GCD)，这样就不会阻塞主线程。

### NSMainQueueConcurrencyType

创建一个main queue，使用主线程，会阻塞。

还有一个重要的变化是MOC可以指定parent。有了parent后，CUD操作会冒泡到parent。一个parent可以有多个child。parent还可以有parent。

因为UI相关的数据必须在主线程获取，同时又要避免数据库的I/O操作阻塞主线程，所以就有了下面这个模型：

![multi thread ios 5 nested MOC](/image/multi_thread_core_data_nested.png)

我对这种实现方式的一个困惑是：child无法得知parent的变化，也就是说，如果NSFetchedResultsController绑定了Main MOC，当Background Write MOC save时，NSFetchedResultsController为何能知晓？求指点。

这种方式比「Thread Confinement」稍微简单了点，也更明了。不过个人还是推荐使用MagicalRecord，因为实现起来更加简单，等有空再写一篇。

写了一个使用了这个模型的demo，配合TableView和NSFetchedResultsController，有兴趣的可以看下：[https://github.com/limboy/coredata-with-tableview](https://github.com/limboy/coredata-with-tableview)

### 2013/06/17更新

之前的困惑已消除，`NSFetchedResultsController`跟PSC无关，只要绑定的MOC有了`save`动作，`NSFetchedResultsController`就会收到通知，无论这个`save`操作有没有写入到持久层。

### 参考

* [http://www.cocoanetics.com/2012/07/multi-context-coredata/](http://www.cocoanetics.com/2012/07/multi-context-coredata/)
* [http://www.slideshare.net/Inferis/adventures-in-multithreaded-core-data](http://www.slideshare.net/Inferis/adventures-in-multithreaded-core-data)
