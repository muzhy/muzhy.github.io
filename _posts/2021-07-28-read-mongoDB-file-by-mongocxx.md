---
layout: post
title: "基于mongocxx实现对MongoDB存储的文件的随机读取"
date: 2021-07-28 +0800
categories: C++
---

# 背景
最近所开发的项目使用MongoDB的GridFS保存文件，且文件大部分均大于500M，在程序运行过程中需要读取文件数据进行处理。但是MongoDB提供的mongocxx只能将整个文件下载下来或者暂时保存在内存中。每次处理数据都需要下载到本地是比较耗时的，且每次处理完还需要删除临时文件。
GridFS是将一个大文件分成多个chunk进行保存的，因此考虑每次进下载文件的一个chunk到内存中。

# 总体设计
为了对读取MongoDB中的文件与读取本地文件的接口统一起来，增加了一个抽象基类，对本地文件的读取也通过一个实现了该基类的类来进行。
整体有三个类，一个用于处理本地文件，一个用于处理MongoDB存储的文件还有一个定义了接口的基类.

## 基类 fileReader
```C++
// fileReader
    class FileReader{
        public:
            /**
             * 禁止拷贝操作， 复制一个文件操作对象的副本是没有意义的
             * 移动操作不需要禁止
             */
            FileReader(const FileReader&) = delete;
            FileReader& operator=(const FileReader&) = delete;
            FileReader();
            /**
             * 析构函数 当用户没有显式调用closeFile关闭文件时
             * 析构函数需要负责关闭文件，即做最后的一道保险
             */
            virtual ~FileReader();

            /**
             * 打开文件
             * 文件由构造函数指定
             * @return 成功打开文件返回true，否则返回false
             */
            virtual void openFile() = 0;
            /**
             * @return 返回文件是否已打开
             **/
            virtual const bool isOpen() const  = 0;
            /**
             * 关闭已打开的文件
             * @return 文件未打开或成功关闭文件返回true
             */
            virtual const bool closeFile() = 0;
            /**
             * 从文件中读取一行数据，并以字符串常量返回
             * 遇到"\n"时认为一行
             * @return 用字符串表示的该函数数据
             */
            virtual const std::optional<std::string> getLine() = 0;
            virtual const bool getLine(char *s, const size_t n) = 0;
            
            /**
             * 从文件的当前位置读取n个字节的数据存储到buff中
             * 并且读取文件的位置会向后移动n个字节
             * @param buff  用于存放读取到的数据的字节数组，长度必须不小于n指定的大小
             * @param n     指定要读取的字节数
             * @return      读取失败时返回0，否则返回读到的字节数
             */
            virtual const size_t read(char* buff, const size_t n) = 0;

            /**
             * @return 文件的字节数
             */
            virtual const int getFileSize() const = 0;
            /**
             * @return 用字符串表示的文件类型，无法识别出文件类型时返回UNKNOW_TYPE.
             * UNKNOW_TYPE为静态字符串常量
             */
            virtual const std::string getFileType() const = 0;

            /**
             * @return 文件名
             */
            virtual const std::string getFileName() const = 0;

            /**
             * 设置要从输入流中读取的下一个字节数据的位置
             * @param   offset      根据dir设置的模式，从指定位置的偏移量
             * @param   dir         使用ios_base::seekdir作为类型，含义与标准输入输出一致
             * @return 设置成功时返回true，设置的位置非法时返回false
             **/
            virtual bool seekg(const size_t offset, std::ios_base::seekdir dir) = 0;   
            /**
             * @return 返回当前在文件中的位置
             **/
            virtual const int tellg() = 0;
            /**
             * @return 到达文件末尾时返回true
             */
            virtual const bool eof() = 0;
        protected:
            static constexpr char* UNKNOW_TYPE = (char *)"unknow";
    };
```
接口参考`std::fstream`，由于MongoDB保存的文件不支持直接修改，所以不提供写的接口。

由于对本地文件的操作就只是做一层简单的转发，在此不展开

## MongoFileReader

