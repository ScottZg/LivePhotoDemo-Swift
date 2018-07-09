## 前言
Apple从iPhone6s开始支持Live Photo。Live Photo 会录下拍照前后 1.5 秒所发生的一切，因此用户获得的不仅仅是一张精美照片，还有拍照前后时刻的动作和声音。具体的操作可以参见[拍照和编辑](https://support.apple.com/zh-cn/HT207310)。
本文接下来要介绍的是如何在项目开发过程中使用Live Photo以及兼容其他平台使用Live Photo。这些平台包括iOS、Web和Android。接下来就开始进行介绍。

## 正文
先了解几个概念。
HEVC：全称High Efficiency Video Coding。它是一种高效的视频编码，是符合行业标准的下一代视频编码技术，继承自H.264编码。Apple想要添加新的功能特性，但是当前的H.264已经无法满足Apple的需求，因此HEVC应运而生。
HEIF：全称High Efficiency Image File(Format)，是一种高效率的图片文件格式，是中静止图像和图像序列的现代容器格式。
苹果从iOS11开始已经默认启动了HEVC电影和HEIF图像存储。也就是说iOS11以及以后版本的手机拍摄的图片默认存储的格式都是HEIF。但是我们可以尝试将手机拍摄的图片发送给其他人，你会发现图片的格式依然是JPG。这是Apple做了兼容，让拍摄的照片更好地跨平台支持。但是如果你用Mac上的Photo（应用）将Live Photo以原图的形式导出，你会发现它导出的内容不再是JPG格式的文件，而是一个HEIC文件+一个mov文件。  
Apple其实是通过图片+视频的方式实现了Live Photo。
先简单介绍多平台展示Live Photo的思路：
苹果手机用户将Live Photo上传到服务器，此时上传的是一张图片+视频。当展示的时候分以下几种情况：

1. 对于苹果手机的用户，可以从服务端获取图片+视频，然后将其合成Live Photo进行展示
2. 对于Android手机用户，可以模拟Live Photo，将图片覆盖到视频上，然后进行隐藏展示播放。当播放时隐藏图片，让视频播放；当停止播放时显示图片覆盖视频，停止视频播放
3. 对于Web用户，可以直接使用Apple官方提供的[LivePhotosKit JS](https://developer.apple.com/documentation/livephotoskitjs)，按照其使用方法将图片和视频加载到DOM元素中展示。Apple也提供了官方的一个Web展示Live Photo的Demo，点击[这里](https://developer.apple.com/live-photos/)查看。

接下来分平台进行操作处理。

### iOS
首先，我们如果想要手动获取Live Photo的源文件，苹果推荐了下面几种方式：
#### 1.Using macOS Image Capture
* Connect your iOS device to your Mac.(使用数据线将设备连接到你的Mac)
* Select the Live Photo you wish to import from your device to your local file system.(选择你想要导出到你本地文件系统的Live Photo)
* Choose the destination folder and click on Import.(选择你的目标文件夹，然后点击导入)

#### 2.Using macOS Photos
* Connect your iOS device to your Mac.(将你的iOS设备和Mac相连)
* Import your photos into the Photos application.(把你手机上的图片导入到Photos应用程序中)
* Select the Live Photo you wish to export.(选中你想要导入的Live Photo)
* Use File > Export > Export Unmodified Original to export to your file system.(导出，选择导出一张未修改的原件即可)

#### 3.Using Windows 10 File Explorer
* Ensure that iTunes for Windows is installed. You can download it from here: http://www.apple.com/itunes/download/
* Open File Explorer. This can be opened by pressing the Windows Key and E at the same time.
* Connect your iOS device to your PC.
* You should see your iOS device in the "This PC" folder.
* Navigate to the following folder: (your device) > Internal Storage > DCIM and look for the Live Photo you wish to import.
* Your Live Photo will be stored as a pair of files: a JPG file and a MOV file.
* Drag the pair of files to your local file system.

导出之后，得到了两个文件：一个是后缀为HEIC的图像文件，一个是mov后缀的视频文件。此时，便可以手动将图片+视频上传到Server，然后供其他端使用。
如果是用户使用自己的iOS设备上传图片，我们可以先通过PHAssetCollection或者PHAsset获取图片，这里有个demo：我通过PHAsset.fetchAssets(with:photoOptions)可以获取手机上面所有的图片。还有一个PHAssetCollection的类，它代表图库中的组，例如时刻、用户创建的相册或者是smart album。我们可以使用该类获取所有的smartAlbum集合：

```swift
var smartAlbums: PHFetchResult<PHAssetCollection>!   //smart albums

 smartAlbums = PHAssetCollection.fetchAssetCollections(with: .smartAlbum, subtype: .albumRegular, options: nil)
```
这里的.smartAlbum就是图库中的组的集合，是一个枚举：

```swift
public enum PHAssetCollectionType : Int {
    case album
    case smartAlbum
    case moment
}

```
此时拿到的smartAlbums就是一组group，每个group中又包含了符合该组条件的图片例如：
![Demo页面展示](https://raw.githubusercontent.com/ScottZg/MarkDownResource/master/livephoto/demo.PNG)
左边Smart Albums是获取到的smartAlbums，里面对图片做了智能分类，包括最近删除的、屏幕快照、Live Photos、Videos等等。右边是点击Live Photos进入的页面。里面全部是Live Photo。图片缩略图页面的数据是通过上一个页面传入的group中单个collection：

```swift
  imgListVC.photosList = PHAsset.fetchAssets(in: smartAlbums.object(at: indexPath.row), options: nil)
```

这里的PHAsset.fetchAssets是从某个PHAssetCollection中获取该Collection中的所有图片集合，返回结果:

```swift
var photosList: PHFetchResult<PHAsset>? = nil
```
也就是PHFetchResult类型，是一个结果集。拿到结果集之后，便可以在图片列表页面展示：

```swift
  func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let collectionViewCell = collectionView.dequeueReusableCell(withReuseIdentifier: CellIdentifier, for: indexPath) as! ImageCollectionCell
        
        let asset = photosList?.object(at: indexPath.row)
        if (asset?.mediaSubtypes.contains(.photoLive))! {
            collectionViewCell.badgeImage = PHLivePhotoView.livePhotoBadgeImage(options: .overContent)
        }
        imageManager.requestImage(for: asset!, targetSize: CGSize.init(width: 80, height: 80), contentMode: .aspectFill, options: nil, resultHandler: { image, _ in
            // The cell may have been recycled by the time this handler gets called;
            // set the cell's thumbnail image only if it's still showing the same asset.
           collectionViewCell.smallImage = image
        })
        
        return collectionViewCell
    }
```

这里使用的UICollectionView充当容器。collectionViewCell.badgeImage（自定义的image，用于展示左上角的live photo标识）的获取方式很独特，是PhotosUI中自带的API获取的：

```swift
PHLivePhotoView.livePhotoBadgeImage(options: .overContent)
```
PHLivePhotoView是继承与UIview的一个子类，可以把它理解为UIImageView，只不过UIImageView是用于展示静态图片，而PHLivePhotoView用于展示Live Photo。该类有一个livePhotoBadgeImage的方法用于获取live photo的标识图片,选项.overContent是Live Photo正常展示的角标，而.liveOff则是在角标上添加了斜杠，可自行尝试。
接下来就是获取要展示的图片，这里使用到了PHCachingImageManager类，该类主要是提供用于检索或者生成预览图像。所以展示的预览图就是通过该类生成的。调用它的requestImage方法，将asset传入，便可获UIImage对象。
当点击某个图片进去详情页面时，通过传入的asset便可获取Live Photo，并在PHLivePhotoView上展示：

```swift
 PHImageManager.default().requestLivePhoto(for: asset, targetSize: view.frame.size, contentMode: .aspectFit, options: options) { (livePhoto, info) in
            guard livePhoto != nil else {return}
            self.livePhotoImageView.livePhoto = livePhoto
            
        }
```

这里使用的是PHImageManager，可以通过该类获取 PHLivePhoto对象。

写了这么多，只是从相册中获取了Live Photo，然后将其展示。那如何获取该Live Photo的源文件呢？很简单，直接看下面代码：

```swift
  @objc func getSourceAction() {
        let arr = PHAssetResource.assetResources(for: asset)
        let manager = PHAssetResourceManager.default()
        let resourceReqOptions = PHAssetResourceRequestOptions.init()
        manager.requestData(for: arr[0], options: resourceReqOptions, dataReceivedHandler: { (data) in
            let image = UIImage.init(data: data, scale: 1)
            print(image ?? "没有图片")
        }) { (error) in
            print(error?.localizedDescription ?? "err")
        }
        print(arr)
    }
```
这是点击获取资源触发的Action操作，主要用到了PHAssetResource和PHAssetResourceManager。
PHAssetResource是于照片库中的图片视频或者Live Photo 相关连的底层数据资源，也就是说我可以通过此类获取Live Photo的图片+视频:
![PHAssetResource解释](https://raw.githubusercontent.com/ScottZg/MarkDownResource/master/livephoto/PHAssetResource.png)
通过PHAsset获取asset 资源数组，对Live Photo而言，数组包含了图片+视频。这样如果用户是通过iOS设备上传Live Photo，开发者可以获取到视频和图片分别上传。然后其他端通过使用图片+视频模拟Live Photo的展示。
还有一个问题，如果是iOS设备上，如何将网络获取的图片+视频展示位Live Photo呢？
既然Apple提供了API让开发者获取Live Photo的原始资源，也可以通过原始资源合成Live Photo：

```swift
    open class func request(withResourceFileURLs fileURLs: [URL], placeholderImage image: UIImage?, targetSize: CGSize, contentMode: PHImageContentMode, resultHandler: @escaping (PHLivePhoto?, [AnyHashable : Any]) -> Swift.Void) -> PHLivePhotoRequestID
```
此方法是PHLivePhoto的类方法，作用是根据提供的资源文件异步合成Live Photo。这个方法中的URL为一个数组，内容为使用Photos库导出的Live Photo的源文件(HEIC+mov)。
#### 将生成的Live Photo保存到本地
直接看代码：

```swift 
    PHPhotoLibrary.shared().performChanges({
            let request = PHAssetCreationRequest.forAsset()
            let options = PHAssetResourceCreationOptions.init()
            let imageUrl = Bundle.main.path(forResource: "livephoto1", ofType: "HEIC")!
            let vidoUrl = Bundle.main.path(forResource: "livephoto1", ofType: "mov")!
            request.addResource(with: .pairedVideo, fileURL: URL.init(fileURLWithPath: vidoUrl), options: options)
            request.addResource(with: .photo, fileURL: URL.init(fileURLWithPath: imageUrl), options: options)
        }) { (boo, error) in
            if boo {
                print("保存到手机成功")
            }else {
                print(error?.localizedDescription ?? "error")
            }
        }
```
这里主要使用的是PHAssetCreationRequest类。这里要注意一点，那就是LivePhoto的视频添加时， PHAssetResourceType为pairedVideo，这种类型是提供Live Photo原始视频数据的格式。通过add操作之后，可以将合成的Live Photo保存到手机中。
按照上述的方式，便可以在iOS平台上面去使用Live Photo。

### Android
Android本身不支持Live Photo，但是可以进行模拟。先从服务端拉取要展示的图片+视频，展示时，直接将图片覆盖到视频上，当进行按压时，隐藏图片，播放视频即可。

### Web
Apple为了做在线播放Live Photo，官方开发了一套[LivePhotoKit](https://developer.apple.com/documentation/livephotoskitjs)的js，通过该JS，开发者可以很容易地将图片+视频合称为Live Photo展示到网页中。这里是Apple官方提供的[Demo](https://developer.apple.com/live-photos/)。自己有按照LivePhotoKit的指南去开发，但是发现兼容性并不是很好，在Safari中展示没有什么问题，但是在Chrome和Firefox上展示提示播放失败。这里后续有待进一步研究。另外，在Web展示的时候如果你使用的外链图片和视频，容易产生跨域问题：

```html
 No 'Access-Control-Allow-Origin' header 
```
所以最好通过自己在本地起一个服务，然后同源进行操作。具体的LivePhotoKit使用可以直接查看官方网站的使用。

## 结束
LivePhoto本质上就是图片+视频生成的一种新的照片格式。在对其进行操作的过程中主要用到的Photos+PhotosUI。
代码Demo可参见[这里](https://github.com/ScottZg/LivePhotoDemo-Swift)。

如有什么疑问，可留言咨询！



## 参考链接
1.[LivePhotosKit JS](https://developer.apple.com/documentation/livephotoskitjs)   
2.[Example app using Photos framework](https://developer.apple.com/library/archive/samplecode/UsingPhotosFramework/Introduction/Intro.html)  
3.[Live Photo Editing and RAW Processing with Core Image](https://developer.apple.com/videos/play/wwdc2016/505/)  
4.[Working with HEIF and HEVC](https://developer.apple.com/videos/play/wwdc2017/511)  
5.[PHAssetResourceManager usage?](https://stackoverflow.com/questions/32925482/phassetresourcemanager-usage)  
6.[拍摄和编辑livephoto](https://support.apple.com/zh-cn/HT207310)  
7.[FLLivePhotoDemo](https://github.com/filelife/FLLivePhotoDemo)  
8.[Live Photo存储与应用](https://blog.upyun.com/?p=845)  
9.[iOS开发创建合成一张LivePhoto](https://juejin.im/post/5a3548cb51882529c70f3337)  
