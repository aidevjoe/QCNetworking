# QCNetworking


### 缓存
在开发中我们常常需要向服务器发送请求来获得数据, 正常的流程是这样:
1. 发送请求
2. 网络正常, 将数据刷新到UI
3. 无网络，没有数据, 视图空白

这样就会导致同一个URL请求多次，服务器返回的数据可能都是一样的，比如服务器上的某张图片，无论下载多少次，返回的数据都是一样的。这样的情况可能会导致一下问题:

1. 重复获取数据，造成用户流量的浪费
2. 请求响应速度可能很慢
3. 服务器不必要的压力
4. 没网络的情况下没有数据
5. ...

所以为了解决上面的问题，我们一般会对数据进行缓存处理，给用户更好的用户体验，减轻服务器的压力，实现缓存后的步骤:

1. 先判断是否有缓存数据，如果有先加载缓存数据
2. 判断有没有网络
3. 没网,结束.
4. 有网，继续请求，刷新数据，将服务器的数据缓存到硬盘


### 具体实现
在iOS中，苹果已经为我们提供了**NSURLCache**类来实现缓存

但是这里我并没有使用苹果提供的，而是第三方[YYCache](https://github.com/ibireme/YYCache)来实现对网络请求的缓存

使用[AFNetworking](https://github.com/AFNetworking/AFNetworking)进行网络请求管理


- 对GET请求进行缓存

```
#pragma mark - 发送 GET 请求

/**
 *   GET请求
 *
 *   @param url           url
 *   @param params        请求的参数字典
 *   @param cache         是否缓存
 *   @param successBlock  成功的回调
 *   @param failureBlock  失败的回调
 *   @param showHUD       是否加载进度指示器
 */
+ (NSURLSessionTask *)getRequestWithUrl:(NSString *)url
                                 params:(NSDictionary *)params
                                  cache:(BOOL)isCache
                           successBlock:(QCSuccessBlock)successBlock
                           failureBlock:(QCFailureBlock)failureBlock
                                showHUD:(BOOL)showHUD{
    
    __block NSURLSessionTask *session = nil;

    if (isCache) {
        
        id responseObject = [QCNetworkCache getCacheResponseObjectWithRequestUrl:url params:params];
        
        if (responseObject) {
            
            int code = 0;
            NSString *msg = nil;
            if (responseObject) {
                //这个字段取决于 服务器
                code                = [responseObject[@"rsCode"] intValue];
                msg                 = responseObject[@"rsMsg"];
            }
            successBlock ? successBlock(responseObject, code, msg) : 0;
        }
    }
    
    //没有网络直接返回
    if (networkStatus == QCNetworkStatusNotReachable) {
        failureBlock ? failureBlock(QC_ERROR) : 0;
        return session;
    }
    
    if(showHUD) NSLog(@"加载中");

    session = [_manager GET:url parameters:params progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        NSLog(@"加载完成");

        int code = 0;
        NSString *msg = nil;
        
        if (responseObject) {
            //这个字段取决于 服务器
            code                = [responseObject[@"rsCode"] intValue];
            msg                 = responseObject[@"rsMsg"];
        }
        successBlock ? successBlock(responseObject, code, msg) : 0;
        
        //缓存数据
        isCache ? [QCNetworkCache cacheResponseObject:responseObject requestUrl:url params:params] : 0;
        
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        NSLog(@"加载完成");
        failureBlock ? failureBlock(error) : 0;
    }];
    
    [session resume];
    
    return session;
}

```




保存缓存具体实现

```
/**
 *  对请求进行缓存
 *
 *  @param responseObject 需要缓存的数据
 *  @param requestUrl     请求url
 *  @param params         参数
 */
+ (void)cacheResponseObject:(id)responseObject
                 requestUrl:(NSString *)requestUrl
                     params:(NSDictionary *)params{
    assert(responseObject);
    assert(requestUrl);
    
    if (!params) params = @{};
    NSString *originString = [NSString stringWithFormat:@"%@-%@",requestUrl,params];
    NSString *hash = [self md5:originString];
    
    [cache setObject:responseObject forKey:hash withBlock:^{
        NSLog(@"成功 hash = %@", hash);
    }];
}


```


获取已缓存的请求

```
/**
 *  获取已缓存的请求
 *
 *  @param requestUrl 请求地址
 *  @param params     参数
 *
 *  @return 缓存的数据
 */
 + (id)getCacheResponseObjectWithRequestUrl:(NSString *)requestUrl
                                    params:(NSDictionary *)params{
    assert(requestUrl);
    
    if (!params) params = @{};
    NSString *originString = [NSString stringWithFormat:@"%@-%@",requestUrl,params];
    NSString *hash = [self md5:originString];
    
    id cacheData = [cache objectForKey:hash];
    
    return cacheData;
}


```



- 对下载请求进行缓存


```
#pragma mark - 文件下载

/**
 *  文件下载 (带缓存)
 *
 *  @param url           下载文件接口地址
 *  @param progressBlock 下载进度
 *  @param successBlock  成功回调
 *  @param failBlock     下载回调
 *  @param showHUD       是否加载进度指示器
 *
 *  @return 返回的对象可取消请求
 */
+ (NSURLSessionTask *)downloadWithUrl:(NSString *)url
                        progressBlock:(QCProgressBlock)progressBlock
                         successBlock:(QCDownloadSuccessBlock)successBlock
                         failureBlock:(QCFailureBlock)failureBlock
                              showHUD:(BOOL)showHUD{
    
    __block NSURLSessionTask *session = nil;
    
    NSString *type = nil;
    NSArray *subStringArr = nil;
    
    NSURL *fileUrl = [QCNetworkCache getDownloadDataFromCacheWithRequestUrl:url];
    
    if (fileUrl) {
        if (successBlock) successBlock(fileUrl);
        return session;
    }
    
    //没有网络直接返回
    if (networkStatus == QCNetworkStatusNotReachable) {
        failureBlock ? failureBlock(QC_ERROR) : 0;
        return session;
    }
    
    if(showHUD) NSLog(@"加载中");
    
    if (url) {
        subStringArr = [url componentsSeparatedByString:@"."];
        if (subStringArr.count > 0) {
            type = subStringArr[subStringArr.count - 1];
        }
    }
    
    //响应内容序列化为二进制
    _manager.responseSerializer = [AFHTTPResponseSerializer serializer];
    
    session = [_manager GET:url parameters:nil progress:^(NSProgress * _Nonnull downloadProgress) {
        
        progressBlock ? progressBlock((float)downloadProgress.completedUnitCount/(float)downloadProgress.totalUnitCount) : 0;

    } success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        NSLog(@"加载完成");
        
        if (successBlock) {
            NSData *data = (NSData *)responseObject;
            
            [QCNetworkCache saveDownloadData:data requestUrl:url];
            
            NSURL *downFileUrl = [QCNetworkCache getDownloadDataFromCacheWithRequestUrl:url];
            
            successBlock(downFileUrl);
        }
        
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        NSLog(@"加载完成");
        
        failureBlock ? failureBlock(error) : 0;

    }];
    
    [session resume];
    
    return session;
}

```


对下载的数据进行缓存

```
/**
 *  对下载的数据进行缓存
 *
 *  @param data       下载的数据
 *  @param requestUrl 请求url
 */
 
 + (void)saveDownloadData:(NSData *)data
              requestUrl:(NSString *)requestUrl {
    assert(data);
    assert(requestUrl);
    
    NSString *fileName = nil;
    NSString *type = nil;
    NSArray *strArray = nil;
    
    strArray = [requestUrl componentsSeparatedByString:@"."];
    if (strArray.count > 0) {
        type = strArray[strArray.count - 1];
    }
    
    if (type) {
        fileName = [NSString stringWithFormat:@"%@.%@",[self md5:requestUrl],type];
    }else {
        fileName = [NSString stringWithFormat:@"%@",[self md5:requestUrl]];
    }
    
    NSString *directoryPath = nil;
    directoryPath = [[NSUserDefaults standardUserDefaults] objectForKey:downloadDirKey];
    if (!directoryPath) {
        directoryPath = [[[NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) firstObject] stringByAppendingPathComponent:@"QCNetworking"] stringByAppendingPathComponent:@"download"];
        
        [[NSUserDefaults standardUserDefaults] setObject:directoryPath forKey:downloadDirKey];
        [[NSUserDefaults standardUserDefaults] synchronize];
    }
    
    NSError *error = nil;
    if (![[NSFileManager defaultManager] fileExistsAtPath:directoryPath isDirectory:nil]) {
        [[NSFileManager defaultManager] createDirectoryAtPath:directoryPath withIntermediateDirectories:YES attributes:nil error:&error];
    }
    
    if (error) {
        NSLog(@"创建目录错误: %@",error.localizedDescription);
        return;
    }
    
    NSString *filePath = [directoryPath stringByAppendingPathComponent:fileName];
    
    [[NSFileManager defaultManager] createFileAtPath:filePath contents:data attributes:nil];
}

```


获取已缓存的下载数据

```
/**
 *  获取已缓存的下载数据
 *
 *  @param requestUrl 请求url
 *
 *  @return 缓存的url路径
 */
 
 + (NSURL *)getDownloadDataFromCacheWithRequestUrl:(NSString *)requestUrl {
    
    assert(requestUrl);
    
    NSData *data = nil;
    NSString *fileName = nil;
    NSString *type = nil;
    NSArray *strArray = nil;
    NSURL *fileUrl = nil;
    
    strArray = [requestUrl componentsSeparatedByString:@"."];
    if (strArray.count > 0) {
        type = strArray[strArray.count - 1];
    }
    
    if (type) {
        fileName = [NSString stringWithFormat:@"%@.%@",[self md5:requestUrl],type];
    }else {
        fileName = [NSString stringWithFormat:@"%@",[self md5:requestUrl]];
    }
    
    
    NSString *directoryPath = [[NSUserDefaults standardUserDefaults] objectForKey:downloadDirKey];
    
    if (directoryPath){
        NSString *filePath = [directoryPath stringByAppendingPathComponent:fileName];
        data = [[NSFileManager defaultManager] contentsAtPath:filePath];
    }
    
    if (data) {
        NSString *path = [directoryPath stringByAppendingPathComponent:fileName];
        fileUrl = [NSURL fileURLWithPath:path];
    }
    
    return fileUrl;
}

```


### 具体代码实现[Github](https://github.com/Joe0708/QCNetworking)