接口不在此展开，仅展开部分头文件代码
```C++
    class MongoFileReader: public FileReader{
    public:
        explicit MongoFileReader(std::string_view dbName, 
                         std::string_view bucketName,
                         std::string_view fileId);
        ~MongoFileReader();

    public:
        //与接口一致，不展开

    private:
        std::optional<bsoncxx::document::value>
        getChunkDoc_(error::PCError &e, const int n);

    private:
        static const std::string FILES_SUFFIX;
        static const std::string CHUNKS_SUFFIX;
    private:
        std::string dbName_;
        std::string bucketName_;
        //文件在数据库中的id(ObjectId)
        std::string fileId_;
        bool isOpen_;

        int64_t fileSize_;        
        std::string fileName_;   
        mutable std::string fileType_; 

        bsoncxx::document::view metaDataDoc_;

        /**
         * 缓冲区大小由读取的文件的chunkSize来决定
         * 因此无法使用固定大小的字节数据来作为缓冲区
         * 使用unqiue_ptr来存储指向缓冲区的指针
         * 缓冲区中存放一个chunck的数据
         * 除了文件的最后一个块之外，所有的块大小均为chunckSize_
         */
        int32_t chunkSize_;
        const int lineBufferSize_;

        std::unique_ptr<uint8_t[]> buffPtr_;     
        //chunckNo为当前chunck在此文件中的位置
        int currChunkNo_;      
        //当前读取的数据在文件中的位置
        size_t currPos_;
        uint32_t currChunkSize_;

        int chunksCount_;
    };
```

# 实现
## openFile
MongoFileReader在打开文件时需要获取到文件的一些相关信息并进行保存
```C++
    void MongoFileReader::openFile(){
        if(isOpen_){ return; }
        //获取bucketName.files的文档信息
        error::PCError err;
        auto doc = MongoStorage::getDoc(err, dbName_, 
            bucketName_ + FILES_SUFFIX, fileId_);
        if(err.type != error::ERROR_TYPE::SUCCESS){
            isOpen_ = false; 
            return;
        }
        //获取文件信息
        mongoHelper::getInt64(doc, "length", fileSize_);
        mongoHelper::getInt32(doc, "chunkSize", chunkSize_);
        mongoHelper::getString(doc, "filename", fileName_);
        mongoHelper::getDoc(doc, "metadata", metaDataDoc_);

        //计算文件应该有多少个chunks
        chunksCount_ = (fileSize_ + chunkSize_ - 1) / chunkSize_;

        //加载第一个chunk的数据
        currPos_ = 0;
        currChunkNo_ = 0;

        if(auto chunkDoc = getChunkDoc_(err, currChunkNo_); 
           chunkDoc.has_value()){
            isOpen_ = mongoHelper::getBinData(chunkDoc.value(), 
                                              "data", buffPtr_, currChunkSize_);
        }

        return ;
    }
```
其中`MongoStorage::getDoc`是封装的一个辅助函数，作用是从MongoDB中获取一个doc。MongoDB中的文件分成两部分，一部分是xxx.files，记录文件的文件名，大小，metadata等信息，另一部分是xxx.chunks，保存文件的内容。
在获取到文件的基本信息后，获取文件的第一个chunk并保存到buffer中。

## getChunkDoc_
getChunkDoc_用来获取文件的一个chunk，n为xxx.chunks中每个文档的n一致
```C++
    optional<bsoncxx::document::value> 
    MongoFileReader::getChunkDoc_(error::PCError &err, const int n){
        using bsoncxx::builder::basic::kvp;
        using bsoncxx::builder::basic::make_document;

        bsoncxx::builder::basic::document condition;
        try{
            condition.append(kvp("files_id", bsoncxx::oid(this->fileId_)), 
                         kvp("n", n));
        
            auto chunks = MongoStorage::getDocs(err, dbName_, 
                                    bucketName_ + CHUNKS_SUFFIX,
                                    condition);
            if(err.type != error::ERROR_TYPE::SUCCESS || chunks.size() != 1){
                return nullopt;    
            }
            return chunks[0];
        }catch(bsoncxx::exception &e){
            return nullopt;
        }
    }
```

## read
read函数的参数与功能与标准库中的read基本一致
```C++
    const size_t MongoFileReader::read(char *buff, const size_t n){
        auto posInChunk = currPos_ - (chunkSize_ * currChunkNo_);
        if(currChunkSize_ - posInChunk >= n){
            //当前的缓冲区中的数据足够所需的
            memcpy(buff, buffPtr_.get() + posInChunk, n);
            currPos_ += n;
            return n;
        }
        //当前的缓冲区中的数据不够，但是已经是最后的chunk了
        if(currChunkNo_ + 1 == chunksCount_){
            size_t lack = currChunkSize_ - posInChunk;
            memcpy(buff, buffPtr_.get() + posInChunk, lack);
            return lack;
        }
        //当前缓冲区中的数据不够，但还有后续的chunk       
        uint32_t copyed = currChunkSize_ - posInChunk;
        memcpy(buff, buffPtr_.get() + posInChunk, copyed);

        auto nextChunkNo =  currChunkNo_ + 1;
        uint32_t nextChunkSize = 0;
        unique_ptr<uint8_t[]> nextBuff;
        error::PCError err;

        while(copyed < n && nextChunkNo < chunksCount_){
            if(auto chunkDoc = getChunkDoc_(err, nextChunkNo); 
               chunkDoc.has_value()){
                DAO::mongoHelper::getBinData(chunkDoc.value(), 
                                            "data", nextBuff, nextChunkSize);
            }else{
                throw std::runtime_error("cann't read next chunk");
            }
            
            auto nextCopyBytes = min(nextChunkSize, (uint32_t)n - copyed);
            memcpy(buff + copyed, nextBuff.get(), nextCopyBytes);
            copyed += nextCopyBytes;
        }
        //没有发生异常，完成拷贝，更新数据
        currPos_ += copyed;
        currChunkNo_ = nextChunkNo;
        currChunkSize_ = nextChunkSize;
        buffPtr_ = move(nextBuff);

        return copyed;
    }
```    

