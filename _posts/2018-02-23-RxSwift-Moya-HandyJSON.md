---
title: RxSwift + Moya + HandyJSON
date: 2018-02-23 09:47:22
categories:
  - iOS
---


###   RxSwift     简单介绍与使用

* 什么是RxSwift：RxSwift是一种函数响应式编程的第三方库，可以让代码中的所有事件驱动都更容易进行管理，增强了代码的可读性。函数响应式编程可以大大简化异步操作，可以像操作变量一样操作一个闭包。 
* 什么是响应式编程：基本思想，你的程序可以对底层数据的变化做出响应，而不需要你直接告诉它。这样，你可以更专注于所需要处理的业务逻辑，而不需要去维护特定的状态。 对应的KVO，就是这种机制。 而RxSwift就是利用观察者机制实现了响应式编程。
具体内容可以参考这几篇文章：

<!-- more -->

[Getting Started With RxSwift and RxCocoa](http://southpeak.github.io/2017/01/16/Getting-Started-With-RxSwift-and-RxCocoa/)

[RxSwift 函数响应式编程](https://academy.realm.io/cn/posts/slug-max-alexander-functional-reactive-rxswift/)


###   二.Moya 简单介绍与使用

Moya是作用在Alamofire之上，让我们不再直接去使用Alamofire了，Moya也就可以看做我们的网络管理层，只不过他拥有更好更清晰的网络管理。可以看到下图，我们的APP直接操作Moya，让Moya去管理请求，不在跟Alamofire进行接触。

![](https://upload-images.jianshu.io/upload_images/49368-4bf918cee5f14d1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

而Moya的使用又非常简单。只需要实现Moya的一个协议TargetType，配置相关的网络请求代码即可。 定义一个枚举，用于列举所需要的进行的网络请求方法。 实现的TargetType协议中，baseURL为基地址。path为请求路径等等看代码即可理解，需要注意的是配置参数需要在task中配置。(Swift3.0时期可以直接配置参数)。

网上大多数例子都是3.0时期的写法，4.0时期发生了较大变化，踩了很多坑。

```
import Foundation
import Moya

enum HDZQNetworkTool {
    case getBannerImageList
    case getGoodsList(firsrId:String,secondId:String)
    case getWenchuangRelativedList(String)
}

extension HDZQNetworkTool: TargetType {
    var baseURL: URL {
        return URL.init(string: "http://59.49.25.69:65200")!
    }
    
    var path: String {
        switch self {
        case .getBannerImageList:
            return "api/wenchuang/get_wenchuang_banner"
        case .getGoodsList(firsrId:_,secondId:_):
            return "/api/wenchuang/get_wenchuang_list_by_class"
        case .getWenchuangRelativedList(_):
            return "/api/wenchuang/wenchuang_detail"
        }
    }
    
    var headers: [String : String]? {
        return ["Content-Type":"application/json"]
    }
    
    // 该方法已过期，配置参数在task中配置
//    var parameters: [String : Any]? {
//        switch self {
//        case .getBannerImageList:
//            return ["p":"i"]
//        case .getGoodsList(firsrId:let firstId,secondId:let secondId):
//            return ["p":"i","api_token":"","first_class": firstId,"second_class": secondId]
//        case .getWenchuangRelativedList(let wenchuangId):
//            return ["p":"i","api_token":"","id":wenchuangId]
//        }
//    }
    
    var method: Moya.Method {
        return .get
    }
    
    var sampleData: Data {
        return "".data(using: String.Encoding.utf8)!
    }
    
    var task: Task {
        switch self {
        case .getBannerImageList:
             return .requestParameters(parameters: ["p":"i"], encoding: URLEncoding.default)
        case .getGoodsList(firsrId:let firstId,secondId:let secondId):
            return .requestParameters(parameters: ["p":"i","api_token":"","first_class": firstId,"second_class": secondId], encoding: URLEncoding.default)
        case .getWenchuangRelativedList(let wenchuangId):
            return .requestParameters(parameters: ["p":"i","api_token":"","id":wenchuangId], encoding: URLEncoding.default)
        }
    }
    
    var validate: Bool {
        return false
    }
}

```

以上即完成了一个网络请求框架，所有的网络请求api均在这个类中。 



```
provider.request(.getGoodsList(firsrId: "1", secondId: "1")) { (result) in
            switch result {
            case let .success(moyaResponse):
                let data = moyaResponse.data
                let statusCode = moyaResponse.statusCode
            case let .failure(_): break
            }
        }
```
返回的结果可能是success也可能是failure。

可以阅读[GitHub文档](https://github.com/Moya/Moya)


###   三.HandyJSON的简单介绍与使用

HandyJSON是阿里大神开源的一个JSON转模型的库，我觉得这个较为好用，所以使用它来转模型。
需要注意的是，Moya/RxSwift是不支持HandyJSON的，所以需要我们对其进行扩展，使之能够支持HandyJSON

```
import Foundation
import RxSwift
import Moya
import HandyJSON

extension ObservableType where  E == Response {
    public func mapModel<T: HandyJSON>(_ type:T.Type) -> Observable<T> {
        return flatMap {response -> Observable<T> in
            return Observable.just(response.mapModel(T.self))
            
        }
    }
}

extension Response {
    func mapModel<T:HandyJSON>(_ type:T.Type) -> T {
        let JSONString = String.init(data: data, encoding: .utf8)
        return JSONDeserializer<T>.deserializeFrom(json: JSONString)!
    }
}
```


###  四.优雅的网络请求与数据转换

如果利用RxSwift与Moya共同使用的话，那么请求时写法就需要发生改变，同时配合HandyJSON，快速简单实现一个网络请求。


```
 provider.rx.request(.getGoodsList(firsrId: "1",secondId:"1")).asObservable().mapModel(GoodsModel<priceModel> .self)
            .subscribe({ (event) in
                if let model = event.element{
                    print(model.data)
                    let priceModels = model.data
                    self.dataArray.value = priceModels
                } else {
                    print(event)
                }
            })
            .disposed(by: disposeBag)
```


provider 与 disposeBag的定义为全局 dataArray为数据源


```
let disposeBag = DisposeBag()
let provider = MoyaProvider<HDZQNetworkTool>()
var dataArray = Variable([priceModel]())
```

###   五.RxSwift中如何使用tableview

将第四步中得到的dataArray与tableview进行绑定，无需实现tableview的datasource方法，利用的storyboard实现cell高度自适应。 非常简洁。 当然如果自己实现也可以，只不过该方法将会失效。 delegate方法同理。

*  此处利用xib格式创建的cell无法被正确使用  必须在storyboard中使用。否则会出现
     reason: 'unable to dequeue a cell with identifier GoodsTableViewCell - must register a nib or a class for the identifier or connect a prototype cell in a storyboard'  （可能是我操作不当？大家可以试一下）
     

```
dataArray.asObservable().bind(to: tableView.rx.items(cellIdentifier: "GoodsTableViewCell", cellType: GoodsTableViewCell.self)) {
            row,model,cell in
            cell.titleLabel.text = model.title
            cell.priceLabel.text = model.price
            cell.goodsImageView.kf.setImage(with: URL.init(string: ("http://59.49.25.69:65200" + model.list_img)))
            }
            .disposed(by: disposeBag)
```


下面这个是tableview的点击方法。。 


```
tableView.rx.modelSelected(priceModel.self).subscribe { (event) in
            let model = event.element
            print(model!)
            let SB = UIStoryboard.init(name: "Main", bundle: nil)
            let vc = SB.instantiateViewController(withIdentifier: "DetailViewController") as! DetailViewController
            vc.model = model!
            self.navigationController?.pushViewController(vc, animated: true)
        }.disposed(by: disposeBag)
```



###   总结

利用RxSwift进行响应式编程是目前Swift编程的一个大的趋势，由命令式编程过渡到响应式编程，带来的代码体验是非常棒的。借助RxSwift帮助我们迈入MVVM的大门。 不过RxSwift的学习成本较高，以上是我几天的成果，也只是出来了一个小Demo，对于其中的很多API，很多写法也没有完全掌握，再加上网络上的学习资料又大多数都是3.0时期的（有些甚至会给你带进坑），故而加大了学习难度。 不过如果能够掌握RxSwift，带来的收获也是巨大的，从RxSwift在GitHub收获的12000多star也可以看出，该框架的重要性。 


###   后续


现在的[demo](https://gitee.com/xxxxxxzq/RxSwiftDemo.git)只是代码的一些杂糅，没有利用MVVM进行操作， 后面将会写上一些数据绑定，实现一个以MVVM为架构的小Demo。


