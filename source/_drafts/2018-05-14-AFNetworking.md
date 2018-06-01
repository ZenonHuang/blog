# 前言

[AFNetworking](https://github.com/AFNetworking/AFNetworking) 是 iOS 开发里一个经常使用的库，我们有必要了解它的实现。首先根据 HTTP 协议的定义，再来看它是如何实现的。

为方便大家理解，会按以下三个部分来进行说明 : 

**NSURLSession**

 - AFHTTPSessionManager 是使用方直接接触的类，是我们分析的入口。
 - AFURLSessionManger 负责了 dataTask 的构建和回调等大量的工作，会是重点分析的类。
  
**Serialization**
  
  - AFHTTPRequestSerializer 负责请求的序列化，一个 HTTP 请求需要有符合格式的地址，参数，请求方法。对它的主要内容有: 1.如何对不同类型的数据进行拆解，例如数组，字典。2.如何对数据进行拼接 3.如何根据方法不同，决定请求参数的写入位置。
  - AFHTTPResponseSerializer 响应结果的序列化

**Additional Functionality**
 
- AFSecurityPolicy
- AFNetworkReachabilityManager
- 如何处理 HTTPS 


# HTTP 协议

HTTP 协议是 HyperText Transfer Protocol 的缩写，翻译为超文本传输协议。


## HTTP 请求报文

一个 HTTP 请求的分为三部分组成:

HTTP 请求报文结构   |
------- | 
Request Line（请求行） | 
CRLF | 
Request Header（请求头） |
CRLF | 
Request Body（请求体，可选） |


>CRLF: CRLF由两个字节组成。CR值为16进制的0x0D，对应ASCII中的回车键，LF值为0x0A，对应ASCII中的换行键，CRLF合起来就是我们平常所说的\r\n，即: 回车符和换行符。

### 请求行

**Request Line** 包括： Method , URI , HTTP Version 。一个请求行看起来是这样的： 

Method  | 空格 | URI | 空格 | HTTP Version | CRLF 
------- | ------- | ------- | ------- | ------- | -------
GET     | | /user/avatar.jpg?t=1480992153.564331 | | HTTP/1.1 | \r\n 



其中利用空格，分割不同部分。 要注意的是， URI 可以是相对的路径， host 可以被放在请求头里，也可以是一个完整的 absoluteURI，包含 Schema 和 Host .

### 请求头

**Request Header** 相当于一个字典，里面存储一些键值对，键值对的形式为: `Key: 空格 Value CRLF`. 
 
 类似于下面:
> Host: abc.com\r\n
> User-Agent: xxxxfds\r\n
> ...

### 请求体

**Request Body** 是可选的，例如对于方法为 GET 的请求， body 是空的。而对于 POST 的请求说， body 一般不为空。

[扒一扒HTTP的构成](http://mrpeak.cn/blog/http-constitution/)

## HTTP 响应报文

HTTP 响应报文结构   |
------- | 
Status Line（响应行） | 
CRLF | 
Response Header（响应头） |
CRLF | 
ResPonse Body（响应体，可选） |

### 响应行

响应行，又叫 status line 或者 状态行 ，包含了 HTTP Version ， HTTP Status Code : 


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

# AFHTTPSessionManger 

如上面代码看到的，要利用 `AFNetworking` 发出一个 HTTP 请求，需要使用到 `AFHTTPSessionManger`.

## AFHTTPSessionManager - 初始化

AFHTTPSessionManager.m 内有 5 个初始化获取实例的方法:

```objc
+ (instancetype)manager;

- (instancetype)init;

- (instancetype)initWithBaseURL:(nullable NSURL *)url;

- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration;

- (instancetype)initWithBaseURL:(nullable NSURL *)url
           sessionConfiguration:(nullable NSURLSessionConfiguration *)configuration NS_DESIGNATED_INITIALIZER;

```

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
    
    //确保 baseURL 结尾是斜杠。对长度大于 0 且不以 "/" 为后缀的 url，它的末尾加一个斜杠
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

### 关于 baseURL 的拼接

因为程序中会使用 `[NSURL URLWithString:relativeToURL:]` 做地址拼接，所以要专门判断 baseURL 的后缀，防止出现意外的情况。例如:

```objc
    NSURL *baseURL = [NSURL URLWithString:@"http://example.com/v1/"];
    [NSURL URLWithString:@"foo" relativeToURL:baseURL];                  // http://example.com/v1/foo
    [NSURL URLWithString:@"/foo" relativeToURL:baseURL];                 // http://example.com/foo
```

如上所示， 如果使用 "/foo" ,我们地址会出现问题。因而 baseURL 要拼接的地址 "foo"，应该是没有 "/" 做前缀，这就要保证 baseURL 自己是一定有 "/" 做后缀。

## AFHTTPSessionManger - HTTP 请求的 6 种方法

看完初始化到设置，再回到 AFHTTPSessionManger ，它提供了 HTTP 请求的 6 种方法，分别是:

- GET: `[sessionManager  GET:parameters:success:failure:]` / `[sessionManager  GET:parameters:progress: success:failure:]`.
- HEAD: `[sessionManager  HEAD:parameters:success:failure:]`
- POST: `[sessionManager  POST:parameters:success:failure:]` / `[sessionManager  POST:parameters:progress: success:failure:]`/`[sessionManager  POST:parameters: constructingBodyWithBlock: success:failure:]`/`[sessionManager  POST:parameters:constructingBodyWithBlock:progress: success:failure:]`
- PUT: `[sessionManager  PUT:parameters:success:failure:]`
- PATCH: `[sessionManager  PATCH:parameters:success:failure:]`
- DELETE: `[sessionManager  DELETE:parameters:success:failure:]`


## AFHTTPSessionManger - 如何构建 HTTP 请求 

上面提到在 AFHTTPSessionManger 的 6 种 HTTP 请求方式，基本都使用到这一个方法来构建请求:

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

这里做了两个主要的步骤	:
- 生成一个 request 请求,里面包含有请求的方法，参数，地址等。
- 生成一个 dataTask 任务,并将结果传入 block 。



# 序列化

在上面 AFHTTPSessionManager 的初始化方法，做了请求和响应的序列化处理.现在进一步了解序列化的功能。

请求相关:

- AFURLRequestSerialization (协议)
   - AFHTTPRequestSerializer 
     - AFJSONRequestSerializer
     - AFPropertyListRequestSerializer
- AFMultipartFormData (上传下载相关协议)

响应相关:

- AFURLResponseSerialization (协议)
   - AFHTTPResponseSerializer
     - AFJSONResponseSerializer
     - AFXMLParserResponseSerializer
     - AFXMLDocumentResponseSerializer
     - AFPropertyListResponseSerializer
     - AFImageResponseSerializer
     - AFCompoundResponseSerializer
   

## 请求序列化

把参数编码为查询字符串、HTTP主体、必要时设置适当的HTTP头字段. 

### AFURLRequestSerialization

AFURLRequestSerialization 作为请求序列化的协议之一，只有一个方法:

```objc
/**
 返回一个从原始请求复制过来的参数被编码的请求。

 @param request 要进行序列化的请求
 @param parameters 要编码的参数
 @param error 试图对请求参数进行编码时发生的错误

 @return  一个序列化好的请求 
 */
- (nullable NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(nullable id)parameters
                                        error:(NSError * _Nullable __autoreleasing *)error NS_SWIFT_NOTHROW;
```

### AFHTTPRequestSerializer 

AFHTTPRequestSerializer 是一个遵守 AFURLRequestSerialization 协议的对象。

AFHTTPSessionManger 里这样用它来生成一个 NSMutableURLRequest 对象:

```objc
...
// self.requestSerializer 类型为 AFHTTPRequestSerializer<AFURLRequestSerialization>
NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
...
```

看它的初始化：

```objc
+ (instancetype)serializer {
    return [[self alloc] init];
}

- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    //设置字符串编码格式为 UTF8 ,具体支持的类型可以查看 NSStringEncoding
    self.stringEncoding = NSUTF8StringEncoding;

    //初始化 mutableHTTPRequestHeaders ，为一个可变字典
    self.mutableHTTPRequestHeaders = [NSMutableDictionary dictionary];
    //涉及 mutableHTTPRequestHeaders 修改的任务，要提交到这个 Queue , 为并发队列
    self.requestHeaderModificationQueue = dispatch_queue_create("requestHeaderModificationQueue", DISPATCH_QUEUE_CONCURRENT);

    // Accept-Language HTTP Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.4
    // 对请求头部 Accept-Language 字段的定义 
    NSMutableArray *acceptLanguagesComponents = [NSMutableArray array];
    [[NSLocale preferredLanguages] enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        float q = 1.0f - (idx * 0.1f);
        [acceptLanguagesComponents addObject:[NSString stringWithFormat:@"%@;q=%0.1g", obj, q]];
        *stop = q <= 0.5f;
    }];
    //Accept-Language: fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5
    [self setValue:[acceptLanguagesComponents componentsJoinedByString:@", "] forHTTPHeaderField:@"Accept-Language"];


    // userAgent 设置 ，User-Agent 首部包含了一个特征字符串，用来让网络协议的对端来识别发起请求的用户代理软件的应用类型、操作系统、软件开发商以及版本号。
    NSString *userAgent = nil;
#if TARGET_OS_IOS
    // User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
    userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
#elif TARGET_OS_WATCH
    // User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
    省略...
#elif defined(__MAC_OS_X_VERSION_MIN_REQUIRED)
    省略...
#endif
    if (userAgent) {
        if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) { //判断是否能被编码为 ASCII
        
            NSMutableString *mutableUserAgent = [userAgent mutableCopy];
            if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
                userAgent = mutableUserAgent;
            }
            
        }
        [self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
    }

    // HTTP Method Definitions; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html
    self.HTTPMethodsEncodingParametersInURI = [NSSet setWithObjects:@"GET", @"HEAD", @"DELETE", nil];


    //通过 AFHTTPRequestSerializerObservedKeyPaths 函数获取 AFN 监听哪些头部字段的变化, 并且提供监听方法, 在监听方法中如果有值改变, 就赋值给 mutableHTTPRequestHeaders 字典.
    self.mutableObservedChangedKeyPaths = [NSMutableSet set];
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }

    return self;
}

```
 
提供给外部调用，生成 NSMutableURLRequest 的方法代码：
 
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

    //对 AFHTTPRequestSerializer 监听的属性进行遍历
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        //如果这个属性，也包含在 self.mutableObservedChangedKeyPaths 之中的话
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            //使用 KVC 方法，给 request 设置对应的值
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }

    //对请求进行序列化
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
 ```
 
### AFHTTPRequestSerializerObservedKeyPaths() 获取观察的属性
 
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

### AFURLRequestSerialization 协议方法的实现

实现 AFURLRequestSerialization 协议里的方法，对请求参数进行序列化，在  AFHTTPRequestSerializer 里的代码为 ：

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
        if (self.queryStringSerialization) {//如果 queryStringSerialization 有值，则调用它，执行后返回序列化好的字符串赋值给 query
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            //如果存在序列化错误，则返回 nil ，并赋值给 error
            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {//没有设置过 queryStringSerialization，用默认序列化方式
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {//如果请求方法是 GET、HEAD、DELETE
        if (query && query.length > 0) {//query 存在且长度大于 0，就把 query 拼接到 url 后面
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {//其它的请求方法，
    
        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {//假如 Content-Type 值为空，设置它为 application/x-www-form-urlencoded
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        //设置请求的 body，把 query 拼接到 body 中
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```

#### 默认的参数拼接

默认方式用的 AFQueryStringFromParameters(NSDictionary *parameters) 做参数序列化，实现如下:

```objc
NSString * AFQueryStringFromParameters(NSDictionary *parameters) {
    NSMutableArray *mutablePairs = [NSMutableArray array];
    //从 AFQueryStringPairsFromDictionary 取数组，每一个都为 AFQueryStringPair 对象，
    for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
         //mutablePairs 添加执行  URLEncodedStringValue 方法得到的对象
        [mutablePairs addObject:[pair URLEncodedStringValue]];
    }
    //将 mutablePairs 数组中的元素，用 & 做间隔，组合成一个字符串
    return [mutablePairs componentsJoinedByString:@"&"]; 
}

```

里面用到了 AFQueryStringPairsFromDictionary 来组织 key/value:
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
    // 即@[@"foo", @"bar", @"bae"] ----> @[@"bae", @"bar",@"foo"]
    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];
    
    
    //根据 value 的类型，选择不同的 key 的参数形式.例如数组是用 key[],字典是用 key[nestedKey].然后一直递归调用，直到为单项元素，变为 AFQueryStringPair 对象。 
    if ([value isKindOfClass:[NSDictionary class]]) { //对 NSDictionary 类型进行序列化
        NSDictionary *dictionary = value;

        for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            id nestedValue = dictionary[nestedKey];
            if (nestedValue) {//key[nestedKey] 或者 nestedKey 为键, nestedValue 为值，递归调用自己
                [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
            }
        }
        
    } else if ([value isKindOfClass:[NSArray class]]) {//对 NSArray 类型进行序列化
        NSArray *array = value;
        for (id nestedValue in array) { // key[] 为键，nestedValue 为值，递归调用自己
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
        }
    } else if ([value isKindOfClass:[NSSet class]]) {//对 NSSet 类型进行序列化
        NSSet *set = value;
        for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {//key 为键，obj 为值，递归调用自己
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue(key, obj)];
        }
    } else {
            //对于非 NSDictionary,NSArray,NSSet 的类型，把 key，value 组织到一个 AFQueryStringPair 对象中
            [mutableQueryStringComponents addObject:[[AFQueryStringPair alloc] initWithField:key value:value]];
    }

    return mutableQueryStringComponents;
}

