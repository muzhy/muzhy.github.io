+++
isCJKLanguage = true
title = "在C++中通过libcurl发起http请求"
description = "在C++中通过libcurl发起http请求"
keywords = ["C++", "HTTP", "curl"]
date = 2023-11-01T14:51:39+08:00
authors = ["木章永"]
tags = ["C++", "HTTP"]
categories = ["C++", "HTTP"]
cover = "/images/curl.webp"
+++

需要在C++程序中通过HTTPS调用第三方接口，之前的库中虽然也有发起HTTP请求的，但是没有使用HTTPS，并且需要在请求头中添加Signature进行鉴权，简单搜索了下，感觉通过调用`libcurl`来实现该请求会比较简单，并且`libcurl`的可靠性大概要比自己实现或者其他的库更好

使用`libcurl`实现的发起http请求时同步的，但是借助C++ 11提供的`std::async`以及更新版本提供的异步工具，可以很方便的修改为异步的。

# 整体设计
实现两个类：`THttpRequest`和`THttpResponse`分别表示HTTP请求和响应。

当需要发起请求时，创建一个`THttpRequest`对象并设置需要请求的各项请求参数，如url, method, body等。

创建完对象并设置好请求参数后，需要提供一个函数用于调用`libcurl`发起http请求，这里有两个选择
1. 在`THttpRequest`中提供一个成员函数`THttpResponse request()`
2. 提供一个外部函数，接收一个`THttpRequest`对象并返回一个`THttpReponse`对象：`THttpResponse sendHttpRequest(const THttpRequest& request)`

两种方式并没有实质上的差异，成员函数也可以通过调用外部函数实现。

选择第二种方式，外部函数的方式实现

除了`THttpRequest`和`THttpResponse`，由于需要在请求头中添加鉴权，增加一个抽象基类`THttpHeaderPartGenerater`用于表示需要添加到请求头的信息，在设置`THttpRequest`对象时通过创建`THttpHeaderPartGenerater`对象并添加到`THttpRequest`中，以便在发起请求时生成所需的头部信息

`THttpHeaderPartGenerater`设计为一个抽象基类，声明一个抽象函数
``` C++
virtual void generate(const THttpRequest& request, std::map<tstring, tstring>& mapHeaderParams) = 0;
```
用于生成相应的请求头信息。

需要添加自定义的请求头信息时可以继承该类，在发起请求时创建一个实例并添加到`THttpRequest`中

# THttpRequest
```C++
typedef std::string					tstring ;

class THttpRequest
{
public:
    // http need host and uri, so require these param in construct
    // default use http and GET method,
    THttpRequest(const tstring& strHost, const tstring& strUri, 
        const bool useHttps = false, const E_HTTP_METHOD method = HTTP_METHOD_GET) :
        m_strHost(strHost), m_strUri(strUri), m_bUseHttps(useHttps), m_eMethod(method)
    {}

    ~THttpRequest(){}

    static constexpr char* HTTP_CONTENT_TYPE_JSON_STR = "application/json";
    static constexpr char* HTTP_CONTENT_TYPE_STR = "Content-Type";

    inline void setMethod(const E_HTTP_METHOD method) { m_eMethod = method; }
    inline E_HTTP_METHOD getMethod() const { return m_eMethod; }

    inline void setHost(const tstring& strHost) { m_strHost = strHost; }
    inline tstring getHost() const { return m_strHost; }

    inline void setUri(const tstring& strUri) { m_strUri = strUri; }
    inline tstring getUri() const { return m_strUri; }

    inline void useHttps() { m_bUseHttps = true; }
    inline void useHttp() { m_bUseHttps = false; }
    inline bool isUseHttps() const { return m_bUseHttps; }

    inline tstring getFullUrl() const
    {
        return (m_bUseHttps ? "https" : "http") + tstring("://") + m_strHost + m_strUri;
    }
    // simple header param set buy setHeaderParam function, 
    // complex header param recommend use THttpHeaderPartGenerater and add by addHeaderPartGenerater
    void setHeaderParam(const tstring& key, const tstring& value)
    {
        m_mapHeaderParam[key] = value;
    }
    void setContentType(const tstring& contentType)
    {
        m_mapHeaderParam[THttpRequest::HTTP_CONTENT_TYPE_STR] = contentType;
    }
    inline tstring getContentType() const
    {
        auto it =  m_mapHeaderParam.find(THttpRequest::HTTP_CONTENT_TYPE_STR);
        if(it == m_mapHeaderParam.end())
        {
            return "";
        }
        else
        {
            return it->second;
        }
    }

    std::map<tstring, tstring> generateFullHeaderParam() const ;

    inline void setBodyString(const tstring& strBody) { m_strBody = strBody; }
    inline tstring getBodyString() const { return m_strBody; }
    inline int getContentLength() const { return m_strBody.size(); }

    inline void addHeaderPartGenerater(std::shared_ptr<THttpHeaderPartGenerater> ptrGenerater)
    {
        m_vecHeaderGenerater.push_back(ptrGenerater);
    }

private:
    tstring m_strHost;
    tstring m_strUri;
    bool m_bUseHttps;
    E_HTTP_METHOD m_eMethod;
    std::map<tstring, tstring> m_vecHeaderGenerater;
    tstring m_strBody;  

    std::vector<std::shared_ptr<THttpHeaderPartGenerater>> m_vecHeaderGenerater;
};
```

