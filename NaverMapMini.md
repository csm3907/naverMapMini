# Naver Map Mini

## Application 구조

MVC 디자인 Pattern을 사용하여 Application 을 구현하였습니다.

![MVC Pattern](https://user-images.githubusercontent.com/16497594/103184142-fdf9df80-48f9-11eb-8309-2e30346434ec.png)

검색 데이터를 담은 Model 폴더의 SearchPlaceModel.swift 사용자에게 보여질 화면을 담당할 Storyboard 와 SubView 를 담는 View, 마지막으로 Model 과 View사이의 동적관리를 해주는 Controller 폴더는 ViewController 파일을 담고 있습니다. 

제가 MVC 패턴을 사용한 이유는 Project의 총 기간 3일 이내에 단 하나의 뷰만 존재하는 **심플한 구조에서** MVC  가장 적합하다고 생각하였습니다.  MVC 패턴의 대표적인 단점인 Massive View Controller 의 특성을 보안하기 위하여 


-> MVVM 과의 비교를 진행하자


![ViewController Extension](https://user-images.githubusercontent.com/16497594/103185098-38fe1200-48fe-11eb-8dc8-5c6a2c29cfb3.png)

위와 같이 해당 ViewController의 특정 기능의 경우 따로 + Extension 서비스명 의 명칭으로 파일을 신규하여 한 파일에 모든 서비스의 구현 내용이 들어가 있지 않도록 ( 너무 방대해지지 않도록 ) 개발하였습니다.


## Application 시작

<img src="https://user-images.githubusercontent.com/16497594/103250862-61971200-49b9-11eb-9adc-97cee54ae951.gif" alt="start View" style="zoom: 33%;" />


```Swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Override point for customization after application launch.
        LocationManager.shared.initialize()
        return true
    }
```

Application 이 기동 하면 **AppDelegate 의 didFinishLaunchingWithOptions 함수를 호출**하게 되고, 

LocationManager 클래스의 Singleton 객체를 통하여 fileprivate 변수 locationManager 를 **초기화** 하게 됩니다.

```Swift
func initialize() {
        locationManager = CLLocationManager()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        locationManager.startUpdatingLocation()
    }
```

1. locationManager.delegate 를 self 로 받아 CLLocationManagerDelegate 의 didUpdateLocations 함수를 통하여 지속적인 위치 파악이 가능하도록 개발하였으며

```Swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let localVal: CLLocationCoordinate2D = manager.location?.coordinate else { return }
        latitude = localVal.latitude
        longitude = localVal.longitude
        coordinate = localVal
        if latitude != 0.0 && longitude != 0.0 {
            notifyLocationAvailability()
        }
    }
```

2. LocationManager 의 위치기능이 정상적으로 사용이가능할 경우 

```Swift
fileprivate func notifyLocationAvailability() { // Location 의 사용가능을 NotificationCenter 에 등록한다.
        NotificationCenter.default.post(name: Notification.Name("LocationAvailable"), object: nil)
    }
```

NotificationCenter 를 통하여 "LocationAvailable" 을 등록하였습니다. 

---------------------------------------

#### Application MainView


```Swift
override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
        mapParentView.mapView.positionMode = .normal
        infoWindow.dataSource = defaultDataSource
        registerNib() // XIB 등록
        getLocationPermission() // 위치권한 설정 요구
        NotificationCenter.default.addObserver(self, selector: #selector(locationAvailabilityCheck(notification:)), name: NSNotification.Name("LocationAvailable"), object: nil)
    }
    }
```

viewDidLoad 함수에서 mapParentView (NMFNaverMapView) 객체의 mapView의 positionMode 를 Normal mode 로 default 설정을 하며, 해당 지역 marker 를 클릭하였을시 보여줄 정보를 저장하는 infoWindow 객체를 초기화 합니다. 

 또한 registerNib() 함수를 통하여 TableView 에서 사용할 TableViewCell의 xib를 등록하였으며, getLocationPermission() 함수를 통하여 위치권한 설정이 denied 혹은 restricted 인경우 AlertView 를 띄워 위치 서비스를 활성화 하지 않은 경우 Application을 이용하지 못하도록 진행을 하였습니다. 

마지막으로 LocationManager를 통하여 등록된 LocationAvailable observing하는 함수를 연결합니다.

```Swift
@objc func locationAvailabilityCheck(notification: Notification) { // Location이 사용이 가능하다는 Noti 를 받은경우 현위치를 찍어주는 버튼을 Mapview 에 보여준다.
        if LocationManager.shared.coordinate != nil {
            mapParentView.showLocationButton = true
        }
    }
```

해당 함수 내에서는 위치 서비스가 사용 가능하므로 MapView의 showLocationButton 을 활성화 하여 현재 자신의 위치를 찾아갈수 있는 버튼을 보여주도록 하였습니다.

---------------------------------------

## Search 서비스의 구현

### 1. NetworkManager 

1. 에러 타입의 정의

Http 통신 중 발생할 수 있는 에러 코드들을 정의한 Erro들을 Enum type 으로 정의합니다.

```Swift
enum NetworkResponse: String, Error {
    case success
    case authenticationError = "You need to be authenticated first"
    case badRequest = "Bad request"
    case outdated = "The url you requested is outdated"
    case failed = "Network request failed"
    case noData = "Response returned with no data to decode."
    case unableToDecode = "We could not decode the response"
}
```

2. 네트워크 통신 상태값을 Input으로 각 상황에 맞는 Error, 혹은 성공을 return 하는 함수를 정의 

```Swift
fileprivate func handleNetworkResponse(_ response: HTTPURLResponse) -> NetworkResponse {
    switch response.statusCode {
    case 200...299:
        return .success
    case 401...500:
        return NetworkResponse.authenticationError
    case 501...599:
        return NetworkResponse.badRequest
    case 600:
        return NetworkResponse.outdated
    default:
        return NetworkResponse.failed
    }
}
```

3. 마지막으로 URLSession 을 통한 Http 통신을 수행합니다.

```Swift
static func request(url: URL, headers: HTTPHeaders, parameters: Parameters, completion: @escaping ((_ result: Result<Data, NetworkResponse>) -> Void)) {
        
```

request 함수에서 input 으로 받는 Parameter 값은 가장 먼저 http 통신을 진행할 baseUrl, Http 통신의 헤더, body, 마지막으로 통신의 결과값을 return 할 completion Closure 를 전달받습니다. 

```Swift
var components = URLComponents(url: url, resolvingAgainstBaseURL: true)
```

URLComponents 객체를 생성하고 baseUrl 에 parameter 로 전달받은 값들을 quiryItems.append 함수를 통해 이어붙이기를 진행합니다.  이를 통하여 만들어진 Full Url 을

```Swift
var request = URLRequest(url: url, cachePolicy: .useProtocolCachePolicy, timeoutInterval: 60)
        request.httpMethod = "GET"
```

URLRequest 객체를 생성합니다.  timeInterval 을 60초로 정하여 timeOut 시간을 1분으로 설정하였습니다.
또한 검색 API 의 통신 방식이 GET 방식을 사용하므로 request 의 httpMethod 로 "GET" 을 지정하였습니다. 

마지막으로 

```Swift
let task = session.dataTask(with: request) { (data, response, error) in
            DispatchQueue.main.async {
                let httpResponse = response as? HTTPURLResponse
                guard let status = httpResponse else { completion(.failure(.authenticationError)); return }
                let result = handleNetworkResponse(status)
                
                if result == .success {
                    guard let sendData = data else { completion(.failure(.noData)); return }
                    completion(.success(sendData))
                } else {
                    completion(.failure(result))
                }
            }
        }
        task.resume()
```

data 통신을 진행하였을때 HttpResponse 의 status 값을 handleNetworkResponse 함수를 통하여 성공, 혹은 상황에 맞는 에러값을 return 하도록 한 후 completion handler 를 호출합니다.

task 의 통신 완료처리를 DispatchQueue 의 Main Thread 에서 진행하는 이유는 데이터를 받은 후 ViewController의 tableView 를 reload 하는 UI 작업을 하게 되므로 MainQueue 에서 동작하도록 코드를 작성하였습니다.

<img src="https://user-images.githubusercontent.com/16497594/103223541-d2aed900-4969-11eb-89f8-bbf33e66f26b.gif" alt="search" style="zoom: 33%;" />


### 2. NetworkOperation

-> Netwrok 를 빠르게 할 수 있는 방법 Multi Thread 를 통하여  여러개의 작업이 동시에 이루어지도록 하여  속도를 올린다. Cocoa Touch Framework 에서는 Concurrency 프로그래밍에 사용하는 방법으로 GCD, Operation 이렇게 2가지를 제공한다.

검색어를 통한 지역 검색의 결과가 (**최대 50가지**) 가장 빠르게 tableView에 load 하는 방식으로 Cocoa Touch Framework 에서 제공하는 Concurrency 프로그래밍을 생각하였고 그 중 Operation 을 사용하여 구현을 진행하였습니다. Operation 을 사용한 이유는 검색된 리스트가 50개 이상이 되었을시 취소 처리를 용이하기 하기 위해서 입니다. 


현재 네이버 지역 검색 API [지역검색](https://developers.naver.com/docs/search/local/) https://developers.naver.com/docs/search/local/ 의경우
start = 1, display = 1 ~ 5 사이의 숫자로 정해져 있어 start 가 1 ~ n 개로 늘어날 수 있다고 가정을 하였습니다.

```Swift
init(pageIndex: String,searchText: String, completion: @escaping ((_ result: Result<SearchPlaceModel, NetworkResponse>) -> Void))
```

먼저 NetworkOperation 의 초기화 함수를 살펴보면, pageIndex (API 의 start 값), searchText (검색어), 그리고 completion handler 로 구성되어 있습니다.


```Swift
override func main() {
        autoreleasepool {
            guard !isCancelled && !isFinished else { return } // cancel 되면 Operation 을 종료한다.
            SearchService.requestList(searchText: searchText, startPage: pageIndex, completion: completionHandler)
        }
    }
```

그리고 Operation 의 구현부분을 살펴보면 먼저 완료처리가 되었거나, 중간에 취소가 된경우 return 을 진행하며, 그외에 정상 케이스들의 경우 SearchService( NetworkManager 의 request 함수를 사용하는 서비스 클래스) 의 searchList 함수를 호출합니다.


### 3. ViewController

```Swift
@IBAction func searchBtnClicked(_ sender: Any) {
```

검색어 textField 를 통해 검색어를 입력받고, 검색 버튼을 누르면 searchBtnClicked 함수가 실행이 됩니다.

```Swift
for index in 1..<20 {
            let pageIndex = String(index)
            
            let networkOp = FetchSearchListOperation.init(pageIndex: pageIndex, searchText: searchText) { [unowned self] (result) in
```

for in 구문을 통하여 networkOp 라는 FetchSearchListOperation의 pageIndex 값을 1씩 증가시키며  객체들을 생성합니다. 

```Swift
let queue: OperationQueue = OperationQueue() // Concurrency 를 보장하는 Global Queue 생성
```

이렇게 생성한 Operation 객체들을  ViewController 의 Concurrency queue 인 OperationQueue() 에 

```Swift
queue.addOperation(networkOp) // Concurrent queue 에 Operation 값을 추가한다.
```

Concurrency queue 에 append 하고 Operation이 Concurrency 하게 실행됩니다.

```Swift
var searchPlaceData: SearchPlaceModel = SearchPlaceModel.init(lastBuildDate: "", total: 0, start: 1, display: 1, items: []) {
        didSet(oldValue) {
            if self.searchPlaceData.items.count > 50 {
                self.isFinished = true
                self.queue.cancelAllOperations() // queue 의 모든 Operation 들을 cancel 시킨다.
            }
            self.searchListTableView.reloadData() // data 가 변화하였을시에 TableView 를 다시 그려준다.
        }
    }
```

마지막으로 searchPlaceData 의 프로퍼티 감시자를통하여 item 의 갯수가 50개를 초과한경우 queue 의 모든 Operation이 cancel 되도록 구현을 하였습니다. 


### Network 리스트 Append 방식

```Swift
case .success(let searchData):
                    self.semaphore.wait() // semaphore 를 통하여 searchPlaceData 전역변수에 append 시키는 처리가 비동기 -> 동기 처리가 이루어지도록 반영한다. 
                    let noCommonArr: [Item] = searchData.items.filter{ (searchItem) in !self.searchPlaceData.items.contains(where: { (item) -> Bool in
                        return (item.mapx == searchItem.mapx) && (item.mapy == searchItem.mapy)
                    }) } // 중복을 제거한다.
                    self.searchPlaceData.items.append(contentsOf: noCommonArr) // 중복 제거한 값을 넣어준다.
                    self.semaphore.signal()
```

networkOperation 의경우 Concurrent 하게 동작을 하게 되므로 하나의 변수인 searchPlaceData 에 접근(붙이기)을 할시 append 가 정상적으로 동작하지 않거나, 메모리 참조 오류가 발생할수 있습니다. 

이와같은 현상을 방지하기 위하여 DispatchSemaphore 를 사용하였습니다.

```Swift
let semaphore: DispatchSemaphore = DispatchSemaphore(value: 1)
```

semaphore 의경우 value 의 값을 1로 초기화하여 처음 Network completion 에 도착하였을경우 semaphore.wait() 에서 코드가 block 되지 않고 다음으로 넘어가도록 하였으며, 이후에는 signal 과 wait 함수를 통해 searchPlaceData 에 값을 안전하게 Append 할수 있도록 구현 하였습니다. 


