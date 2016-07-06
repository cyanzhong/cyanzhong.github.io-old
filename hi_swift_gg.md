# 通过 Swift 在 HealthKit 中使用睡眠状况数据

原作者：[Anushk Mittal](http://www.appcoda.com/author/anushkmittal/) 译者：[钟颖](https://github.com/cyanzhong)  2016年7月6日

------

目前，睡眠的革命是一个非常流行的话题，比起了解自己睡觉的时间，用户更好奇搜集睡眠数据以得到睡眠的趋势。新的硬件尤其是移动端技术的崛起，给这个不断火热的话题带来了新的希望。

iOS 内置了一个健康应用，可以安全的存储和传输健康数据，这是一个非常酷的方式。你不仅可以用 HealthKit 来[构建一个健身类的应用](https://www.appcoda.com/healthkit-introduction/)，还能通过他访问睡眠数据。

本教程将会快速的介绍 HealthKit 这个框架，并且展示如何通过他来构建一个睡眠辅助应用。

# 介绍

HealthKit 框架提供了一个叫做 `HealthKit Store` 的结构，用于把数据存储到一个加密的数据库里面。iPhone 和 Apple Watch 都有他们自己的 HealthKit Store，健康数据在 Apple Watch 和 iPhone 两端是被同步的。出于节省空间的考虑，老的数据将会从 Apple Watch 上面周期性的被移除掉。HealthKit 和健康 app 在 iPad 上面是无法使用的。

如果你需要通过健康数据构建一个 iOS/watchOS 应用的话，HealthKit 是一个强大的工具。HealthKit 可以管理各种健康数据源，并且根据用户设置自动的聚合。当然应用也可以访问各个来源的原始数据，然后进行他们自己的聚合策略。数据源不仅包括身体测量、营养摄入等等，当然也有睡眠状况数据。

在接下来的内容里面，我将会展示如何通过 HealthKit 框架在 iOS 上面进行睡眠数据的保存和读取。同样的方法可以直接用在 watchOS 应用上面。请注意本教程所示例子使用 Swift 2.0 和 Xcode 7，所以请保证你在使用 Xcode 7（或者以上的版本）。

在开始之前，你可以在这里下载并解压原始项目。我已经构建好了基本的界面，运行项目之后你将会看到一个开始按钮，点击后将会有一个倒计时的界面。

# 使用 HealthKit 框架

我们这个应用的目标是通过 `Start` 和 `Stop` 按钮来存储和读取睡眠状况数据。为了使用 HealthKit，你需要在项目的 Capabilities 选项里面打开 HealthKit 选项。导航到你当前的 target->Capabilities，然后打开 HealthKit 选项。

![image](http://www.appcoda.com/wp-content/uploads/2016/05/HealthKit-allow-1240x775.png)

然后，在你的 `ViewController` 里面通过下面的代码新建一个 `HKHealthStore` 的实例：

```swfit
let healthStore = HKHealthStore()
```

在后面我们将使用这个实例来访问 HealthKit store。

需要注意的是，HealthKit 赋予了用户对健康数据权限控制的权利。所以在读写睡眠数据之前，你必须先向用户请求权限。
要做到这一点，首先要引入 `HealthKit` 框架，然后在 `ViewController` 里面更新你的 viewDidLoad 方法：

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    let typestoRead = Set([
        HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis)!
        ])
    
    let typestoShare = Set([
        HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis)!
        ])
    
    self.healthStore.requestAuthorizationToShareTypes(typestoShare, readTypes: typestoRead) { (success, error) -> Void in
        if success == false {
            NSLog(" Display not allowed")
        }
    }
}
```

这些代码会像用户展示一个请求权限的窗口，用户可以决定允许或者拒绝该权限。你能在方法的回调里面获得最终的结果，在这里处理成功或者失败的情况。用户不一定会允许全部的权限，你必须在应用里面优雅的处理错误情况，给予用户友好的交互。

但是为了测试的方便，你必须勾选 “允许” 选项，这样我们才能正常的访问到健康数据。

![image](http://www.appcoda.com/wp-content/uploads/2016/05/Health-App-Permission.png)

# 存储睡眠状况数据

首先，我们如何得到睡眠状况数据呢？通过苹果的文档我们可以知道，每个睡眠状况数据的样本只含有一个值，代表用户在床上和睡着了，HealKit 使用两个或多个时间重叠的样本。通过对比样本的开始时间和结束时间，我们可以计算一系列的用户行为：

- 用户入睡需要的时间
- 用户睡着时间占在床上时间的百分比
- 用户在床上起来的次数
- 在床上和入睡的总时间

![image](http://www.appcoda.com/wp-content/uploads/2016/05/record_sleep_data.png)

简而言之，你可以通过下面的方法来存储睡眠状况数据到 HealthKit store：

##### 1. 定义两个 `NSDate` 对象用来表示开始和结束时间
##### 2. 通过 `HKCategoryTypeIdentifierSleepAnalysis` 实例化一个 `HKObjectType`
##### 3. 实例化 `HKCategorySample`，通常使用这个对象来记录数据。每个独立的样本代表用户在床上或者睡着的时间周期。所以我们构建时间重叠的两个样本，一个代表在床上，一个代表已经睡着
##### 4. 最后，我们通过 `HKHealthStore` 的 `saveObject` 来存储数据

> 编辑注：你可以在这里找到关于数据类型的介绍：[HealthKit Constants Reference](https://developer.apple.com/library/ios/documentation/HealthKit/Reference/HealthKit_Constants/index.html#//apple_ref/doc/uid/TP40014710)

如果用 Swift 来实现上面的需求的话，这个代码片段可以用来存储 In Bed 和 asleep 类型的数据。请在你的 `ViewController` 里面插入这些代码：

```swift
func saveSleepAnalysis() {
    
    // alarmTime and endTime are NSDate objects
    if let sleepType = HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis) {
        
        // we create our new object we want to push in Health app
        let object = HKCategorySample(type:sleepType, value: HKCategoryValueSleepAnalysis.InBed.rawValue, startDate: self.alarmTime, endDate: self.endTime)
        
        // at the end, we save it
        healthStore.saveObject(object, withCompletion: { (success, error) -> Void in
            
            if error != nil {
                // something happened
                return
            }
            
            if success {
                print("My new data was saved in HealthKit")
                
            } else {
                // something happened again
            }
            
        })
        
        
        let object2 = HKCategorySample(type:sleepType, value: HKCategoryValueSleepAnalysis.Asleep.rawValue, startDate: self.alarmTime, endDate: self.endTime)
        
        healthStore.saveObject(object2, withCompletion: { (success, error) -> Void in
            if error != nil {
                // something happened
                return
            }
            
            if success {
                print("My new data (2) was saved in HealthKit")
            } else {
                // something happened again
            }
            
        })
        
    }
    
}
```

在我们想要把数据存储到 HealthKit 里面的时候，可以调用这个方法。

# 读取睡眠状况数据

为了读取睡眠状况数据，我们需要新建一个查询对象。通过 `HKCategoryTypeIdentifierSleepAnalysis` 实例化一个 `HKObjectType`，通过 predicate 可以对读取出来的数据用 `startDate`（起始时间）和 `endDate`（结束时间）过滤，同样你也需要通过 `sortDescriptor` 来对查询结果进行排序。

读取数据的代码看起来像是这个样子：

```swift
func retrieveSleepAnalysis() {
    
    // first, we define the object type we want
    if let sleepType = HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis) {
        
        // Use a sortDescriptor to get the recent data first
        let sortDescriptor = NSSortDescriptor(key: HKSampleSortIdentifierEndDate, ascending: false)
        
        // we create our query with a block completion to execute
        let query = HKSampleQuery(sampleType: sleepType, predicate: nil, limit: 30, sortDescriptors: [sortDescriptor]) { (query, tmpResult, error) -> Void in
            
            if error != nil {
                
                // something happened
                return
                
            }
            
            if let result = tmpResult {
                
                // do something with my data
                for item in result {
                    if let sample = item as? HKCategorySample {
                        let value = (sample.value == HKCategoryValueSleepAnalysis.InBed.rawValue) ? "InBed" : "Asleep"
                        print("Healthkit sleep: \(sample.startDate) \(sample.endDate) - value: \(value)")
                    }
                }
            }
        }
        
        // finally, we execute our query
        healthStore.executeQuery(query)
    }
}
```

这些代码会在 HealthKit 里面查询所有的睡眠状况数据并且将其按照降序排列。对于每个查询，会打印出该样本的起始时间、结束时间和类型，例如 In Bed 或者 Asleep。在这里我只读取最近的 30 条数据，你也可以通过 predicate 来配置起始和结束时间。

# App 测试

为了做演示，我在应用里设置了一个 NSTimer，在你点击 `start` 按钮之后，会展示时间的流逝。在点击开始和接受之后记录 `NSDate`，用于表示经过的时间。在 stop 方法里面，你可以通过调用 `saveSleepAnalysis()` 和 `retrieveSleepAnalysis()` 来进行数据的存取。

在你的应用里面，你也许需要将 NSDate 对象换成你真实的起始和结束时间（也许是不同的）来存储 In Bed 和 Asleep 数据。

当你做好这些改动之后，你可以将 Demo 应用跑起来并启动计时器，过几分钟之后点击结束让他停下来。然后打开健康应用，你将在里面看到睡眠数据。

![image](http://www.appcoda.com/wp-content/uploads/2016/06/sleep-analysis-test-1240x878.png)

# 给 HealthKit 应用的一些建议

HealthKit 被设计成用来提供一个公共的平台给应用开发者，开发者可以轻松的分享和读取用户的数据，并且能避免任何可能的数据冗余或不一致。苹果审核指南对 HealthKit 应用十分严格，使用健康数据并进行存取需要明确的指出用途，否则将会遭到苹果审核部门的拒绝。

应用如何存储假的或者不正确的健康数据到健康应用里面，也会遭到审核的拒绝。这意味着你不能像本教程里面那样，随意的实现你的健康数据计算方法。你应该尝试使用内置的传感器数据读取和处理任何参数，以避免计算出错误的数据。

你可以在[这里](https://github.com/appcoda/SleepAnalysis)找到完整的 Xcode 项目。