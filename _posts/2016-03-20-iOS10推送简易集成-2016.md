---
layout: post
title:  iOS10的通知增加图片的实现（一个坑)
subtitle:   iOS10推送通知
author: zhuly
date:   2016-11-10 12:00:00
catalog:    true
tags:
    - iOS
---

安卓小伙伴跟我讲你们的push可以放图片了，还不赶紧做，我们安卓可以放。好吧既然你有这个愿望我就满足你！
#实现
这里只完成了一个简单的在push边上多加一张图，因为没有要加其他拓展（其实我是懒）
不多说直接贴代码。
```
- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request withContentHandler:(void (^)(UNNotificationContent * _Nonnull))contentHandler {
    self.contentHandler = contentHandler;
    self.bestAttemptContent = [request.content mutableCopy];

    NSURL *url = [NSURL URLWithString:request.content.userInfo[@"image"]];
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
    NSURLSessionDataTask *task = [session dataTaskWithURL:url completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        if (error) {
            NSLog(@"download attachment image error : %@", error);
        }else{
            NSString *path = [NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES).firstObject
                              stringByAppendingPathComponent:@"download"];
            NSFileManager *manager = [NSFileManager defaultManager];
            if (![manager fileExistsAtPath:path]) {
                [manager createDirectoryAtPath:path withIntermediateDirectories:YES attributes:nil error:nil];
            }
            NSString *fileName = [NSString stringWithFormat:@"%lld.jpg", (long long)[[NSDate date] timeIntervalSince1970] * 1000];
            path = [path stringByAppendingPathComponent:fileName];
            UIImage *image = [UIImage imageWithData:data];
            NSLog(@"path : %@", path);
            
            NSError *err = nil;
            [UIImageJPEGRepresentation(image, 1) writeToFile:path options:NSAtomicWrite error:&err];
            
            UNNotificationAttachment *attachment = [UNNotificationAttachment attachmentWithIdentifier:@"remote-atta1" URL:[NSURL fileURLWithPath:path] options:nil error:&err];
            if (attachment) {
                self.bestAttemptContent.attachments = @[attachment];
            }
        }
        
        self.contentHandler(self.bestAttemptContent);
    }];
    
    [task resume];
}
```
先通过左上角的file -> New ->target增加一个NotificationService的extension就好了。然后里面提供了2个方法。将如上代码放入第一个方法中就完成了。这里要注意的点是：
###第一点

```
if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 10.0) {
        UNUserNotificationCenter *notificationCenter = [UNUserNotificationCenter currentNotificationCenter];
        
        notificationCenter.delegate = [UIApplication sharedApplication].delegate;
        
        [notificationCenter requestAuthorizationWithOptions:UNAuthorizationOptionBadge | UNAuthorizationOptionSound | UNAuthorizationOptionAlert completionHandler:^(BOOL granted, NSError * _Nullable error) {
            
            NSLog(@"%d--%@", granted, error);
            
        }];;
        [notificationCenter getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings * _Nonnull settings) {
            
        }];
        
    }
    
    if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 8.0)
    {
        [[UIApplication sharedApplication] registerUserNotificationSettings:[UIUserNotificationSettings
                                                                             settingsForTypes:(UIUserNotificationTypeSound | UIUserNotificationTypeAlert | UIUserNotificationTypeBadge)
                                                                             categories:nil]];
        
        
        [[UIApplication sharedApplication] registerForRemoteNotifications];
    }
    else
    {

        [[UIApplication sharedApplication] registerForRemoteNotificationTypes:(UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound | UIRemoteNotificationTypeAlert)];

    }

```
注册通知的时候记得区分版本。

###第二点（真的好气啊）

我在测试是时候用的是smartpush。一个push的测试小工具，我通过request.content.userInfo将image取出来（这里完全是根据你自己push过来的数据格式决定）。这时候我发现通知一直有而图就是没有出来。为什么会这样。是我哪里实现有问题吗？测试了大概一天，我突然想难道图有毒，我就换了一张图。意外的发现原来图的尺寸是有问题的，之前用的是一张大图虽然大小只有97KB但是依旧无法加载。之后换了一张小尺寸的成功了！当时就喜极而泣。如果有小伙伴要测试。一定要用一张小尺寸的图。不然你可能跟我一样悲惨。