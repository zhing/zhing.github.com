---
layout: post
title: "AFNetworking使用总结"
description: ""
category: ios
tags: [ios]
---
###AFNetworking使用总结

IOS开源网络库AFNetworking已经成为了IOS程序开发的首选、亦可以说是必备，无数IOS
的“先哲”们撰文称赞此库良好的设计和功能的强大，以致后来的开发者在项目中都不会去
考虑其它的网络库，而直接选择AFNetworking。这里就来总结一下使用它的一般程式，在
总结过程中学习和成长。

##HttpClient

我们在使用AFHTTPSessionManager的时候，一般均会对其进行封装，以满足App的各种要求。
所以这里选择对其进行扩展，设计如下：

    @interface LNHttpClient : AFHTTPSessionManager

    + (instancetype)sharedClient;
    + (void)setTimeout:(NSTimeInterval)timeout;
    + (void)setResponseType:(LNHttpResponseType)type;
    - (void)setHttpHeader;

    @end
    
该继承类的实现需要注意如下几点：

* 继承AFHTTPSessionManager免不了对initWithBaseURL的覆写，并在其中注册一些通知，用于
检测用户的登陆和登出，以便Client做相应的处理。
* setHttpHeader可以设置Http头部，比如token、userId等等。
* 中间两个方法使得开发者可以控制每一次请求的timeout和responseType。

##APIService

APIService是所有网络请求的入口，所有Service的网络调用均使用该类来完成，我们项目中
使用proto-buf来作为数据交换的类型，其设计力求简介：

    typedef void (^APISuccessHandler)(id responseObject);
    typedef void (^APIFailureHandler)(NSInteger code, NSString *msg);

    @interface APIService : NSObject


    + (NSURLSessionTask *)POST:(NSString *)relativePath
                 protobuf:(NSData *)proto
               modelClass:(Class)modelClass
                  success:(APISuccessHandler)success
                  failure:(APIFailureHandler)failure;


    + (NSURLSessionTask *)GET:(NSString *)relativePath
                 protobuf:(NSData *)proto
               modelClass:(Class)modelClass
                  success:(APISuccessHandler)success
                  failure:(APIFailureHandler)failure;

该类的设计是对于AFHTTPSessionManager的封装，是所有Service类的基类。实现要点：

* 定义了两个block，分别用来处理成功和失败的调用。
* modelClass用来解析ContentType的数据，此处是proto-buf。
* 此类派生的各个Service来处理不同的业务场景。

##AFHTTPRequestSerializer覆写

在客户端发送请求时，我们有时需要设置request的content-Type，以便于服务端能够根据
content-Type来处理不同格式的数据，比如AFNetworking中自带的AFJSONRequestSerializer,
就能够把请求的数据转化为JSON格式，并且把content-Type设置为application/json。这里
我们的请求数据格式为proto-buf，而AF库并没有给我们提供相关的默认实现，这时候就需要
我们自己来实现AFProtoRequestSerializer。

    - (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
    {
    NSParameterAssert(request);

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        return [super requestBySerializingRequest:request withParameters:parameters error:error];
    }

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    if (parameters) {
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
        }

        [mutableRequest setHTTPBody:[NSJSONSerialization dataWithJSONObject:parameters options:self.writingOptions error:error]];
    }

    return mutableRequest;
    }

上面的代码是AFJSONRequestSerializer的主要覆写方法。同理我们只需要仿照这个例子来
实现AFProtoRequestSerializer即可。

##URL缓存

说起HTTP请求，就不得不聊到缓存，每次去请求相同的URL的数据显然是不划算的，所以将
每次URL请求的数据缓存起来，以后当有相同的URL请求时，直接使用缓存数据即可。使用
缓存一般有两种选择。

* NSURLCache
    
    系统提供的默认缓存，使用该方式可以减少开发的难度，但是在使用过程中需要注意的
是
    * 该缓存只能用在GET请求上，并不支持Post。
    * 缓存方式尽量选择NSURLRequestReturnCacheDataDontLoad，如果有缓存直接返回数据
    如果没有缓存则不发送请求，返回nil，我们手工来再发一次请求。这样做可以规避一
    些苹果实现缓存的坑。

* **URLCache

    自己实现的缓存，我们只需要扩展NSURLCache即可，使用扩展的cache来代替原生的实例。
    这样我们就可以人为控制缓存的URL范围和数据存储了，简单实现如下：
    
    ```
    @implementation LNURLCache

    - (void)storeCachedResponse:(NSCachedURLResponse *)cachedResponse forRequest:(NSURLRequest *)request {
        if ([self shouldManuallyCacheRequest:request]) {
            [[LNCache globalCache] setObject:cachedResponse forKey:request.URL.absoluteString withTimeoutInterval:kTimeOneYear];
        } else {
            [super storeCachedResponse:cachedResponse forRequest:request];
        }
    }

    - (NSCachedURLResponse *)cachedResponseForRequest:(NSURLRequest *)request {
        if ([self shouldManuallyCacheRequest:request]) {
            return (NSCachedURLResponse *)[[LNCache globalCache] objectForKey:request.URL.absoluteString];
        } else {
            return [super cachedResponseForRequest:request];
        }
    }

    - (BOOL)shouldManuallyCacheRequest:(NSURLRequest *)request {
        return [request.URL.host hasSuffix:kCDNHostName];
    }
    
    @end
```

##总结

通过以上讲解，相信你可以从容地处理好网络请求模块的设计。



































  
  
  
  
  
  
  









  
  
  
  
  
  
  

    
    
    
    
    


    

    































  
  
  
  
  
  
  

  
  
  
  
  
  
  
  