## getLine(char *s, const size_t n)
获取一行数据，实现思路与`read`类似
```C++
    const bool MongoFileReader::getLine(char *s, const size_t n){
        if(!isOpen()){
            throw runtime_error("not open file");
        }
        if(currPos_ >= fileSize_){
            return false;
        }
        auto posInChunk = currPos_ - (chunkSize_ * currChunkNo_);
        assert(posInChunk >= 0 );
        
        std::unique_ptr<uint8_t[]> buffUp;      //存放当前缓冲块的
        uint8_t *buffP = buffPtr_.get();        //指向当前读取的缓冲块的指针
        auto currChunkNo = currChunkNo_;        //当前读取的数据的chunk
        auto currChunkSize = currChunkSize_;    // 当前读取的块的大小
        error::PCError err;

        
        for(size_t i = 0; i < n; ){            
            if(posInChunk < currChunkSize){
                s[i] = buffP[posInChunk++];
                //读到一行
                if(s[i] == '\n' || s[i] == '\0'){
                    break;
                }
                ++i;                
            }else if(currChunkNo + 1 == chunksCount_){
                //到达文件末尾
                s[i] = '\0';
                break;
            }else{
                //读取下一个块
                currChunkNo += 1;
                posInChunk = 0;
                if(auto chunkDoc = getChunkDoc_(err, currChunkNo); 
                    chunkDoc.has_value()){
                    DAO::mongoHelper::getBinData(chunkDoc.value(), 
                                            "data", buffUp, currChunkSize);
                    buffP = buffUp.get();
                }else{
                    throw std::runtime_error("cann't read next chunk");
                }
            }
        }

        currChunkNo_ = currChunkNo;
        currPos_ = currChunkNo_  * chunkSize_ + posInChunk;
        currChunkSize_ = currChunkSize;
        // buffPtr_ = std::move(buff);
        if(buffPtr_.get() != buffP){
            buffPtr_ = std::move(buffUp);
        }
        return true;
    }
```

## getLine()
getLine的另一个版本，不需要调用者提供字符数组，直接返回字符串
内部实现也是通过调用`getLine(char *s, const size_t n)`实现的
返回值使用optional是为了处理读到空行的情况
```C++
    const std::optional<std::string> MongoFileReader::getLine(){
        char buffer[lineBufferSize_];
        auto currPos = tellg();
        if(getLine(buffer, lineBufferSize_) == false){
            return nullopt;
        }
        string line = buffer;
        for(int times = 1; line.size() == times * lineBufferSize_; times++){
            if(getLine(buffer, lineBufferSize_) == false){
                seekg(currPos, ios_base::beg);
                return nullopt;
            }
            line.append(buffer);
        }
        return line;
    }
```

## seekg
计算要移动到的位置所在的块，如果所在块与当前的缓冲区中的块是同一块，则不需要从数据库获取。
```C++
    bool MongoFileReader::seekg(const size_t offset, ios_base::seekdir dir){
        size_t nextPos;
        if(dir == ios_base::beg){
            nextPos = offset;
        }else if(dir == ios_base::cur){
            nextPos = currPos_ + offset;
        }else{
            nextPos = fileSize_ - offset;
        }

        if(nextPos < 0 || nextPos >= fileSize_){
            return false;
        }
        auto nextChunkNo = nextPos / chunkSize_;
        
        error::PCError err;
        if(nextChunkNo != currChunkNo_){
            //加载新的chunk
            if(auto chunkDoc = getChunkDoc_(err, nextChunkNo); 
                chunkDoc.has_value()){
                DAO::mongoHelper::getBinData(chunkDoc.value(), 
                                                "data", buffPtr_, currChunkSize_);
            }else{
                throw std::runtime_error("cann't read next chunk");
            }
        }
        currPos_ = nextPos;
        currChunkNo_ = nextChunkNo;

        return true;
    }
```