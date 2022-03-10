# IniHelper.hpp

source: `{{ page.path }}`

```cpp
/*!
 * \file IniHelper.hpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief Ini文件辅助类,利用boost的property_tree来实现,可以跨平台使用
 */
#pragma once

#include <string>
#include <vector>
#include <map>

#include <boost/property_tree/ptree.hpp>  
#include <boost/property_tree/ini_parser.hpp>

/*
property_tree是一个保存了多个属性的树形数据结构

可以使用类似访问路径的方式问任意节点的属性, 而且每个节点都可以用类似STL的风格遍历子节点

 property_tree适合于应用程序的配置数据处理, 可以解析xml, ini, json和info四种格式的文本数据据
*/

typedef std::vector<std::string>			FieldArray;
typedef std::map<std::string, std::string>	FieldMap;

class IniHelper
{
private:
	boost::property_tree::ptree	_root;		// 树对象
	std::string					_fname;		// 要读取的文件名
	bool						_loaded;	// 

	static const uint32_t MAX_KEY_LENGTH = 64;

public:
	IniHelper(): _loaded(false){}

	void	load(const char* szFile)
	{
		_fname = szFile;
		try
		{
			boost::property_tree::ini_parser::read_ini(szFile, _root);
		}
		catch(...)
		{

		}
		
		_loaded = true;
	}

	void	save(const char* filename = "")
	{
		if (strlen(filename) > 0)
			boost::property_tree::ini_parser::write_ini(filename, _root);
		else
			boost::property_tree::ini_parser::write_ini(_fname.c_str(), _root);
	}

	inline bool isLoaded() const{ return _loaded; }

public:
	// 擦除值
	void	removeValue(const char* szSec, const char* szKey)
	{
		try
		{
			boost::property_tree::ptree& sec = _root.get_child(szSec);
			sec.erase(szKey);
		}
		catch (...)
		{
			
		}
	}
	// 擦除节点
	void	removeSection(const char* szSec)
	{
		try
		{
			_root.erase(szSec);
		}
		catch (...)
		{

		}
	}
	// 读取值模板
	template<class T>
	T	readValue(const char* szPath, T defVal)
	{
		try
		{
			return _root.get<T>(szPath, defVal);
		}
		catch (...)
		{
			return defVal;
		}
	}
	// 读取对应类型值
	std::string	readString(const char* szSec, const char* szKey, const char* defVal = "")
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		return readValue<std::string>(path, defVal);
	}

	int			readInt(const char* szSec, const char* szKey, int defVal = 0)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		return readValue<int>(path, defVal);
	}

	uint32_t	readUInt(const char* szSec, const char* szKey, uint32_t defVal = 0)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		return readValue<uint32_t>(path, defVal);
	}

	bool		readBool(const char* szSec, const char* szKey, bool defVal = false)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		return readValue<bool>(path, defVal);
	}

	double		readDouble(const char* szSec, const char* szKey, double defVal = 0.0)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		return readValue<double>(path, defVal);
	}
	// 读取父节点的键并保存到数组
	int			readSections(FieldArray &aySection)
	{
		for (auto it = _root.begin(); it != _root.end(); it++)
		{
			aySection.emplace_back(it->first.data());
		}

		return _root.size();
	}
	// 读取子节点的键并保存到数组
	int			readSecKeyArray(const char* szSec, FieldArray &ayKey)
	{
		try
		{
			const boost::property_tree::ptree& _sec = _root.get_child(szSec);
			for (auto it = _sec.begin(); it != _sec.end(); it++)
			{
				ayKey.emplace_back(it->first.data());
			}

			return _sec.size();
		}
		catch (...)
		{
			return 0;
		}
		
	}
	// 读取节点键值分别保存到数组
	int			readSecKeyValArray(const char* szSec, FieldArray &ayKey, FieldArray &ayVal)
	{
		try
		{
			const boost::property_tree::ptree& _sec = _root.get_child(szSec);
			for (auto it = _sec.begin(); it != _sec.end(); it++)
			{
				ayKey.emplace_back(it->first.data());
				ayVal.emplace_back(it->second.data());
			}

			return _sec.size();
		}
		catch (...)
		{
			return 0;
		}
	}
	// 写入值模板
	template<class T>
	void		writeValue(const char* szPath, T val)
	{
		_root.put<T>(szPath, val);
	}
	// 写入不同类型的值
	void		writeString(const char* szSec, const char* szKey, const char* val)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		writeValue<std::string>(path, val);
	}

	void		writeInt(const char* szSec, const char* szKey, int val)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		writeValue<int>(path, val);
	}

	void		writeUInt(const char* szSec, const char* szKey, uint32_t val)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		writeValue<uint32_t>(path, val);
	}

	void		writeBool(const char* szSec, const char* szKey, bool val)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		writeValue<bool>(path, val);
	}

	void		writeDouble(const char* szSec, const char* szKey, double val)
	{
		static char path[MAX_KEY_LENGTH] = { 0 };
		sprintf(path, "%s.%s", szSec, szKey);
		writeValue<double>(path, val);
	}
};
```