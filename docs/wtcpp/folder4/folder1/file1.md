# 01-01 路径处理

source: `{{ page.path }}`

## WtHelper.h

处理工作路径和输出路径

### WtHelper

```cpp
class WtHelper
{
public:
	static std::string getCWD();

	static const char* getOutputDir();

	static const std::string& getInstDir() { return _inst_dir; }
	static void setInstDir(const char* inst_dir) { _inst_dir = inst_dir; }
	static void setOutputDir(const char* out_dir);

private:
	static std::string	_inst_dir;	//实例所在目录
	static std::string	_out_dir;
};
```

## WtHelper.cpp

实现逻辑

```cpp
std::string WtHelper::_inst_dir;
std::string WtHelper::_out_dir = "./outputs_bt";

// 获取当前工作绝对路径
std::string WtHelper::getCWD()
{
	static std::string _cwd;
	if(_cwd.empty())
	{
		char   buffer[255];
#ifdef _MSC_VER
		_getcwd(buffer, 255);
#else	//UNIX
		getcwd(buffer, 255);
#endif
		_cwd = buffer;
		_cwd = StrUtil::standardisePath(_cwd);
	}	
	return _cwd;
}

// 设置输出路径
void WtHelper::setOutputDir(const char* out_dir)
{
	_out_dir = StrUtil::standardisePath(std::string(out_dir));
}

// 获取输出路径
const char* WtHelper::getOutputDir()
{
	if (!boost::filesystem::exists(_out_dir.c_str()))
        boost::filesystem::create_directories(_out_dir.c_str());
	return _out_dir.c_str();
}
```