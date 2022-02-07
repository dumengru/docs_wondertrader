# 01-04 文件映射

source: `{{ page.path }}`

## BoostMappingFile.hpp

### BoostMappingFile

将文件的内容映射到进程的地址空间

```cpp
class BoostMappingFile
{
private:
	std::string _file_name;                             // 文件名称
	boost::interprocess::file_mapping *_file_map;       // 
	boost::interprocess::mapped_region *_map_region;

public:
	BoostMappingFile()
	{
		_file_map=NULL;
		_map_region=NULL;
	}

	~BoostMappingFile()
	{
		close();
	}

    // 关闭文件指针
	void close()
	{
		if(_map_region!=NULL)
			delete _map_region;

		if(_file_map!=NULL)
			delete _file_map;

		_file_map=NULL;
		_map_region=NULL;
	}

	void sync()
	{
		if(_map_region)
			_map_region->flush();
	}

	void *addr()
	{
		if(_map_region)
			return _map_region->get_address();
		return NULL;
	}

	size_t size()
	{
		if(_map_region)
			return _map_region->get_size();
		return 0;
	}

	bool map(const char *filename,
		int mode=boost::interprocess::read_write,
		int mapmode=boost::interprocess::read_write,bool zeroother=true)
	{
		if (!boost::filesystem::exists(filename))
		{
			return false;
		}
		_file_name = filename;

		_file_map = new boost::interprocess::file_mapping(filename,(boost::interprocess::mode_t)mode);
		if(_file_map==NULL)
			return false;

		_map_region = new boost::interprocess::mapped_region(*_file_map,(boost::interprocess::mode_t)mapmode);
		if(_map_region==NULL)
		{
			delete _file_map;
			return false;
		}

		return true;
	}

	const char* filename()
	{
		return _file_name.c_str();
	}

	bool valid() const
	{
		return _file_map != NULL;
	}
};
```