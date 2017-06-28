---
title: ios-app-signer 源码解析
date: 2017-06-27 20:29:35
tags: [逆向, 源码]
---

在之前的制作[非越狱抢红包插件](http://swiftyper.com/2016/12/26/wechat-redenvelop-tweak-for-non-jailbroken-iphone/)文章中，我们曾经使用过 [ios-app-signer](https://github.com/buginux/ios-app-signer) 对微信进行过重签名。`ios-app-signer` 提供了简单易用的图形界面来帮助我们进行重签名，但是我们今天要透过它的图形界面，深入到源码中学习下 iOS 应用重签名的原理。

<!-- more -->

首先，从 `ios-app-signer` 的 [Github 仓库](https://github.com/buginux/ios-app-signer)上将其 `clone` 下来。

> 我自己 fork 了一份源码，并为其添加了一些中文注释，你也可以直接 clone [我的这份源码]()。

打开项目后会发现，里面的源文件不少，不过重签名的主要逻辑全都在 `MainView.swift` ，所以我们只需要阅读这个文件就可以了。

首先，从 `awakeFromNib` 这个入口方法开始，这个方法内依次调用了 `populateProvisioningProfiles` 和 `populateCodesigningCerts`，这两个方法分别用于填充界面上描述文件列表和证书列表。

## 获取描述文件列表

在 `populateProvisioningProfiles` 方法中调用了 `ProvisioningProfile` 的一个类方法 `getProfiles`，这个类用于表示一个描述方法的实体，而该方法用于获取系统中的所有描述文件。

系统中的描述文件都存放在 `~/Library/MobileDevice/Provisioning Profiles` 目录中，所以 `getProfiles` 方法只是遍历了该目录，并将遍历所得到的描述文件列表进行返回而已。

```swift
if let libraryDirectory = fileManager.urls(for: .libraryDirectory, in: .userDomainMask).first {
      let provisioningProfilesPath = libraryDirectory.path.stringByAppendingPathComponent("MobileDevice/Provisioning Profiles") as NSString
      if let provisioningProfiles = try? fileManager.contentsOfDirectory(atPath: provisioningProfilesPath as String) {
          
          for provFile in provisioningProfiles {
              if provFile.pathExtension == "mobileprovision" {
                  let profileFilename = provisioningProfilesPath.appendingPathComponent(provFile)
                  if let profile = ProvisioningProfile(filename: profileFilename) {
                      output.append(profile)
                  }
              }
          }
      }
}
```

在获取到描述文件列表后，`populateProvisioningProfiles` 对该列表进行了筛选，并删除了其中过期的描述文件。

```swift
for profile in provisioningProfiles {
  zeroWidthPadding = "\(zeroWidthPadding)\(zeroWidthSpace)"
  if profile.expires.timeIntervalSince1970 > Date().timeIntervalSince1970 {
      newProfiles.append(profile)
      
      ProvisioningProfilesPopup.addItem(withTitle: "\(profile.name)\(zeroWidthPadding) (\(profile.teamID))")
      
      let toolTipItems = [
          "\(profile.name)",
          "",
          "Team ID: \(profile.teamID)",
          "Created: \(formatter.string(from: profile.created as Date))",
          "Expires: \(formatter.string(from: profile.expires as Date))"
      ]
      ProvisioningProfilesPopup.lastItem!.toolTip = toolTipItems.joined(separator: "\n")
      setStatus("Added profile \(profile.appID), expires (\(formatter.string(from: profile.expires as Date)))")
  } else {
      setStatus("Skipped profile \(profile.appID), expired (\(formatter.string(from: profile.expires as Date)))")
  }
}
```

## 获取证书列表

`populateCodesigningCerts` 方法调用了 `getCodesigningCerts` 进行证书列表的获取。

系统中的证书都是存放在 `keyChain` 当中的，但是我们可以通过 `security find-identity -v -p codesigning` 命令来进行获取。`getCodeSigningCerts` 方法中也是这样做的，它只是在获取到列表之后再对它的格式进行了一些处理而已。

```swift
func getCodesigningCerts() -> [String] {
   var output: [String] = []
   let securityResult = Process().execute(securityPath, workingDirectory: nil, arguments: ["find-identity","-v","-p","codesigning"])
   if securityResult.output.characters.count < 1 {
       return output
   }
   let rawResult = securityResult.output.components(separatedBy: "\"")
   
   var index: Int
   
   for index in stride(from: 0, through: rawResult.count - 2, by: 2) {
       if !(rawResult.count - 1 < index + 1) {
           output.append(rawResult[index+1])
       }
   }
   return output
}
```

## 重签名的步骤

有了描述文件跟证书，再加上需要进行重签名的输入文件，就可以开始进行重签名的步骤了。

在 `ios-app-signer` 中新开了一个线程进行重签名的操作，方法名是 `signingThread`，而所有重签名的步骤都写在该方法里面了，可以说这个方法是整个应用的关键方法。

这个方法一开始先进行了一些变量的声明，以及参数的检查，接着创建了一些临时目录，用于存放重签名过程中生成的一些临时文件。

接下来，它使用用户选择的描述文件跟证书对自身进行了重签名测试，主要目的应该是判断描述文件与证书是否匹配，并且可以正常使用。这个方法里面也进行重签名操作，不过我们暂时先略过，后面再详细讲解。

```swift
func testSigning(_ certificate: String, tempFolder: String ) -> Bool? {
   let codesignTempFile = tempFolder.stringByAppendingPathComponent("test-sign")
   
   // Copy our binary to the temp folder to use for testing.
   // 将自身的可执行文件（ios-app-signer）复制到临时目录，以进行签名测试
   let path = ProcessInfo.processInfo.arguments[0]
   if (try? fileManager.copyItem(atPath: path, toPath: codesignTempFile)) != nil {
       _ = codeSign(codesignTempFile, certificate: certificate, entitlements: nil, before: nil, after: nil)
       
       let verificationTask = Process().execute(codesignPath, workingDirectory: nil, arguments: ["-v",codesignTempFile])
       try? fileManager.removeItem(atPath: codesignTempFile)
       if verificationTask.status == 0 {
           return true
       } else {
           return false
       }
   } else {
       setStatus("Error testing codesign")
   }
   return nil
}
```

再后面，就是对用户的输入文件进行处理了，`ios-app-signer` 支持了四种类型文件的重签，分别是 `.deb`，`.ipa`，`.app` 以及 `.xcarchive` 文件。

其中，`.deb` 跟 `.ipa` 本质上是压缩文件，需要先对它们进行解压缩，而 `.app` 跟 `.xcarchive` 本质上是目录文件，所以可以直接进行处理。

这里以 `.app` 类型的文件来进行说明。

```swift
case "app":
  //MARK: --Copy app bundle
  if !inputIsDirectory.boolValue {
      setStatus("Unsupported input file")
      cleanup(tempFolder); return
  }
  do {
      try fileManager.createDirectory(atPath: payloadDirectory, withIntermediateDirectories: true, attributes: nil)
      setStatus("Copying app to payload directory")
      try fileManager.copyItem(atPath: inputFile, toPath: payloadDirectory.stringByAppendingPathComponent(inputFile.lastPathComponent))
  } catch {
      setStatus("Error copying app to payload directory")
      cleanup(tempFolder); return
  }
  break
```

这里在临时目录中新建了一个 `Payload` 目录，并将 `.app` 拷贝到该目录中。之后，就是真正地对 `.app` 进行重签名了。

### 重签名

首先，需要将 `Info.plist` 中的 `CFBundleREsourceSpecification` 删除，这个 key 貌似是跟资源的签名有关。

```swift
//MARK: Delete CFBundleResourceSpecification from Info.plist
Log.write(Process().execute(defaultsPath, workingDirectory: nil, arguments: ["delete",appBundleInfoPlist,"CFBundleResourceSpecification"]).output)
```

接着，就是将用户选择的描述文件拷贝到 app bundle 中，并将其命名为 `embedded.mobileprovision`。

```swift
setStatus("Copying provisioning profile to app bundle")
do {
    try fileManager.copyItem(atPath: provisioningFile!, toPath: appBundleProvisioningFilePath)
} catch let error as NSError {
    setStatus("Error copying provisioning profile")
    Log.write(error.localizedDescription)
    cleanup(tempFolder); return
}
```

然后，生成 `entitlements.plist`，这个文件指定了该应用可以使用哪些权限，即我们平常在 Xcode 的  `Capabilities` 选项卡下选择的权限。这步很容易被忽略，网上的很多教程也都少了这一步，因此重签名也都无法成功。

`entitlements.plist` 的内容可以从描述文件中的 `Entitlements` 对应的值中取得，这里使用了 `PlistBuddy` 进行读取和写入。

```swift
func getEntitlementsPlist(_ tempFolder: String) -> NSString? {
   let mobileProvisionPlist = tempFolder.stringByAppendingPathComponent("mobileprovision.plist")
   do {
       try self.rawXML.write(toFile: mobileProvisionPlist, atomically: false, encoding: String.Encoding.utf8)
       let plistBuddy = Process().execute("/usr/libexec/PlistBuddy", workingDirectory: nil, arguments: ["-c", "Print :Entitlements",mobileProvisionPlist, "-x"])
       if plistBuddy.status == 0 {
           return plistBuddy.output as NSString?
       } else {
           Log.write("PlistBuddy Failed")
           Log.write(plistBuddy.output)
           return nil
       }
   } catch let error as NSError {
       Log.write("Error writing mobileprovision.plist")
       Log.write(error.localizedDescription)
       return nil
   }
}
```

因为在操作的过程中，我们有可能改变了可执行权限，所以我们需要再设置下可执行文件的权限为 755。

```swift
if let bundleExecutable = getPlistKey(appBundleInfoPlist, keyName: "CFBundleExecutable"){
    _ = Process().execute(chmodPath, workingDirectory: nil, arguments: ["755", appBundlePath.stringByAppendingPathComponent(bundleExecutable)])
}
```

最后，也是最重要的一步，就是使用描述文件与证书对应用进行重签名。`ios-app-signer` 中使用了递归对 app bundle 中所有支持的类型进行重签，然后再对自身进行了重签。

```swift
func generateFileSignFunc(_ payloadDirectory:String, entitlementsPath: String, signingCertificate: String)->((_ file:String)->Void){
     let useEntitlements: Bool = ({
         if fileManager.fileExists(atPath: entitlementsPath) {
             return true
         }
         return false
     })()
     
     func shortName(_ file: String, payloadDirectory: String)->String{
         return file.substring(from: payloadDirectory.endIndex)
     }
     
     func beforeFunc(_ file: String, certificate: String, entitlements: String?){
             setStatus("Codesigning \(shortName(file, payloadDirectory: payloadDirectory))\(useEntitlements ? " with entitlements":"")")
     }
     
     func afterFunc(_ file: String, certificate: String, entitlements: String?, codesignOutput: AppSignerTaskOutput){
         if codesignOutput.status != 0 {
             setStatus("Error codesigning \(shortName(file, payloadDirectory: payloadDirectory))")
             Log.write(codesignOutput.output)
             warnings += 1
         }
     }
     
     func output(_ file:String){
         codeSign(file, certificate: signingCertificate, entitlements: entitlementsPath, before: beforeFunc, after: afterFunc)
     }
     return output
 }
                
 let signableExtensions = ["dylib","so","0","vis","pvr","framework","appex","app"]
 
 //MARK: Codesigning - App
 let signingFunction = generateFileSignFunc(payloadDirectory, entitlementsPath: entitlementsPlist, signingCertificate: signingCertificate!)
 
 recursiveDirectorySearch(appBundlePath, extensions: signableExtensions, found: signingFunction)
 signingFunction(appBundlePath)
```

其实的 `codeSign` 方法执行了重签名的操作，这个方法其实就是调用了 `codesign` 命令来进行重签名。使用该命令进行重签名的方法：

```bash
$ codesign -vvv -fs <证书> --entitlements=<entitlements.plist> --no-strict <重签名文件>
```

至此，重签名的步骤就全部完成了。

## 小结

虽然 `ios-app-signer` 执行重签名的步骤还比较多，但其实里面有很多是处理其它情况的代码。而对于普通的重签名，可以总结为以下几个步骤：

1. 解压 .ipa 文件，获取到对应的 .app 目录
2. 找到你要使用的描述文件（如果没有，就使用开发者帐号写一个 Demo 应用生成一个）
3. 将描述文件拷贝到 .app 目录下，并全名为 `embedded.mobileprovision`
4. 找到描述文件对应的证书
5. 生成一个 entitlements.plist 文件（文件内容可以直接从描述文件中获取）
6. 删除 _CodeSignature 目录，这里面存放的是旧的资源文件签名
    * 在 ios-app-signer 中并没有这一条操作，因此猜测删除`Info.plist` 中的 `CFBundleREsourceSpecification` 作用与操作的效果一样
7. 使用 `codesign` 进行重签名
8. 将 .app 重新打包成 .ipa

## 参考资料

* [ios-app-signer](https://github.com/buginux/ios-app-signer)
* [Inside Code Signing](https://www.objc.io/issues/17-security/inside-code-signing/)
* [iOS 签名的原理](http://blog.cnbang.net/tech/3386/)

## 关注

如果你喜欢这篇文章，可以关注我的公众号，随时获取我最新的博客文章。

![qrcode_for_gh_6e8bddcdfca3_430.jpg](http://upload-images.jianshu.io/upload_images/650096-9155581b667f64b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)