对于简单的头部参数设置可以直接通过调用`setHeaderParam`函数进行设置，如`Content-Type`。只有需要较为负责的头部信息生成方式才需要通过`THttpHeaderPartGenerater`来创建。

`THttpRequest`提供`addHeaderPartGenerater`函数来添加所需生成的头部信息生成器，通过类成员变量`m_vecHeaderGenerater`存储，在发起请求的时候才获取这些生成器来生成

`THttpRequest`类内部通过`m_vecHeaderGenerater`记录头部信息，在发起请求时通过该map构造请求头。


# 自定义头部信息生成器
作为参考，以下给出了几个生成器
## Content Digest生成器
`Content-Digest`在很多需要校验请求内容的时候都需要携带，在添加Signature的时候也需要计算该信息

### 声明
```C++
enum E_HASH_ALGO
{
    HASH_ALGO_SHA256,
};

class TContentDigestGenerater : public THttpHeaderPartGenerater
{
public:
    // Content-Digest can chose hash algorithm 
    TContentDigestGenerater(E_HASH_ALGO hashAlgo) : m_eHashAlgo(hashAlgo) {}
    virtual ~TContentDigestGenerater(){}

    virtual void generate(const THttpRequest& request, std::map<tstring, tstring>& mapHeaderParams);

    static constexpr char* HEADER_KEY_CONTENT_DIGEST = "Content-Digest";
private:
    E_HASH_ALGO m_eHashAlgo;
};
```

`Content-Digest`能使用多种不同的哈希算法，通过定义一个枚举来表示支持的哈希算法，在这里为了简单起见仅使用`SHA-256`

### 实现
```C++
#include <openssl/sha.h>
#include <openssl/evp.h>
#include <openssl/bio.h>

tstring toBase64String(unsigned char* data, unsigned int len)
{
    BIO *b64 = BIO_new(BIO_f_base64());
    BIO_set_flags(b64, BIO_FLAGS_BASE64_NO_NL);
    BIO *bmem = BIO_new(BIO_s_mem());
    b64 = BIO_push(b64, bmem);
    BIO_write(b64, data, len);
    BIO_flush(b64);
    BUF_MEM* bptr;
    BIO_get_mem_ptr(b64, &bptr);
    tstring result(bptr->data, bptr->length);
    BIO_free_all(b64);
    
    return result;
}

tstring getHashAlgoName(const E_HASH_ALGO hashAlgo)
{
    switch (hashAlgo)
    {
    case HASH_ALGO_SHA256:
        return "sha-256";
    default:
        return "";
    }
}

void TContentDigestGenerater::generate(const THttpRequest& request, std::map<tstring, tstring>& mapHeaderParams)
{
    auto strBody = request.getBodyString();

    std::shared_ptr<EVP_MD_CTX> ptrCtx(EVP_MD_CTX_new(), [](EVP_MD_CTX* ctx){ EVP_MD_CTX_free(ctx); });
    switch (m_eHashAlgo)
    {
    case HASH_ALGO_SHA256:
        EVP_DigestInit_ex(ptrCtx.get(), EVP_sha256(), NULL);
        break;
    default:
        EVP_DigestInit_ex(ptrCtx.get(), EVP_sha256(), NULL);
        break;
    }    

    EVP_DigestUpdate(ptrCtx.get(), strBody.c_str(), strBody.size());
    unsigned char md[EVP_MAX_MD_SIZE];
    unsigned int len = 0;
    EVP_DigestFinal_ex(ptrCtx.get(), md, &len);
    // in order to transmission in http, trans binary to base64 string
    auto digest = toBase64String(md, len);
    
    mapHeaderParams[TContentDigestGenerater::HEADER_KEY_CONTENT_DIGEST] = 
        std::format("{}=:{}:", getHashAlgoName(m_eHashAlgo), digest);
}
```

