# 01-03 文件操作

source: `{{ page.path }}`

## BoostFile.hpp

### BoostFile

```cpp
class BoostFile
{
private:
	boost::interprocess::file_handle_t _handle;     // 文件句柄

public:
	BoostFile()
	{
		_handle=boost::interprocess::ipcdetail::invalid_file(); 
	}
	~BoostFile()
	{
		close_file();
	}

    // 创建新的文件对象
	bool create_new_file(const char *name, boost::interprocess::mode_t mode = boost::interprocess::read_write, bool temporary = false)
	{
		_handle=boost::interprocess::ipcdetail::create_or_open_file(name, mode, boost::interprocess::permissions(), temporary);

		if (valid())
			return truncate_file(0);
		return false;
	}

    // 创建或打开文件
	bool create_or_open_file(const char *name, boost::interprocess::mode_t mode = boost::interprocess::read_write, bool temporary = false)
	{
		_handle=boost::interprocess::ipcdetail::create_or_open_file(name, mode, boost::interprocess::permissions(), temporary);

		return valid();
	}

    // 打开已存在文件
	bool open_existing_file(const char *name, boost::interprocess::mode_t mode = boost::interprocess::read_write, bool temporary = false)
	{
		_handle=boost::interprocess::ipcdetail::open_existing_file(name, mode, temporary);
		return valid();
	}

    // 判断是否是无效文件
	bool is_invalid_file()
	{  
		return _handle==boost::interprocess::ipcdetail::invalid_file();  
	}

    // 判断是否是有效文件
	bool valid()
	{
		return _handle!=boost::interprocess::ipcdetail::invalid_file();
	}

    // 关闭文件
	void close_file()
	{
		if(!is_invalid_file())
		{
			boost::interprocess::ipcdetail::close_file(_handle);
			_handle=boost::interprocess::ipcdetail::invalid_file();
		}
	}

    // 删除文件
	bool truncate_file (std::size_t size)
	{
		return boost::interprocess::ipcdetail::truncate_file(_handle,size);
	}

    // 获取文件大小
	bool get_file_size(boost::interprocess::offset_t &size)
	{
		return boost::interprocess::ipcdetail::get_file_size(_handle,size);
	}

	unsigned long long get_file_size()
	{
		boost::interprocess::offset_t size=0;
		if(!get_file_size(size))
			size=0;
		return size;
	}

	static unsigned long long get_file_size(const char *name)
	{
		BoostFile bf;
		if (!bf.open_existing_file(name))
			return 0;

		auto ret = bf.get_file_size();
		bf.close_file();
		return ret;
	}

    // 设置文件指针
	bool set_file_pointer(boost::interprocess::offset_t off, boost::interprocess::file_pos_t pos)
	{
		return boost::interprocess::ipcdetail::set_file_pointer(_handle,off,pos);
	}

    // 文件光标操作
	bool seek_to_begin(int offsize=0)
	{
		return set_file_pointer(offsize,boost::interprocess::file_begin);
	}

	bool seek_current(int offsize=0)
	{
		return set_file_pointer(offsize,boost::interprocess::file_current);
	}

	bool seek_to_end(int offsize=0)
	{
		return set_file_pointer(offsize,boost::interprocess::file_end);
	}

    // 获取文件指针
	bool get_file_pointer(boost::interprocess::offset_t &off)
	{
		return boost::interprocess::ipcdetail::get_file_pointer(_handle,off);
	}

	unsigned long long get_file_pointer()
	{
		boost::interprocess::offset_t off=0;
		if(!get_file_pointer(off))
			return 0;
		return off;
	}

    // 文件读写
	bool write_file(const void *data, std::size_t numdata)
	{
		return boost::interprocess::ipcdetail::write_file(_handle,data,numdata);
	}

	bool write_file(const std::string& data)
	{
		return boost::interprocess::ipcdetail::write_file(_handle, data.data(), data.size());
	}

	bool read_file(void *data, std::size_t numdata)
	{
		unsigned long readbytes = 0;
#ifdef _WIN32
		int ret = ReadFile(_handle, data, (DWORD)numdata, &readbytes, NULL);
#else
		readbytes = read(_handle, data, (std::size_t)numdata);
#endif
		return numdata == readbytes;
	}

	int read_file_length(void *data, std::size_t numdata)
	{
		unsigned long readbytes = 0;
#ifdef _WIN32
		int ret = ReadFile(_handle, data, (DWORD)numdata, &readbytes, NULL);
#else
		readbytes = read(_handle, data, (std::size_t)numdata);
#endif
		return readbytes;
	}

public:
    // 删除文件
	static bool delete_file(const char *name)
	{
		return boost::interprocess::ipcdetail::delete_file(name);
	}

    // 读取文件内容
	static bool read_file_contents(const char *filename,std::string &buffer)
	{
		BoostFile bf;
		if(!bf.open_existing_file(filename,boost::interprocess::read_only))
			return false;
		unsigned int filesize=(unsigned int)bf.get_file_size();
		if(filesize==0)
			return false;
		buffer.resize(filesize);
		return bf.read_file((void *)buffer.c_str(),filesize);
	}

    // 将内容写入文件
	static bool write_file_contents(const char *filename,const void *pdata,uint32_t datalen)
	{
		BoostFile bf;
		if(!bf.create_new_file(filename))
			return false;
		return bf.write_file(pdata,datalen);
	}

    // 创建文件夹
	static bool create_directory(const char *name)
	{
		if(exists(name))
			return true;

		return boost::filesystem::create_directory(boost::filesystem::path(name));
	}

    // 递归式创建文件夹
	static bool create_directories(const char *name)
	{
		if(exists(name))
			return true;

		return boost::filesystem::create_directories(boost::filesystem::path(name));
	}

    // 判断文件是否存在
	static bool exists(const char* name)
	{
		return boost::filesystem::exists(boost::filesystem::path(name));
	}
};

typedef boost::shared_ptr<BoostFile> BoostFilePtr;
```