# 前言

[AFNetworking](https://github.com/AFNetworking/AFNetworking) 是 iOS 开发里一个经常使用的库，我们有必要了解它的实现。下面会按: 发出 HTTP 请求 -> HTTP 请求结果返回 的两个步骤来分析源码。 

# 如何使用

首先说一下简单的使用,例如发出一个 `GET` 请求:

```objc
    [[AFHTTPSessionManager manager] GET:urlString
                         parameters:parametersDict
                           progress:^(NSProgress * _Nonnull downloadProgress) {
                              //进度
    }                       success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
                              //请求成功
    }                       failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) { 
                              //请求失败
    }];
```

# 请求过程

## AFHTTPSessionManger 

如上面代码看到的，要利用 `AFNetworking` 发出一个 HTTP 请求，需要使用到 `AFHTTPSessionManger`.

### AFHTTPSessionManager - 初始化

AFHTTPSessionManager.m 内有 5 个初始化获取实例的方法:

```objc
///---------------------
/// @name Initialization
///---------------------

+ (instancetype)manager;

- (instancetype)init;

- (instancetype)initWithBaseURL:(nullable NSURL *)url;

- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration;

- (instancetype)initWithBaseURL:(nullable NSURL *)url
           sessionConfiguration:(nullable NSURLSessionConfiguration *)configuration NS_DESIGNATED_INITIALIZER;

```

--插图说明--

而最后都是落在 ` initWithBaseURL: sessionConfiguration:` 方法里去做初始化：

```objc
- (instancetype)initWithBaseURL:(NSURL *)url
           sessionConfiguration:(NSURLSessionConfiguration *)configuration
{
    //调用父类 AFURLSessionManager 的 initWithSessionConfiguration: 初始化
    self = [super initWithSessionConfiguration:configuration];
    if (!self) {
        return nil;
    }
    
    //确保 baseURL 路径里的斜杠，给不以 "/" 为后缀的 url 末尾加一个斜杠
    if ([[url path] length] > 0 && ![[url absoluteString] hasSuffix:@"/"]) {
        url = [url URLByAppendingPathComponent:@""];
    }
    self.baseURL = url;
     
    //设置 请求 和 响应 的序列化器
    self.requestSerializer  = [AFHTTPRequestSerializer serializer];
    self.responseSerializer = [AFJSONResponseSerializer serializer];

    return self;
}
```

#### 关于 baseURL 的拼接

因为程序会使用 `[NSURL URLWithString:relativeToURL:]` 做地址拼接，所以要专门判断 baseURL 的后缀，防止出现意外的情况:

```
    NSURL *baseURL = [NSURL URLWithString:@"http://example.com/v1/"];
    [NSURL URLWithString:@"foo" relativeToURL:baseURL];                  // http://example.com/v1/foo
    [NSURL URLWithString:@"/foo" relativeToURL:baseURL];                 // http://example.com/foo
```

如上所示， baseURL 要拼接的地址 "foo"，应该是没有 "/" 做前缀，这就要保证 baseURL 自己是一定有 "/" 做后缀。


### AFHTTPSessionManger - HTTP 请求的 6 种方法

看完初始化到设置，再回到 AFHTTPSessionManger ，它提供了 HTTP 请求的 6 种方法，分别是:

- GET: `[sessionManager  GET:parameters:success:failure:]` / `[sessionManager  GET:parameters:progress: success:failure:]`.
- HEAD: `[sessionManager  HEAD:parameters:success:failure:]`
- POST: `[sessionManager  POST:parameters:success:failure:]` / `[sessionManager  POST:parameters:progress: success:failure:]`/`[sessionManager  POST:parameters: constructingBodyWithBlock: success:failure:]`/`[sessionManager  POST:parameters:constructingBodyWithBlock:progress: success:failure:]`
- PUT: `[sessionManager  PUT:parameters:success:failure:]`
- PATCH: `[sessionManager  PATCH:parameters:success:failure:]`
- DELETE: `[sessionManager  DELETE:parameters:success:failure:]`

## AFURLSessionManager 


### AFURLSessionManager - 初始化

接下来，看父类 AFURLSessionManager 做了什么。AFURLSessionManager 被调用的 方法 `initWithSessionConfiguration:` 的代码如下：

```
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    //设置 configuration 
    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }
    self.sessionConfiguration = configuration;

    //设置 operationQueue
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    //设置 session 
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

    // 设置响应请求的序列化器
    self.responseSerializer = [AFJSONResponseSerializer serializer];

    //设置安全策略
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

    //设置网络状态管理器
#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

    //设置 Task 的 delegate 标记
    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    //设置锁
    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    //设置 session 的 task 回调 （生成 session 之后调用） 
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        
        // dataTasks 数组循环，为对应的 task 添加代理
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        // uploadTasks 数组循环，为对应的 uploadTask 添加代理
        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        // downloadTasks 数组循环，为对应的 downloadTask 添加代理
        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}
```

#### operationQueue 数量限制

我们可以看到：

```objc
self.operationQueue.maxConcurrentOperationCount = 1;
```

将队列的最大并发数量，限制为 1 个，使得 operationQueue 为串行队列，会一个一个的执行任务。

#### getTasksWithCompletionHandler 获取未完成的 task

根据苹果文档的定义：
>The returned arrays contain any tasks that you have created within the session, not including any tasks that have been invalidated after completing, failing, or being cancelled.

`[self.session getTasksWithCompletionHandler]`获取当前 session 的所有未完成的task（不包括失败和取消的）。它会在从后台返回时，获得执行。 


### AFURLSessionManager - 构建 HTTP 请求的 Task 

对于上述的 6 种 HTTP 请求方式，基本都使用了同一个方法进行请求,属于 `AFURLSessionManager` 里面的方法:

```objc
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    
    //通过 self.requestSerializer 创建一个 request 
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    //判断是否解析错误
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    //创建一个 dataTask
    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```

一个 HTTP 请求，需要对应的字段信息 , 方法，host, URI(参数)，

这里首先是需要表示的 `method` 请求方法. `URLString` 请求地址.`parameters` 请求参数。上传/下载/成功/失败的 Block 回调。


### AFHTTPRequestSerializer - 生成请求

上面的代码用到了 requeset 对象，它由` AFHTTPRequestSerializer<AFURLRequestSerialization>` 类型的 self.requestSerializer 调用生成:
 
```objc
NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
```
 
 在 AFHTTPRequestSerializer.m 的生成的方法代码：
 
 ```objc
 - (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    //参数断言检查 method 和 URLString 不能为空 
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    NSURL *url = [NSURL URLWithString:URLString];
    //检查 url 不为空
    NSParameterAssert(url);
    
    //生成对应 url,method 到一个 request
    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;

    //将request的各种属性循环遍历
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        //如果自己观察到的发生变化的属性，包含在这些方法之中的话
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            //把给自己设置的属性给request设置
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }

    //对请求进行序列化
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
 ```
 
#### AFHTTPRequestSerializerObservedKeyPaths() 获取观察的属性
 
 上面用到的 `AFHTTPRequestSerializerObservedKeyPaths()` 是一个 C 函数:
 
 ```
 static NSArray * AFHTTPRequestSerializerObservedKeyPaths() {
    static NSArray *_AFHTTPRequestSerializerObservedKeyPaths = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _AFHTTPRequestSerializerObservedKeyPaths = @[NSStringFromSelector(@selector(allowsCellularAccess)), NSStringFromSelector(@selector(cachePolicy)), NSStringFromSelector(@selector(HTTPShouldHandleCookies)), NSStringFromSelector(@selector(HTTPShouldUsePipelining)), NSStringFromSelector(@selector(networkServiceType)), NSStringFromSelector(@selector(timeoutInterval))];
    });

    return _AFHTTPRequestSerializerObservedKeyPaths;
}
 ```

这个函数只执行一次，监听 AFHTTPRequestSerializer/NSMutableURLRequest 上面相关的属性为: 

- allowsCellularAccess
- cachePolicy 
- HTTPShouldHandleCookies
- HTTPShouldUsePipelining
- networkServiceType
- timeoutInterval
 
再看看 mutableObservedChangedKeyPaths ：

```objc
@property (readwrite, nonatomic, strong) NSMutableSet *mutableObservedChangedKeyPaths;
```

在 `[AFHTTPRequestSerializer init]` 方法中:

```objc
...
    // 初始化 mutableObservedChangedKeyPaths
    self.mutableObservedChangedKeyPaths = [NSMutableSet set];
    
    //为对应的属性，添加观察者方法
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }
...
```

对应的 KVO 方法里的处理:

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(__unused id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if (context == AFHTTPRequestSerializerObserverContext) {
    
        //变更后的新值，是否为 Null.是就移除，不是就添加到 mutableObservedChangedKeyPaths 中。
        if ([change[NSKeyValueChangeNewKey] isEqual:[NSNull null]]) {
            [self.mutableObservedChangedKeyPaths removeObject:keyPath];
        } else {
            [self.mutableObservedChangedKeyPaths addObject:keyPath];
        }
    }
}
```

#### 对请求的参数序列化

对应协议 AFURLRequestSerialization 里的方法，在  AFHTTPRequestSerializer 里的实现为 ：

```objc
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        //判断 field 为 key 的值，从 self.mutableHTTPRequestHeaders 取处来是不是为空
        if (![request valueForHTTPHeaderField:field]) {
            //为空则重新赋值
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    NSString *query = nil;
    if (parameters) {
        // queryStringSerialization 是一个 AFQueryStringSerializationBlock 类型的 block,看它是否有设置。
        if (self.queryStringSerialization) {//设置了则把各种类型的参数，array dic set转化成字符串，给request
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {//没有设置过，queryStringSerialization，用默认解析方式
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {//如果是GET、HEAD、DELETE，则把参数quey是拼接到url后面的
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {//而POST、PUT是把query拼接到http body中
        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        //设置请求的 body
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```

#### 对请求参数的拼接

默认方式用的 AFQueryStringFromParameters(NSDictionary *parameters) 实现如下:

```objc
NSString * AFQueryStringFromParameters(NSDictionary *parameters) {
    NSMutableArray *mutablePairs = [NSMutableArray array];
    for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
        [mutablePairs addObject:[pair URLEncodedStringValue]];
    }
    //把数组中的元素，用 & 做间隔，组合成一个字符串
    return [mutablePairs componentsJoinedByString:@"&"]; 
}

```

里面用到了 AFQueryStringPairsFromDictionary 来组织 key/value,
```objc
NSArray * AFQueryStringPairsFromDictionary(NSDictionary *dictionary) {
    return AFQueryStringPairsFromKeyAndValue(nil, dictionary);
}
```

调用了 AFQueryStringPairsFromKeyAndValue :

```objc
NSArray * AFQueryStringPairsFromKeyAndValue(NSString *key, id value) {
    NSMutableArray *mutableQueryStringComponents = [NSMutableArray array];
    // 根据需要排列的对象的description来进行升序排列，并且selector使用的是compare:
    // 因为对象的description返回的是NSString，所以此处compare:使用的是NSString的compare函数
    // 即@[@"foo", @"bar", @"bae"] ----> @[@"bae", @"bar",@"foo"
    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];
    
    
    //根据 value 的类型，选择不同的 key 的参数形式.例如数组是用 key[],字典是用 key[nestedKey].然后一直递归调用，直到为单项元素，变为 AFQueryStringPair 对象。 
    if ([value isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dictionary = value;
        // Sort dictionary keys to ensure consistent ordering in query string, which is important when deserializing potentially ambiguous sequences, such as an array of dictionaries
    
        for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            id nestedValue = dictionary[nestedKey];
            if (nestedValue) {
                [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
            }
        }
        
    } else if ([value isKindOfClass:[NSArray class]]) {
        NSArray *array = value;
        for (id nestedValue in array) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
        }
    } else if ([value isKindOfClass:[NSSet class]]) {
        NSSet *set = value;
        for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue(key, obj)];
        }
    } else {
        [mutableQueryStringComponents addObject:[[AFQueryStringPair alloc] initWithField:key value:value]];
    }

    return mutableQueryStringComponents;
}

```


#### AFQueryStringPair

利用 AFQueryStringPair 来对参数做对应 key/value 拼接，调用 `URLEncodedStringValue` 方法，变成 HTTP 中对参数形式 "xx=xx".

```objc
- (instancetype)initWithField:(id)field value:(id)value {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.field = field;
    self.value = value;

    return self;
}

- (NSString *)URLEncodedStringValue {
    if (!self.value || [self.value isEqual:[NSNull null]]) {
        return AFPercentEscapedStringFromString([self.field description]);
    } else {
        return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
    }
}

```

### AFURLSessionManager - 生成 dataTask

我们通过如下方法，最终生成 dataTask 来进行我们的请求。

```objc

- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}

```	

首先创建一个 block , 在里面把 request 放进去得到 dataTask，再为 dataTask 添加对应的代理。


创建 dataTask 的 `url_session_manager_create_task_safely` 方法:

```objc
static void url_session_manager_create_task_safely(dispatch_block_t block) {
    if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
        // Fix of bug
        // Open Radar:http://openradar.appspot.com/radar?id=5871104061079552 (status: Fixed in iOS8)
        // Issue about:https://github.com/AFNetworking/AFNetworking/issues/2093
        dispatch_sync(url_session_manager_creation_queue(), block);
    } else {
        block();
    }
}

```

判断当前的系统版本 `NSFoundationVersionNumber`, `NSFoundationVersionNumber_With_Fixed_5871104061079552_bug` 实际为 `NSFoundationVersionNumber_iOS_8_0`.

如果是小于 iOS8 的版本，要使用 dispatch_sync 把 block 同步提交到一个串行队列之中.如果是大于 iOS8 的版本，直接执行 block.

#### 为 dataTask 添加代理

添加代理的代码如下 :

```objc
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    //初始化一个 delegate,设置它的 manager 和完成的回调
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:dataTask];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

     //设置 taskDescription 
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}

