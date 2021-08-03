---
layout: post
title: "C++解析multipart/form-data"
date: 2021-08-03 +0800
categories: C++
---

# 背景
使用`boost.beast`实现简单的HTTP服务，但是`boost.beast`没有提供对表单数据的解析，为此需要自己实现解析表单数据。
`application/x-www-form-urlencoded`的格式与URL的查询字符串格式一样，只是会被URL编码，比较容易处理
但`multipart/form-data`相对而言比较复杂

# multipart/form-data 

`multipart/form-data`主要是为了解决`application/x-www-form-urlencoded`编码格式在传输大量二进制数据或包含非ASCII字符文本时的低效问题。`multipart/form-data`的数据由多个part组成，part间通过boundary分隔符进行分割，每个part由header和content组成

`multipart/form-data`的格式大致为：
> 
    ----------------------------904587217962624105581666
    Content-Disposition: form-data; name="projectName"
    
    testProject
    ----------------------------904587217962624105581666
    Content-Disposition: form-data; name="clientName"
    
    aaa
    -----------------------------904587217962624105581666--

发送`multipart/form-data`的Http请求头中的`Content-Type`信息：
> multipart/form-data; boundary=----------------------------904587217962624105581666

更多的关于`multipart/form-data`信息可以查看[Returning Values from Forms: multipart/form-data](https://www.greenbytes.de/tech/webdav/rfc7578.pdf)

# FormItem
使用`FormItem`来表示`multipart/form-data`中的一个part，`FormItem`并不复制数据内容，只保存指向表单数据的指针，并通过保存记录本部分数据在表单数据中的起始位置`_dataStart`和数据长度`_dataLength`来表示数据，避免拷贝造成的开销。

除了数据内容外，还需要保存每部分内容的头部信息，包括`name`,`contentType`,`fileName`

```C++
class FormItem{
    /**
     * 将MultipartContentElement作为MultipartContentParse的友元类
     * 使得MultipartContentParse对MultipartContentElement具有完的控制权
     * 因为两个类是强相关的，且MultipartContentElement中的数据应由
     * MultipartContentParse进行设置
     */
    friend class FormDataParser;
    private:
        std::string _fileName;                       ///< 表单元为文件时，具有文件名
        std::string _name;                           ///< 表单元的key
        std::string _contentType;                    ///< 表单元该部分的类型
        const std::shared_ptr<std::string> _content; ///< 指request中的表单内容
        int _dataStart;                              ///< 属于本单元素的内容的起始下标
        int _dataLength;                             ///< 属于本单元素的内容的长度
    
        /**
         * MultipartContentElement对象只能由MultipartContentPars生成
         * 将构造函数的访问权设置为private，防止外部创建
         * @param name 表单元素的名
         * @param fileName 文件时fileName不为空
         * @param contentType 类型
         * @param content 指向表单数据的指针
         * @param start 本表单元素的起始位置
         * @param length 本表单元素的数据长度
         * @return MultipartContentElement对象
         */
        FormItem(const std::string name, 
            const std::string fileName, const std::stringcontentType,
            const std::shared_ptr<std::string> content, 
            const int start, const int length);
    public:
        inline std::string getFileName() const { return_fileName; }
        inline std::string getName() const { return _name; }
        inline std::string getContenType() const {return_contentType; }
        inline bool isFile() const {return !_fileName.empty(); }
        /**
         * 获取具体的内容，不返回指向原始内容的指针
         * 而是复制内容，防止外部对请求作出更改，影响到同一表单中同元素的内容
         * @return 新复制的内容的指针
         */
        std::unique_ptr<std::string>  getContent() const ;
};
```

类私有成员变量使用`_`开头，`getContent`返回的是复制的数据，而不是指向数据起始位置的指针，这是为了避免外部直接修改表单数据，但是这也造成了一定的开销，使用时可根据具体情况使用不同的返回方式。

`FormItem`如果是文件时，`_fileName`不为空，否则为空，通过`_fileName`是否为空来判断该部分是否为文件

构建函数声明为私有，防止外部构造`FormItem`对象，调用者只需要使用`FormItem`，不需要也不应该构造该对象

# FormDataParser
`multipart/form-data`的boundary在Http的Header中已经包含了，故该值可由调用者提供，`FormDataParser`只负责解析`multipart/form-data`请求的body部分的数据。
```C++
class FormDataParser{
    private:
        std::shared_ptr<std::string> _data; ///< 指向表单数据的针
        std::string _boundary;              ///< 不同元素的分割符串

        bool _lastBoundaryFound;            ///< 是否找到了最后边界
        int _pos;                           ///< 当前解析到的位置
        int _lineStart;                     ///< 当前行的起始位置
        int _lineLength;                    ///< 当前行的长度

        std::string _partName;              ///< 当前元素的名
        std::string _partFileName;          ///< 当前元素的文件名
        std::string _partContentType;       ///< 当前元素的类型
        int _partDataStart;                 ///< 当前元素的数据表单中的起始位置
        int _partDataLength;                ///< 当前元素的数据长度
    public:
        FormDataParser(const std::shared_ptr<std::string> data, 
            const int pos, const std::string boundary);
        /**
         * 调用parse函数后，才会执行对应的解析操作，
         * @return 指向由FormItem组成的vector的unique_ptr
         */
        std::unique_ptr<std::vector<FormItem>> parse();  
    private:
        /**
         * 解析表单数据的头部，即紧跟着boundary后面的一行
         */
        void parseHeaders();
        /**
         * 解析表单元素中的具体数据
         */
        void parseFormData();
        /**
         * 获取下一行的数据，
         * 在此实际上是通过更新类内部的_pos, _lineStart,_lineLength实现的
         * @return 是否成功得到下一行的数据
         */
        bool getNextLine();
        /**
         * 判断是否为边界分割行
         * @return 是边界分割行放回true，否则返回false
         */
        bool atBoundaryLine();
        /**
         * 判断是否到达表单数据的末尾
         */
        inline bool atEndOfData(){
            return _pos >= _data->size() || _lastBoundaryFound;
        }            
        std::string getDispositionValue(
            const std::string source, int pos, const std::stringname);
        /**
         * 去除字符串前后的空白字符
         * @return 去除空白字符的字符串
         */
        inline std::string& trim(std::string &s){
            if(s.empty()){ return s; }
            s.erase(0, s.find_first_not_of(" "));
            s.erase(s.find_last_not_of(" ") + 1);
            return s;
        }
};
```       

`FormDataParser`的私有成员变量可以分成三部分，第一部分为表单数据的内容，包括`_data`和`_doundary`，第二部分是在读取处理表单数据时的状态，用来记录处理表单数据需要记录的临时数据，包括`_lastBoundaryFound`,`_pos`,`_lineStart`和`_lineLength`，最后一部分是读取到一个part时保存的数据，用来构建前文提到的`FormItem`对象，包括`_partName`, `_partFileName`,`_partContentType`, `_partDataStart`, `_partDataLength`。

除了构造函数外，`FormDataParser`只包含一个`parse()`函数，因为给类的定位就是解析，不需要其他的功能。`parse()`返回一个指向`FormItem`的数组的`unique_ptr`

## parse

解析的主要步骤就是通过循环对每个part进行解析并构造`FormItem`对象存储起来，每个part又包含头部和内容。
为了避免body中在表单数据之前其他数据，在找到边界时候再开始解析表单数据，具体代码为：
```C++
std::unique_ptr<std::vector<FormItem>> FormDataParser::parse(){
    auto p = std::make_unique<std::vector<FormItem>>();
        
    //跳过空白行，直到遇到边界boundary，表示一个表单数据的开始
    while(getNextLine()){
        if(atBoundaryLine()){
            break;
        }
    }
    do{
        //处理头部
        parseHeaders();
        //头部过后如果没有数据，跳出循环
        if(atEndOfData()){ break; }
        //处理该项表单数据
        parseFormData();
        //将表单数据添加到结果数组中
        FormItem formItem(_partName, _partFileName, _partContentType, 
            _data, _partDataStart, _partDataLength);
        p->push_back(std::move(formItem));
    }while(!atEndOfData());
    
    return p;
}
```

由于`FormItem`的构造函数是`private`的，因此无法使用`emplace_back`，必须先构建好对象后再添加到`vector`中。使用`std::move`可以避免拷贝

## getNextLine
`getNextLine`用于获取表单中的下一行数据
```C++
bool FormDataParser::getNextLine(){
    int i = _pos;
    _lineStart = -1;

    while(i < _data->size()){
        //找到一行的末尾
        if(_data->at(i) == '\n'){
            _lineStart = _pos;
            _lineLength = i - _pos;
            _pos = i + 1;
            //忽略'\r'
            if(_lineLength > 0 && _data->at(i - 1) == '\r'){
                _lineLength--;
            }
            break;
        }
        //到达表单数据的末尾了
        if(++i == _data->size()){
            _lineStart = _pos;
            _lineLength = i - _pos;
            _pos = _data->size();
        }
    }

    return _lineStart >= 0;
}
```

## atBoundaryLine
判断当前读取到的行是否为边界
```C++
bool FormDataParser::atBoundaryLine(){
    int boundaryLength = _boundary.size();
    //最后的边界会多两个'-'符号
    if(boundaryLength != _lineLength && 
        boundaryLength + 2 != _lineLength){
        return false;
    }

    for(int i = 0; i < boundaryLength; ++i){
        if(_data->at(i + _lineStart) != _boundary[i]){ return false; }
    }

    if(_lineLength == boundaryLength){ return true; }
    //判断是否是最后的边界
    if(_data->at(boundaryLength + _lineStart) != '-' ||
        _data->at(boundaryLength + _lineStart + 1) != '-'){
        return false;
    }  

    //到达最后的边界
    _lastBoundaryFound = true;
    return true;
}
```
再表单数据的最后一个分隔符，会多出两个‘-’，因此需要做不同的判断
在判断两个字符串是否相等之前，先检查长度是否相等，不相等可以不用做后续的比较

## parseHeaders
处理一项表单数据的头部
```C++
void FormDataParser::parseHeaders(){
    //清除之前的数据
    _partFileName.clear();
    _partName.clear();
    _partContentType.clear();

    while(getNextLine()){
        //头部内容结束后，会有一个空白行
        if(_lineLength == 0){ break; }
        const std::string thisLine = _data->substr(_lineStart,_lineLength);

        int index = thisLine.find(':');
        if(index < 0){ continue; }

        const std::string header = thisLine.substr(0, index);
        if(header == "Content-Disposition"){
            _partName = getDispositionValue(thisLine, index + 1, "name");
            _partFileName = getDispositionValue(thisLine, index + 1, "filename");
        }else if(header == "Content-Type"){
            _partContentType = thisLine.substr(index + 1);
            trim(_partContentType);
        }
    }
}
```
处理新的头部，意味着之前的项已经处理完毕了，因此在处理之前先将之前设置了的信息清理掉
`substr`会创建出一个新的`string`对象，但是在`header`中一行的数据通过不会过程，因此此项开销是可以接受的，而且后续操作也比较方便实现，如果要进一步优化可以考虑直接使用_data进行操作，如计算`index`可以修改为：
```C++
int index = _data->find(':', _lineStart);
if(index < _lineStart || index > _lineStart + _lineLength){
    continue;
}
```
后续所有用到`thisLine`的地方都需要进行修改。由于header以及要拿到的name和filename都需要复制其值，因此在此处是否有必要优化，能取得多大的提升或许需要进一步讨论。

`multipart/form-data`每一部分的头部中可能包含多行数据，每一行数据表示不同的含义，通过`Content-Disposition`和`Content-Type`来区分并通过`getDispositionValue`获取其内容

## getDispositionValue
```C++
std::string FormDataParser::getDispositionValue(
    const std::string source, int pos, const std::string name){
    //头部内容：Content-Disposition: form-data; name="projectName"
    //构建模式串
    std::string pattern = " " + name + "=";
    int i = source.find(pattern, pos);
    //更换格式继续查找位置
    if(i < 0){
        pattern = ";" + name + "=";
        i = source.find(pattern, pos);
    }
    if(i < 0){
        pattern = name + "=";
        i = source.find(pattern, pos);
    }
    //尝试了可能的字符串，还没有找到，返回空字符串        
    if(i < 0){ return std::string(); }

    i += pattern.size();
    if(source[i] =='\"'){
        ++i;
        int j = source.find('\"', i);
        if(j < 0 || i == j){ return std::string();}
        return source.substr(i, j - i);
    }else{
        int j = source.find(";", i);
        if(j < 0){ j = source.size(); }
        auto value = source.substr(i, j - i);
        //去掉前后的空白字符
        return trim(value);
    }
}
```

## parseFormData
处理表单的实际数据部分
```C++
void FormDataParser::parseFormData(){
    _partDataStart = _pos;
    _partDataLength = -1;

    while(getNextLine()){
        if(atBoundaryLine()){
            //内容数据位于分解线前一行
            int indexOfEnd = _lineStart - 1;
            if(_data->at(indexOfEnd) == '\n'){ indexOfEnd--; }
            if(_data->at(indexOfEnd) == '\r'){ indexOfEnd--; }
            _partDataLength = indexOfEnd - _partDataStart + 1;
            break;
        }
    }
}
```

在遇到新的分割符时说明本部分的数据结束，但是在数据与分割行之间有换行符，可能是`'\n'`，也可能是`'\r\n'`，需要根据不同的情况减去不同的长度，从而得出当前表单项的数据的长度
RFC7578要求`boundary`不能在数据中出现
> 4.1. "Boundary" Parameter of multipart/form-data
As with other multipart types, the parts are delimited with a boundary delimiter, constructed using CRLF, "--", and the value of the "boundary" parameter. The boundary is supplied as a "boundary" parameter to the multipart/form-data type. As noted in Section 5.1 of [RFC2046], the boundary delimiter MUST NOT appear inside any of the encapsulated parts, and it is often necessary to enclose the "boundary" parameter values in quotes in the Content-Type header field

# 总结
文本介绍了如何使用C++解析`multipart/form-data`。
`multipart/form-data`结构也是比较清晰的，因此解析起来不算太麻烦。