### Swift 常用语法-总结



* **声明一个数组，里面嵌套字典**

  ```swift
  var array = [Dictionary<String, Any>]()
  ```

* **声明一个字典**

  ```swift
  var dic = [string: Any]()
  ```

* **截取字符串**

  ```swift
  let string = "ab1234ba"  
  let headRange = string.range(of: "b", options: .caseInsensitive, range: nil, locale: nil)!
  let tailRange = string.range(of: "b", options: .backwards, range: nil, locale: nil)!
  
  print(string[...headRange.lowerBound])	// ab
  print(string[headRange.upperBound...])	// 1234ba
  print(string[headRange.upperBound..<tailRange.lowerBound])	// 1234
  
  let res = String(string[headRange.upperBound..<tailRange.lowerBound]) // 包起来，否则报错
  ```

* **获取字符串的NSRange**

  ```swift
  let text = "helloword"
  let nsstring = NSString(string: text)
  let range = nsstring.range(of: "hello")
  range.location
  range.length
  ```

* **as! 的替代方式**

  ```
  if let tempArray = Util.getArray() as? [string] {
  	...
  }
  ```

* **将Any转为String**

  ```
          if let albumMidNum = albumInfo["mid"] as? NSNumber {
              let albumMidStr = "\(albumMidNum)"
          }
  ```

* **Type of expression is ambiguous without more context**

  ```
  表达式模糊不清，没有更多参数
  
      let songList = songs.map { (songInfo : SongInfo) -> [String : Any]? in
          return songInfo.dictionaryValue()  
          ->
          return songInfo.dictionaryValue() as? [String : Any]
      }
  ```

* **swift中按位 | & 的写法**

  ```
  OC中：
  	//OC里位移枚举的定义
      enum UIViewAnimationOptions option = UIViewAnimationOptionRepeat | UIViewAnimationOptionLayoutSubviews;
  
  	//OC里普通枚举的定义
      enum UIViewAnimationTransition option = UIViewAnimationTransitionFlipFromLeft;
      
  Swift中：
  	//swift里面位移枚举的定义
      let option:UIViewAnimationOptions = [.repeat, .layoutSubviews]
  
  	//swift里面普通枚举的定义
      let option:UIViewAnimationTransition = .flipFromLeft
  ```

* **UIImage to Data**

  ```
  let image = UIImage(named: "sample")
  let data = image?.pngData()
  let data = image?.jpegData(compressionQuality: 0.8)
  let image : UIImage = UIImage(data: imageData)
  
  // deprecated
  // var imageData : Data = UIImagePNGRepresentation(image)
  ```

* **Any?（Optional） 无法转为Int类型的**

  ```
  song["id"].intValue  * 报错
  
  let songId = JSON.init(song).dictionaryValue["id"]
  songId?.intValue ?? 0
  ```

* **how to translate OC BOOL to swift Bool**

  ```
  - (BOOL)translate {return YES;}
  
  let booleanValue = (Obj.translate != false)
  ```

* **Cannot convert value of type 'UnsafeMutableRawPointer?' to expected argument type 'UnsafeMutablePointer<Float>'**

  ```
  case:
  	let audioBuffer_0 : AudioBuffer = buffer.audioBufferList->mBuffers[0]  > Cannot find 'mBuffers' in scope
  	QMSpectrumManager.shareInstance().caculateFFT(withFloatBuffer: audioBuffer_0.mData)  > 报上述错误
  
  1、先将audioBufferList取出
  	let abl = UnsafeMutableAudioBufferListPointer(audioBufferList)
    abl[0] 就是 AudioBuffer
  2、取出AudioBuffer中的mData，并转换成 float * 指针类型
  	if let mData = abl[0].mData?.assumingMemoryBound(to: Float.self) {
    	QMSpectrumManager.shareInstance().caculateFFT(withFloatBuffer: mData)
  	}
  解决。
  
  fixed：
  	let audioBufferList = UnsafeMutablePointer(mutating: buffer.audioBufferList)
    let abl = UnsafeMutableAudioBufferListPointer(audioBufferList)
                  
    if let mData = abl[0].mData?.assumingMemoryBound(to: Float.self) {
    	QMSpectrumManager.shareInstance().caculateFFT(withFloatBuffer: mData)
  	}
  ```

  