```
 
使用如下函数，将 task 对应的 delegate 存储起来：


```objc
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    //参数断言，检查参数不得为空
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    //上锁
    [self.lock lock];
     
    //将 task.taskIdentifier 作为 key , delegate 作为 value 存储起来。
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    //添加 task 对应的通知
    [self addNotificationObserverForTask:task];
    
    [self.lock unlock];
}
```

使用下面的代码指定 task 为通知发送方:

```objc
- (void)addNotificationObserverForTask:(NSURLSessionTask *)task {
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidResume:) name:AFNSURLSessionTaskDidResumeNotification object:task];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidSuspend:) name:AFNSURLSessionTaskDidSuspendNotification object:task];
}

```


## 文件上传

比较特殊的是使用 POST 上传，是基于另外一个方法: 

```
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
     constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                      progress:(nullable void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer multipartFormRequestWithMethod:@"POST" URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters constructingBodyWithBlock:block error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    __block NSURLSessionDataTask *task = [self uploadTaskWithStreamedRequest:request progress:uploadProgress completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(task, error);
            }
        } else {
            if (success) {
                success(task, responseObject);
            }
        }
    }];

    [task resume];

    return task;
}
```

上面的 requestSerializer 使用到了下面这个方法 :

```objc
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(method);
    //不可以 GET 或者 HEAD 方法
    NSParameterAssert(![method isEqualToString:@"GET"] && ![method isEqualToString:@"HEAD"]);

    NSMutableURLRequest *mutableRequest = [self requestWithMethod:method URLString:URLString parameters:nil error:error];

    __block AFStreamingMultipartFormData *formData = [[AFStreamingMultipartFormData alloc] initWithURLRequest:mutableRequest stringEncoding:NSUTF8StringEncoding];

    if (parameters) {
       //把参数进行循环
        for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
            NSData *data = nil;
            if ([pair.value isKindOfClass:[NSData class]]) {
                data = pair.value;
            } else if ([pair.value isEqual:[NSNull null]]) {
                data = [NSData data];
            } else {
                data = [[pair.value description] dataUsingEncoding:self.stringEncoding];
            }

            if (data) {
                //追加数据到 formData 里
                [formData appendPartWithFormData:data name:[pair.field description]];
            }
        }
    }

    if (block) {
        block(formData);
    }

    return [formData requestByFinalizingMultipartFormData];
}

```

首先初始化了一个 `AFStreamingMultipartFormData` 类型的数据

### AFURLSessionManagerTaskDelegate 为 task 添加代理

这里逐一分析，如何对这些 task 做处理

```
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:dataTask];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

### AFSecurityPolicy 设置

TODO - HTTPS

## AFSecurityPolicy


# 响应处理

## AFURLSerialization


# HTTPS 策略的处理