```


#### AFQueryStringPair

利用 AFQueryStringPair 来对参数做对应 key/value 拼接，调用 `URLEncodedStringValue` 方法，变成 HTTP 请求中的参数形式 "key=value".

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
    if (!self.value || [self.value isEqual:[NSNull null]]) {//如果 value 为空
        return AFPercentEscapedStringFromString([self.field description]);
    } else {
        return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
    }
}

```

上面都用到了 AFPercentEscapedStringFromString(NSString *string) 来处理返回的字符串，把字符串转换为百分号编码的字符串
:

```objc
NSString * AFPercentEscapedStringFromString(NSString *string) {
    //RFC 3986 states that the following characters are "reserved" characters.
    //RFC 3986声明以下字符是“保留”字符。
    //General Delimiters: ":", "#", "[", "]", "@", "?", "/"
    //Sub-Delimiters: "!", "$", "&", "'", "(", ")", "*", "+", ",", ";", "="

    static NSString * const kAFCharactersGeneralDelimitersToEncode = @":#[]@"; // does not include "?" or "/" due to RFC 3986 - Section 3.4
    static NSString * const kAFCharactersSubDelimitersToEncode = @"!$&'()*+,;=";

     
     //URLQueryAllowedCharacterSet 为字符允许集合,包含有： "#%<>[\]^`{|}
    NSMutableCharacterSet * allowedCharacterSet = [[NSCharacterSet URLQueryAllowedCharacterSet] mutableCopy];
    //从可用集合中替换删除掉字符
    [allowedCharacterSet removeCharactersInString:[kAFCharactersGeneralDelimitersToEncode stringByAppendingString:kAFCharactersSubDelimitersToEncode]];

	// FIXME: https://github.com/AFNetworking/AFNetworking/pull/3028
    // return [string stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
    //声明批处理的最大长度
    static NSUInteger const batchSize = 50;

    NSUInteger index = 0;
    NSMutableString *escaped = @"".mutableCopy;

    // 循环将string里面:#[]@!$&'()*+,;=的字符替换成%
    while (index < string.length) {
        NSUInteger length = MIN(string.length - index, batchSize);
        NSRange range = NSMakeRange(index, length);

        // To avoid breaking up character sequences such as 👴🏻👮🏽
        range = [string rangeOfComposedCharacterSequencesForRange:range];

        NSString *substring = [string substringWithRange:range];
        //指定范围内的字符做百分号编码
        NSString *encoded = [substring stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        [escaped appendString:encoded];

        index += range.length;
    }

	return escaped;
}
```

而需要转成百分号字符串的原因，也有相关的说明:
>In RFC 3986 - Section 3.4, it states that the "?" and "/" characters should not be escaped to allow
 query strings to include a URL. Therefore, all "reserved" characters with the exception of "?" and "/"
 should be percent-escaped in the query string.

在 RFC 3986 第3.4节中，它声明“?”和“/”字符不应该被转义，以允许 query strings 包含一个URL。因此，除了“?”和“/”之外，所有“保留”字符都应该在 query strings 中转义。

对于url中包含非标准url的字符时，就需要对其进行编码。

[特殊请求地址编码转换](https://blog.csdn.net/zww1984774346/article/details/51459418)

### AFMultipartFormData 

AFMultipartFormData 也是一个协议，用于 Content-Type 的值为 multipart/form-data 的请求，做文件上传。


> Content-Type ：
>通常我们都会在请求头中，指定 Content-Type 的值，来声明请求的内容类型。如果没有指定 ContentType ，会默认为 text/html。

注意到在 Content-Type 里还有个boundary,这个是用来做分隔的字符串。

TODO  基础 http 文件上传知识
[ HTTP请求 ContentType](https://imququ.com/post/four-ways-to-post-data-in-http.html)
[ContentType](http://homeway.me/2015/07/19/understand-http-about-content-type/)

#### AFStreamingMultipartFormData 

[AFStreamingMultipartFormData](https://blog.csdn.net/tsunamier/article/details/53611811)

AFStreamingMultipartFormData 遵守 AFMultipartFormData 协议，它只对对外暴露两个方法: 

```objc
@interface AFStreamingMultipartFormData : NSObject <AFMultipartFormData>
- (instancetype)initWithURLRequest:(NSMutableURLRequest *)urlRequest
                    stringEncoding:(NSStringEncoding)encoding;

