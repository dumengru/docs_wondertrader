# 标准库封装

source: `{{ page.path }}`

## StdUtils.hpp

### StdLocker

```cpp
// std线程类
typedef std::thread StdThread;
typedef std::shared_ptr<StdThread> StdThreadPtr;

// std互斥量和锁
typedef std::recursive_mutex	StdRecurMutex;
typedef std::mutex				StdUniqueMutex;
typedef std::condition_variable_any	StdCondVariable;

typedef std::unique_lock<StdUniqueMutex>	StdUniqueLock;

template<typename T>
class StdLocker
{
public:
	StdLocker(T& mtx)
	{
		mtx.lock();
		_mtx = &mtx;
	}

	~StdLocker(){
		_mtx->unlock();
	}

private:
	T* _mtx;
};
```

### StdFile

文件辅助类

```cpp
class StdFile
{
public:
    // 读文件
	static inline uint64_t read_file_content(const char* filename, std::string& content)
	{
		FILE* f = fopen(filename, "rb");
		fseek(f, 0, SEEK_END);
		uint32_t length = ftell(f);
		content.resize(length);
		fseek(f, 0, 0);
		fread((void*)content.data(), sizeof(char), length, f);
		fclose(f);
		return length;
	}

    // 写文件
	static inline void write_file_content(const char* filename, const std::string& content)
	{
		FILE* f = fopen(filename, "wb");
		fwrite((void*)content.data(), sizeof(char), content.size(), f);
		fclose(f);
	}

    // 写文件: 二进制
	static inline void write_file_content(const char* filename, const void* data, std::size_t length)
	{
		FILE* f = fopen(filename, "wb");
		fwrite(data, sizeof(char), length, f);
		fclose(f);
	}

    // 判断文件是否存在
	static inline bool exists(const char* filename)
	{
#if _WIN32
		int ret = _access(filename, 0);
#else
		int ret = access(filename, 0);
#endif
		return ret == 0;
	}
};
```