使用`openssl`提供的哈希算法对body计算哈希值，为了在http中传输，将计算的结果转换为base64字符串并添加到头部信息中。

`std::format`是C++20加入的字符串格式化函数

## TSignatureGenerater
鉴权方式参考RFC 8941，在请求头中生成Signature，该Siganture需要依赖于Content-Digest
### 声明
```C++

enum E_SIGNATURE_PARAM
{
    SIGNATURE_PARAM_METHOD,
    SIGNATURE_PARAM_QEUERY,
    SIGNATURE_PARAM_CONTENT_TYPE,
    SIGNATURE_PARAM_CONTENT_LEN,
    SIGNATURE_PARAM_CONTENT_DIGEST
};

// 在请求头构造Signature, 参考RFC 8941
class TSignatureGenerater : public THttpHeaderPartGenerater
{
public:
    // RFC 8941 设计得比较灵活，这里仅实现使用hmac算法生成的Signature, 需要appcode和secretKey
    // 若要设计为支持多种算法，可以考虑设计一个类似THttpHeaderPartGenerater的算法计算类，
    TSignatureGenerater(const tstring& appcode, const tstring& secretKey, const tstring& strSignatureName) : 
        m_strAppcode(appcode), m_strSecretKey(secretKey), m_strSignatureName(strSignatureName) {}
    virtual ~TSignatureGenerater() {}

    virtual void generate(const THttpRequest& request, std::map<tstring, tstring>& mapHeaderParams);

    inline void setSignatureName(const tstring& strSignatureName)
    {
        m_strSignatureName = strSignatureName;
    }

    static constexpr char* HEADER_KEY_SIGNATURE = "Signature";
    static constexpr char* HEADER_KEY_SIGATURE_INPUT = "Signature-Input";
private:
    // 生成Signature的固定参数以及固定的哈希算法
    static constexpr char* BASE_SIGNATURE_PARAMS = "(\"@method\" \"@query\" \"content-type\" \"content-length\" \"content-digest\");alg=\"hmac-sha256\";";

    tstring m_strAppcode;
    tstring m_strSecretKey;   
    tstring m_strSignatureName; 

    std::vector<E_SIGNATURE_PARAM> m_vecSignatureParam;
};
```

### 实现
```C++
#include <random>

tstring generateASCIIRamdonStr(const unsigned int len)
{
    tstring str("0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz");
    while(str.size() < len)
    {
        str += str;
    }
    std::random_device rd;
    std::mt19937 generator(rd());
    shuffle(str.begin(), str.end(), generator);
    return str.substr(0, len);  
}

void TSignatureGenerater::generate(const THttpRequest& request, std::map<tstring, tstring>& mapHeaderParams)
{
    auto now = std::chrono::system_clock::now();
    auto nTimestamp = std::chrono::duration_cast<std::chrono::seconds>(now.time_since_epoch()).count();

    tstring signatureParams = std::format("{}created={};keyid=\"{}\";nonce=\"{}\";tag=\"api1\"",
        tstring(TSignatureGenerater::BASE_SIGNATURE_PARAMS), std::to_string(nTimestamp)
        , m_strAppcode, generateASCIIRamdonStr(32));
    
    tstring signatureInput = m_strSignatureName + "=" + signatureParams;
    // signatureBase需要根据设置的参数构造，这里采用固定的参数简化实现
    tstring signatureBase = "\"@method\": " + getHttpMethodString(request.getMethod()) + "\n";
    // 此处假定没有url参数
    signatureBase += "\"@query\": ?\n";
    signatureBase += "\"content-type\": " + request.getContentType() + "\n";
    signatureBase += "\"content-length\": " + std::to_string(request.getContentLength()) + "\n";
    // 获取Content-Digest
    auto contentDigestIt = mapHeaderParams.find(TContentDigestGenerater::HEADER_KEY_CONTENT_DIGEST);
    if(contentDigestIt == mapHeaderParams.end())
    {
        throw HttpHelperException("gengerate signature, but can't get content digest");
    }
    signatureBase += "\"content-digest\": " + contentDigestIt->second + "\n";
    signatureBase += "\"@signature-params\": " + signatureParams;

    // 使用hmac-256计算signature
    unsigned char signature[EVP_MAX_MD_SIZE];
    unsigned int signatureLen = 0;
    auto decodeSecretKey = TFuncHelper::decodeBase64(m_strSecretKey);
    HMAC(EVP_sha256(), (unsigned char*)decodeSecretKey.c_str(), decodeSecretKey.size(), 
        (unsigned char*)signatureBase.c_str(), signatureBase.size(), signature, &signatureLen);
    auto signatureBase64 = toBase64String(signature, signatureLen);

    mapHeaderParams[TSignatureGenerater::HEADER_KEY_SIGATURE_INPUT] = signatureInput;
    mapHeaderParams[TSignatureGenerater::HEADER_KEY_SIGNATURE] = 
        "recall=:" + signatureBase64 + ":";

    return ;
}
```