- (NSMutableURLRequest *)requestByFinalizingMultipartFormData;
@end
```

初始化:

```objc

- (instancetype)initWithURLRequest:(NSMutableURLRequest *)urlRequest
                    stringEncoding:(NSStringEncoding)encoding
{
    self = [super init];
    if (!self) {
        return nil;
    }
    
    //将传入的参数赋值
    self.request = urlRequest;
    self.stringEncoding = encoding;
    
    // AFCreateMultipartFormBoundary() 创建 boundary 
    self.boundary = AFCreateMultipartFormBoundary();
    
    //对 bodyStream 根据 encoding 格式做设置
    self.bodyStream = [[AFMultipartBodyStream alloc] initWithStringEncoding:encoding];

    return self;
}

```

这里要解释一下 boundary 的概念:

> ContentType 中的 boundary ，是文件分割符，在 HTTP 中这么设置:
> 
> Content-Type: multipart/form-data; boundary=BoundaryValue
>
>而之后在 body 里，就会根据 boundary 分割的位置，一块一块的发送出去。

TODO 超过多少字节使用 boundary ? 还是分文件和内容用 boundary ?比如同时 post 传 用户名，文件 A ,文件 B.


AFCreateMultipartFormBoundary() 的方法实现为: 

```
static NSString * AFCreateMultipartFormBoundary() {
    return [NSString stringWithFormat:@"Boundary+%08X%08X", arc4random(), arc4random()];
}
```

AFMultipartBodyStream 用于设置 HTTP 请求中的 bodyStream ，方法 `requestByFinalizingMultipartFormData` 的实现为:

```objc
- (NSMutableURLRequest *)requestByFinalizingMultipartFormData {
    //bodyStream 如果为空，就直接返回 request
    if ([self.bodyStream isEmpty]) {
        return self.request;
    }

    //对 bodySteam 做初始化和最后的 boundaries 设置
    [self.bodyStream setInitialAndFinalBoundaries];
    //设置 request 的 bodyStream
    [self.request setHTTPBodyStream:self.bodyStream];
    
    // 设置 Content-Type 和 Content-Length 的值
    [self.request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", self.boundary] forHTTPHeaderField:@"Content-Type"];
    [self.request setValue:[NSString stringWithFormat:@"%llu", [self.bodyStream contentLength]] forHTTPHeaderField:@"Content-Length"];

    return self.request;
}
```


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

#### AFURLSessionManager - 为 dataTask 添加 delegate

添加代理的代码如下 :

```objc
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    //初始化一个 delegate,设置它的 manager 和完成回调
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


# 文件上传/下载

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




# HTTPS 的处理

TODO - HTTPS

## AFSecurityPolicy


# HTTPS 策略的处理




