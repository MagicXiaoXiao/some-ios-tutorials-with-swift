#Geofencing Tutorial with Core Location
##Core Location的地理围栏开发指南

***

>* 原文链接 : [Geofencing Tutorial with Core Location](https://www.raywenderlich.com/136165/core-location-geofencing-tutorial)
* 原文作者 : [Andy Pereira](https://www.raywenderlich.com/u/macandyp)
* 译者 : [yrq110](https://github.com/yrq110)

***

>更新: 这篇教程已更新为 Xcode 8 / Swift 3。

![](https://cdn2.raywenderlich.com/wp-content/uploads/2016/09/Geofencing-feature-250x250.png)

来创建围栏吧!

当设备进入或离开你所设置的区域时，地理围栏会给app发送一条通知提醒你。这可以让你做一些很cool的事：比如设置一个当你离开家时的通知，或者当附近有用户喜爱的商店时用最新与最棒的商品来迎接用户。在这篇地理围栏的教程中，会学到如何在iOS中使用Swift进行区域监测 - 使用Core Location中的Region Monitoring API。

在这篇教程中会创建一个名为Geotify的位置提醒应用，让用户创建提示器并与真实世界的实际位置相关联。开始吧!

## 入门

下载[开始工程](https://koenig-media.raywenderlich.com/uploads/2016/09/Geotify-Starter-1.zip)。这个工程提供一个简单的功能：允许用户在地图上添加或删除标注点，每个标注点都表示一个位置提醒，或者可以叫它——地理通知(PS:geotification，自造词)。

构建并运行，会看到如下这样一个空的map view。

![](https://cdn5.raywenderlich.com/wp-content/uploads/2016/06/GeoInitial-281x500.png)

点击导航栏的+按钮添加一个新的地理通知。app会出现一个独立的视图来设置地理通知的各种属性。

在这篇教程中需要在库比蒂诺的苹果总部添加一个大头针，如何你不知道在哪，打开google地图去找到这个点，记得活用缩放功能提高准确度！

> 注意：在模拟器中使用捏合手势进行缩放，按住option键的同时按下shift移动捏合中心，然后松开shift单击拖动进行捏合操作。

![](https://cdn4.raywenderlich.com/wp-content/uploads/2016/06/GeoLooking-Around-281x500.png)

radius表示与指定位置间的距离，超过这个距离会触发iOS的通知，note表示显示在通知中的信息。app也允许用户使用顶部的分段控件来设置通知的触发时机是进入还是离开圆形的地理围栏时。

在radius中输入1000，note中输入Say Hi to Tim!顶部选择Upon Entry，这样，第一次地理通知就设置好了。

点击Add添加，会看到在map view中出现了一个新标记的大头针，就是你刚刚添加的地理通知，标记点附近有一个圆圈表示定义的地理围栏:

![](https://cdn5.raywenderlich.com/wp-content/uploads/2016/06/Geo-Say-Hi-281x500.png)

点击大头针会显示地理通知的详细信息，比如提醒的内容与之前指定的事件类型。别点那个小叉号除非你想删掉它!

可以随意添加或删除任何地理通知。由于app使用NSUserDefaults存储持久化数据，因此重启app时会保留之前的地理通知列表。

## 设置LocationManager与许可

至此，你添加在map view上的任何地理通知都仅仅是显示了出来，并没有实际的检测效果。为了实现围栏检测的功能，需要在Core Location上注册每个与地理通知关联的围栏。

在启用围栏监测前，需要设置一个LocationManager实例并且请求到一些许可。

打开GeotificationsViewController.swift在类顶部声明一个CLLocationManager的常量实例，如下:

```swift
class GeotificationsViewController: UIViewController {
 
  @IBOutlet weak var mapView: MKMapView!
 
  var geotifications = [Geotification]()
  let locationManager = CLLocationManager() // Add this statement
 
  ...
}
```

接着使用如下代码替换viewDidLoad()方法:

```swift
override func viewDidLoad() {
  super.viewDidLoad()
  // 1
  locationManager.delegate = self
  // 2
  locationManager.requestAlwaysAuthorization()
  // 3
  loadAllGeotifications()
}
```

来一步步实现这个方法:

1. 将locationManager实例的委托设为当前视图控制器。
2. 调用requestAlwaysAuthorization()方法，提示用户程序请求一个使用地理位置服务的授权。app的地理围栏功能需要Always authorization(前后台都能使用的授权), 这是因为围栏检测在app未在前台运行时也要进行检测。Info.plist中的NSLocationAlwaysUsageDescription键已经设置好了当请求用户位置时所显示的信息。
3. 调用loadAllGeotifications(),反序列化之前在NSUserDefaults中保存的地理通知列表，并将其保存在本地的地理通知数组中。这个方法也会加载map view上标注的地理通知。

当app提示用户授权时，会显示NSLocationAlwaysUsageDescription中的值，友好的说明为何app需要访问用户的地理位置。这个键是请求位置服务授权时所必须要设置的，若没有这个键，则系统会忽略请求并拒绝提供位置服务。

构建并运行，会看到一个用户提示，显示的信息为之前所设置的:

![](https://cdn4.raywenderlich.com/wp-content/uploads/2016/06/GeoLocationWhenNotUsing-281x500.png)

已完成了设置app请求许可的部分，干得好！点击Allow确保location manager在适当的时候能接收到委托方法的回调。

在实现地理围栏之前，还有一个小问题需要解决：当前的用户位置还未显示在map view上！这个特性还未启用，导致导航栏左上的缩放按钮不起作用。

还好这个问题不是什么大事，需要在app获得授权后启用map view显示当前位置的功能。

进入GeotificationsViewController.swift文件，在CLLocationManagerDelegate扩展中添加如下委托方法，:

```swift
extension GeotificationsViewController: CLLocationManagerDelegate {
  func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
    mapView.showsUserLocation = (status == .authorizedAlways)
  }
}
```
当授权状态改变时，location manager就会调用locationManager(\_:didChangeAuthorizationStatus:)方法。若用户已经给予了app使用位置服务的许可，则这个方法会在初始化location manager并设置它的委托后被调用。

这个方法放在了合适的地方，用来检测app是否被授权，若被授权，则在map view中显示用户的当前位置。

构建并运行app，若是在真机上运行的，则会看到主map view中会出现一个位置标记。若是在模拟器上运行的，点击菜单中的Debug\Location\Apple来显示位置标记:

![](https://cdn3.raywenderlich.com/wp-content/uploads/2016/06/GeoLocationFar-281x500.png)

另外，导航栏上的缩放按钮现在起作用了。:]

![](https://cdn3.raywenderlich.com/wp-content/uploads/2016/06/GeoLocationZoomed-281x500.png)

## 注册地理围栏

在正常设置完location manager后，接下来需要做的工作是在app中注册用户的地理围栏。

在app中，用户的地理围栏信息是使用自定义的地理通知模型存放的。不过，Core Location要求每个地理围栏在注册前都必须是一个CLCircularRegion实例。为了解决这个问题，需要创建一个帮助方法来将地理通知对象转换为CLCircularRegion。

打开GeotificationsViewController.swift文件，添加如下方法:
```swift
func region(withGeotification geotification: Geotification) -> CLCircularRegion {
  // 1
  let region = CLCircularRegion(center: geotification.coordinate, radius: geotification.radius, identifier: geotification.identifier)
  // 2
  region.notifyOnEntry = (geotification.eventType == .onEntry)
  region.notifyOnExit = !region.notifyOnEntry
  return region
}
```
上面的代码中做了哪些事情:

1. 首先使用地理围栏的位置、半径与一个识别器(为了使iOS区分app中注册的围栏)初始化了一个CLCircularRegion。初始化是很简单的，Geotification模型已经包含了所需要的属性。
2. CLCircularRegion实例有2个布尔型属性：notifyOnEntry和notifyOnExit。这些标识指定了当设备进入或离开指定围栏时是否触发围栏事件。若app设计为对每个围栏仅允许一个通知，根据Geotification对象中保存的枚举值，将其中一个标识设为true另一个设为false即可。

接着，需要一个当用户添加地理通知时启动监测的方法。

在GeotificationsViewController中添加如下方法:
```swift
func startMonitoring(geotification: Geotification) {
  // 1
  if !CLLocationManager.isMonitoringAvailable(for: CLCircularRegion.self) {
    showAlert(withTitle:"Error", message: "Geofencing is not supported on this device!")
    return
  }
  // 2
  if CLLocationManager.authorizationStatus() != .authorizedAlways {
    showAlert(withTitle:"Warning", message: "Your geotification is saved but will only be activated once you grant Geotify permission to access the device location.")
  }
  // 3
  let region = self.region(withGeotification: geotification)
  // 4
  locationManager.startMonitoring(for: region)
}
```
来逐步分析下:

1. isMonitoringAvailableForClass(\_:)表示设备是否含有进行围栏检测所需的硬件。若监测功能不可用则显示一个合适的警告。showSimpleAlertWithTitle(\_:message:viewController)是一个Utilities.swift中的辅助函数，接收标题与消息数据，弹出一个警告视图。
2. 接着，需要检查授权状态，确保app得到使用位置服务所需的许可。若app未被授权，则无法接受到任何围栏相关的通知。
   不过，即使在app未被授权的情况下，也要允许用户保存地理通知，Core Location是允许你注册围栏的。在用户在之后给app授权时，自动启用这些围栏的监测功能。
3. 使用输入的地理通知与之前定义的帮助方法创建一个CLCircularRegion实例。
4. 最后，使用Core Location注册CLCircularRegion实例进行监测。

在启用监测的方法搞定后，也需要一个当用户移除地理通知时停止监测的方法。

在GeotificationsViewController.swift文件中，在startMonitoringGeotificiation(\_:)下面添加如下方法:

```swift
func stopMonitoring(geotification: Geotification) {
  for region in locationManager.monitoredRegions {
    guard let circularRegion = region as? CLCircularRegion, circularRegion.identifier == geotification.identifier else { continue }
    locationManager.stopMonitoring(for: circularRegion)
  }
}
```
这个方法命令locationManager停止监测与输入的地理通知关联的CLCircularRegion。

好了，现在完成启动和停止的方法了，会在添加或移除地理通知时使用这些方法，是时候进行添加通知部分的开发了。

首先，看看GeotificationsViewController.swift文件中的addGeotificationViewController(\_:didAddCoordinate)方法。

这个方法是由AddGeotificationViewController调用的委托方法来创建地理通知，这个方法负责使用来自AddGeotificationsViewController的值创建一个新的Geotification对象，并且更新对应的map view与地理通知列表。接着调用saveAllGeotifications()，更新地理通知列表，并保存到NSUserDefaults中。

使用如下代码替换这个方法:
```swift
func addGeotificationViewController(controller: AddGeotificationViewController, didAddCoordinate coordinate: CLLocationCoordinate2D, radius: Double, identifier: String, note: String, eventType: EventType) {
  controller.dismiss(animated: true, completion: nil)
  // 1
  let clampedRadius = min(radius, locationManager.maximumRegionMonitoringDistance)
  let geotification = Geotification(coordinate: coordinate, radius: clampedRadius, identifier: identifier, note: note, eventType: eventType)
  add(geotification: geotification)
  // 2
  startMonitoring(geotification: geotification)
  saveAllGeotifications()
}
```
代码中的两个关键点:

1. 确保半径值与locationManager的maximumRegionMonitoringDistance属性一致，这个属性被定义为地理围栏的最大半径。这很重要，任何超过这个最大值的值都会使监测失败。
2. 调用startMonitoringGeotification(\_:)，使用Core Location注册与新添加的地理通知相关联的监测围栏。

现在，在app中已经可以注册新的监测围栏了，不过有一个限制：地理围栏是一个共享的系统资源，Core Location对于每个app限制注册的围栏数量最多为20个。

虽然对于这个限制有可变通的方法，对于本教程来说，限制一下用户添加的地理通知数量即可。

如下，在updateGeotificationsCount()方法中添加一行代码:
```swift
func updateGeotificationsCount() {
  title = "Geotifications (\(geotifications.count))"
  navigationItem.rightBarButtonItem?.isEnabled = (geotifications.count < 20)  // 添加这一行
}
```
当app的通知数量达到临界值时禁用导航栏上的添加按钮。

最后，来处理一下地理通知的移除操作，这个操作在mapView(\_:annotationView:calloutAccessoryControlTapped:)方法中进行，当用户点击每个标注点附带的delete按钮时会调用这个方法。

在mapView(\_:annotationView:calloutAccessoryControlTapped:)方法中添加调用stopMonitoring(geotification:)的代码:
```swift
func mapView(_ mapView: MKMapView, annotationView view: MKAnnotationView, calloutAccessoryControlTapped control: UIControl) {
  // 删除地理通知
  let geotification = view.annotation as! Geotification
  stopMonitoring(geotification: geotification)   // 添加的语句
  removeGeotification(geotification)
  saveAllGeotifications()
}
```
添加的语句会停止与地理通知相关围栏的监测工作，接着移除通知并将改动保存到NSUserDefaults中。

好了，现在app支持监测与取消监测用户的地理围栏，加油！

构建并运行工程，虽然看不到明显的变化不过app现在可以注册可监测的地理围栏了。然而还不能响应任何围栏事件，不用担心，这是接下来的任务！

![](https://koenig-media.raywenderlich.com/uploads/2015/03/These_eyes.png)

## 添加地理围栏响应事件

首先来实现一些处理错误的委托方法--这很重要，以免发生一些问题后不知何因。

进入GeotificationsViewController.swift文件, 在CLLocationManagerDelegate的扩展中添加如下方法:

```swift
func locationManager(_ manager: CLLocationManager, monitoringDidFailFor region: CLRegion?, withError error: Error) {
  print("Monitoring failed for region with identifier: \(region!.identifier)")
}
 
func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
  print("Location Manager failed with the following error: \(error)")
}
```
这几个委托方法仅仅以日志形式输出了location manager遇到的错误，方便调试。

> 注意: 你肯定会想在生产环境中的app更有效的处理这些错误，而不是使其悄无声息的出错，比如可以提示用户哪里出了问题。

接着，打开AppDelegate.swift，在这里添加监听与响应地理围栏的进入与离开事件的代码。

在文件头部引入CoreLocation框架，添加如下代码:
```swift
import CoreLocation
```
确保AppDelegate中包含一个CLLocationManager实例:
```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
  var window: UIWindow?
 
  let locationManager = CLLocationManager() // Add this statement
  ...
}
```
使用如下代码替换application(\_:didFinishLaunchingWithOptions:):
```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey : Any]? = nil) -> Bool {
  locationManager.delegate = self
  locationManager.requestAlwaysAuthorization()
  return true
}
```
这样就设置了在AppDelegate中接收与围栏相关的事件，不过你也许会想“为何我不让view controller而是AppDelegate来做这件事？”

app中注册的围栏一直处于被监测的状态，即使在app未运行时。若设备在app未运行的情况下触发了一个围栏事件，则iOS会自动从后台重启app。这使得AppDelegate是一个用来处理事件的理想入口，因为此时的view controller有可能还未加载或未准备好。

你也许还会想“一个新建的CLLocationManager实例是如何获知所监测的围栏的？”

实际上所有app中所注册的监测用围栏可以轻松地被所有location managers访问，因此location managers的初始化位置就无所谓了。挺棒的，不是吗？:]

现在剩下的就是实现响应围栏事件的相关委托方法了。在这么做之前，需要添加一个处理围栏事件的函数。

在AppDelegate.swift中添加如下方法:

```swift
func handleEvent(forRegion region: CLRegion!) {
  print("Geofence triggered!")
}
```

这个方法接收一个CLRegion对象并简单的输出一条文本信息。别担心-之后会实现具体的细节。

接下来，在AppDelegate.swift中的CLLocationManagerDelegate扩展下添加如下委托方法，并添加一个刚刚创建的handleRegionEvent(\_:) 函数的调用:

```swift
extension AppDelegate: CLLocationManagerDelegate {
 
  func locationManager(_ manager: CLLocationManager, didEnterRegion region: CLRegion) {
    if region is CLCircularRegion {
      handleEvent(forRegion: region)
    }
  }
 
  func locationManager(_ manager: CLLocationManager, didExitRegion region: CLRegion) {
    if region is CLCircularRegion {
      handleEvent(forRegion: region)
    }
  }
}
```

建议使用合适的名称来给方法起名，当设备进入一个CLRegion区域时启动locationManager(\_:didEnterRegion:)，离开CLRegion区域时启动locationManager(\_:didExitRegion:)。

这些方法都会返回CLRegion，需要检查并确保是否为CLCircularRegion对象，因为有可能是一个CLBeaconRegion对象若app监测的是iBeacon区域的话。若确实是一个CLCircularRegion对象，则调用handleRegionEvent(\_:)。

> 注意: 围栏事件仅会在iOS设备跨越边界时才会被触发。若用户已经在注册的围栏中，则iOS不会产生事件。苹果提供了一个requestStateForRegion(\_:)方法用来查询设备位置是在所给围栏的里面还是外面。

现在app可以接收到围栏事件了，是时候来测试一下了。:]

测试app时最准确的方法是将程序部署到你的设备上，添加一些地理通知，然后去行走或驾车测试。

不过现在最好不要那样做，现在还不能确认触发生产环境中的设备围栏事件时输出的日志信息，除此之外，在实际测试前保证app能正常工作也是极好的。

幸运的是，有一个不用离开舒适的家里就可以搞定的简单方法，Xcode允许在工程中使用一个硬编码的GPX文件来模拟测试位置。为了方便起见在开始工程中就已经包含这个文件了。:]

在Supporting Files文件夹中找到并打开TestLocations.gpx, 选择inspect它的内容，会看到如下所示的代码:

```swift
<?xml version="1.0"?>
<gpx version="1.1" creator="Xcode">
  <wpt lat="37.422" lon="-122.084058">
    <name>Google</name>
  </wpt>
  <wpt lat="37.3270145" lon="-122.0310273">
    <name>Apple</name>
  </wpt>
</gpx>
```
GPX文件实际上是一个包含下面两个基准点的XML文件：山景城的Google总部与库比蒂诺的苹果总部。

构建并运行工程，开始模拟GPX文件中的位置。当app启动主view controller后，回到Xcode中，选择Debug工具栏的Location图标，然后选择TestLocations:

![](https://koenig-media.raywenderlich.com/uploads/2015/03/Screen-Shot-2015-02-02-at-9.04.37-pm.png)

回到app, 使用导航栏左上的缩放按钮找到当前位置。接近这个区域时，会看到一个位置标记在Google与苹果总部间不断移动。

通过添加一些基准点路径上的地理通知来测试app。若在实现注册地理通知之前添加过一些通知，则这些通知是不起作用的，需要清除一下并重启。

对于测试的位置，一个好的点子是将地理通知放置在每个基准点上，可以使用下面的测试方案:

* Google: 半径: 1000m, 信息: “Say Bye to Google!”, 离开时通知
* Apple: 半径: 1000m, 信息: “Say Hi to Apple!”, 进入时通知

![](https://koenig-media.raywenderlich.com/uploads/2016/06/Geo2Fences-281x500.png)

在添加通知后，每次当位置标记进入或离开一个围栏时都会在控制台输出一条日志。若按了home键或锁屏将app放置在了后台，当设备每次穿过围栏时同样能看到日志信息，虽然不能在屏幕中看到并确认移动的动作。
![](https://koenig-media.raywenderlich.com/uploads/2015/03/GeofenceTriggered.png)

> 注意: 位置模拟可以在iOS模拟器中使用也可以在真机中使用。不过，在iOS模拟器中的精度较低，触发事件的时机与看到的模拟位置进出围栏的时刻不是很吻合。最好用自己的真机来进行位置模拟，或者直接出门走走试试！

## 添加围栏事件的通知

已经完成了app中的大部分内容了，现在剩下的工作就是实现当设备穿过一个地理通知的围栏时提醒用户的功能了。

为了获取与触发的CLCircularRegion关联的记录，需要检索NSUserDefaults中对应的地理通知。这不难，只要使用在注册时分配给CLCircularRegion的identifier就可以找到正确的地理通知。

进入AppDelegate.swift,在类的底部添加如下辅助方法:

```swift
func note(fromRegionIdentifier identifier: String) -> String? {
  let savedItems = UserDefaults.standard.array(forKey: PreferencesKeys.savedItems) as? [NSData]
  let geotifications = savedItems?.map { NSKeyedUnarchiver.unarchiveObject(with: $0 as Data) as? Geotification }
  let index = geotifications?.index { $0?.identifier == identifier }
  return index != nil ? geotifications?[index!]?.note : nil
}
```

这个辅助方法会根据identifier从持久化数据中检索并返回地理通知。

现在可以检索到与围栏相关联的记录了，需要再编写一段代码，用于当围栏事件启动时发送一个通知，将记录作为显示的信息。

在application(\_:didFinishLaunchingWithOptions:)方法的底部，return语句之前添加如下代码:

```swift
application.registerUserNotificationSettings(UIUserNotificationSettings(types: [.sound, .alert, .badge], categories: nil))
UIApplication.shared.cancelAllLocalNotifications()
```

添加的代码用于使用户给予app发送通知的许可，并做了一下扫除-清楚已存在的通知。

接着，使用如下代码替换handleRegionEvent(\_:):

```swift
func handleEvent(forRegion region: CLRegion!) {
  // 若app正在运行则显示一个警告框
  if UIApplication.shared.applicationState == .active {
    guard let message = note(fromRegionIdentifier: region.identifier) else { return }
    window?.rootViewController?.showAlert(withTitle: nil, message: message)
  } else {
    // 否则显示一个本地通知
    let notification = UILocalNotification()
    notification.alertBody = note(fromRegionIdentifier: region.identifier)
    notification.soundName = "Default"
    UIApplication.shared.presentLocalNotificationNow(notification)
  }
}
```

若app正在运行，则会显示一个包含记录信息的警告视图，否则发送包含相同信息的本地通知。

构建并运行工程，在测试中触发围栏事件时，会看到一个警告视图，显示提醒信息:

![](https://koenig-media.raywenderlich.com/uploads/2016/06/GeoSayByeToGoogle-281x500.png)

在测试过程中按下Home键或锁屏将app置于后台，依然可以周期性的收到围栏事件的通知:

![](https://koenig-media.raywenderlich.com/uploads/2016/09/IMG_2239-281x500.png)

至此，已经亲手完成了一个功能全面的位置提醒app了。好了，出门试试app吧！