# THttpResponse

```C++
class THttpResponse
{
public:
    THttpResponse(){}
    ~THttpResponse() {}

    TResponseBodyData getParsedBodyData() throw(HttpHelperException)
    {
        return TResponseBodyData(m_strBody);
    }

    long m_lHttpCode;
    tstring m_strHeader;
    tstring m_strBody;
};
```
`THttpResponse`很简单，因为对于body的处理需要根据业务在上层进行处理，而对于当前的需求而言，除了返回的状态码，并不需要其他功能，因此在这里不做太多复杂的封装


# 发起请求

```C++
THttpResponse sendHttpRequest(const THttpRequest& request) throw (HttpHelperException)
{
    CURL* curl = curl_easy_init();
    if(NULL == curl)
    {
        throw HttpHelperException("can't init curl");
    }

    auto curlDeleter = [](CURL* curl){
        curl_easy_cleanup(curl);
    };
    std::shared_ptr<CURL> curlPtr(curl, curlDeleter);

    tstring fullUrl = request.getFullUrl();

    curl_easy_setopt(curl, CURLOPT_URL, fullUrl.c_str());
    switch (request.getMethod())
    {
    case HTTP_METHOD_POST:
        curl_easy_setopt(curl, CURLOPT_POST, 1);
        break;
    default:
        // not set is GET method
        break;
    }
    // 使用https时不检查证书有效性
    if(request.isUseHttps())
    {
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 0L);
    }

    auto mapHeaderParams = request.generateFullHeaderParam();
    struct curl_slist *hs=NULL;   
    // for(auto& it : request.getHeaderParam())
    for(auto  &it : mapHeaderParams)
    {
        tstring line = it.first + ": " + it.second;
        hs = curl_slist_append(hs, line.c_str());
    }
    curl_easy_setopt(curl, CURLOPT_HTTPHEADER, hs);
    auto hsDeleter = [](curl_slist *hs){
        curl_slist_free_all(hs);
    };
    std::shared_ptr<curl_slist> hsStr(hs, hsDeleter);

    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, request.getBodyString().c_str());

    THttpResponse response;    
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, writeToString);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, (void*)&response.m_strBody);
    curl_easy_setopt(curl, CURLOPT_HEADERFUNCTION, writeToString);
    curl_easy_setopt(curl, CURLOPT_HEADERDATA, (void*)&response.m_strHeader); 

    CURLcode curlResCode = curl_easy_perform(curl);
    if(curlResCode != CURLE_OK)
    {
        throw HttpHelperException(std::format("execute http request failed : [{}}]{}", 
            curlResCode, curl_easy_strerror(curlResCode) ));
    }

    curl_easy_getinfo(curl, CURLINFO_RESPONSE_CODE, &response.m_lHttpCode);

    return response;
}
```

到了发起请求的时候才需要调用`libcurl`，前面的都只是为了生成请求的数据。

# 使用方式：
```C++
tstring body = "{\"key\" : \"Hello World\"}";
tstring host = "localhost";
tstring url = "/hello";

THttpRequest req(host, url, needSignature, HTTP_METHOD_POST);
req.setBodyString(body);
req.setContentType(THttpRequest::HTTP_CONTENT_TYPE_JSON_STR);

auto contentDigestGenerater = std::make_shared<TContentDigestGenerater>(HASH_ALGO_SHA256);
req.addHeaderPartGenerater(contentDigestGenerater);
auto signtureGenerater = std::make_shared<TSignatureGenerater>(appcode, securityKey);
req.addHeaderPartGenerater(signtureGenerater);

auto resp = sendHttpRequest(req);
```


