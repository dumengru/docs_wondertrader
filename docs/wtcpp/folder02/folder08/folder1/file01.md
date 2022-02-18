# WtHelper.h

source: `{{ page.path }}`

```cpp
/*!
 * \file WtHelper.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#pragma once
#include <string>
#include <stdint.h>

class WtHelper
{
public:
	// 获取当前工作目录
	static const char* get_cwd();
	// 获取模型输出目录
	static const char* get_module_dir(){ return _bin_dir.c_str(); }
	// 设置模型输出目录
	static void set_module_dir(const char* mod_dir){ _bin_dir = mod_dir; }

private:
	static std::string	_bin_dir;
};


```

## WtHelper.cpp

```cpp
/*!
 * \file WtHelper.cpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#include "WtHelper.h"

#include "../Share/StrUtil.hpp"

#ifdef _MSC_VER
#include <direct.h>
#else	//UNIX
#include <unistd.h>
#endif

std::string WtHelper::_bin_dir;

// 获取当前工作目录
const char* WtHelper::get_cwd()
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
	return _cwd.c_str();
}
```