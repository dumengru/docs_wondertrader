# WtBtDtReader.h

source: `{{ page.path }}`

```cpp
/*!
 * \file IBtDtReader.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#pragma once
#include <string>

#include "../Includes/WTSTypes.h"

NS_WTP_BEGIN
class WTSVariant;

/*
 *	@brief 数据读取模块回调接口
 *	@details 主要用于数据读取模块向调用模块回调
 */
class IBtDtReaderSink
{
public:
	/*
	 *	@brief	输出数据读取模块的日志
	 */
	virtual void		reader_log(WTSLogLevel ll, const char* message) = 0;
};

/*
 *	@brief	回测数据读取接口
 *
 *	向核心模块提供行情数据(tick、K线)读取接口
 */
class IBtDtReader
{
public:
	IBtDtReader() :_sink(NULL) {}
	virtual ~IBtDtReader(){}

public:
	virtual void init(WTSVariant* cfg, IBtDtReaderSink* sink) { _sink = sink; }

	virtual bool read_raw_bars(const char* exchg, const char* code, WTSKlinePeriod period, std::string& buffer) = 0;
	virtual bool read_raw_ticks(const char* exchg, const char* code, uint32_t uDate, std::string& buffer) = 0;

protected:
	IBtDtReaderSink*	_sink;
};

//创建数据存储对象
typedef IBtDtReader* (*FuncCreateBtDtReader)();
//删除数据存储对象
typedef void(*FuncDeleteBtDtReader)(IBtDtReader* store);

NS_WTP_END
```

## WtBtDtReader.cpp

```cpp
#include "WtBtDtReader.h"

#include "../Includes/WTSVariant.hpp"
#include "../Share/TimeUtils.hpp"
#include "../Share/CodeHelper.hpp"
#include "../Share/DLLHelper.hpp"

#include "../Includes/WTSContractInfo.hpp"
#include "../Includes/IBaseDataMgr.h"
#include "../Includes/IHotMgr.h"
#include "../Includes/WTSDataDef.hpp"
#include "../Includes/WTSSessionInfo.hpp"

#include "../WTSUtils/WTSCmpHelper.hpp"
#include "../WTSUtils/WTSCfgLoader.h"

#include <rapidjson/document.h>
namespace rj = rapidjson;

//By Wesley @ 2022.01.05
#include "../Share/fmtlib.h"

// 回测数据读取日志输出
template<typename... Args>
inline void pipe_btreader_log(IBtDtReaderSink* sink, WTSLogLevel ll, const char* format, const Args&... args)
{
	if (sink == NULL)
		return;

	static thread_local char buffer[512] = { 0 };
	memset(buffer, 0, 512);
	fmt::format_to(buffer, format, args...);

	sink->reader_log(ll, buffer);
}

extern bool proc_block_data(std::string& content, bool isBar, bool bKeepHead = true);

extern "C"
{
	EXPORT_FLAG IBtDtReader* createBtDtReader()
	{
		IBtDtReader* ret = new WtBtDtReader();
		return ret;
	}

	EXPORT_FLAG void deleteBtDtReader(IBtDtReader* reader)
	{
		if (reader != NULL)
			delete reader;
	}
};

/*
 *	处理块数据
 */
extern bool proc_block_data(std::string& content, bool isBar, bool bKeepHead);

WtBtDtReader::WtBtDtReader()
{
}


WtBtDtReader::~WtBtDtReader()
{

}

void WtBtDtReader::init(WTSVariant* cfg, IBtDtReaderSink* sink)
{
	_sink = sink;
	if (cfg == NULL)
		return;
	// 获取数据文件
	_base_dir = cfg->getCString("path");
	_base_dir = StrUtil::standardisePath(_base_dir);

	pipe_btreader_log(_sink, LL_INFO, "WtBtDtReader initialized, root data dir is {}", _base_dir);
}

// 读取bar数据
bool WtBtDtReader::read_raw_bars(const char* exchg, const char* code, WTSKlinePeriod period, std::string& buffer)
{
	// 标准化bar数据地址 storage/his/min5/CFFEX/CFFEX.IF_HOT.dsb
	std::stringstream ss;
	ss << _base_dir << "his/" << PERIOD_NAME[period] << "/" << exchg << "/" << code << ".dsb";
	std::string filename = ss.str();
	if (!StdFile::exists(filename.c_str()))
	{
		pipe_btreader_log(_sink, LL_WARN, "Back {} data file {} not exists", PERIOD_NAME[period], filename);
		return false;
	}

	pipe_btreader_log(_sink, LL_DEBUG, "Reading back {} bars from file {}...", PERIOD_NAME[period], filename);
	// 读取文件内容
	StdFile::read_file_content(filename.c_str(), buffer);
	// 处理块数据
	bool bSucc = proc_block_data(buffer, true, false);
	if(!bSucc)
		pipe_btreader_log(_sink, LL_ERROR, "Processing back {} data from file {} failed", PERIOD_NAME[period], filename);

	return bSucc;
}

bool WtBtDtReader::read_raw_ticks(const char* exchg, const char* code, uint32_t uDate, std::string& buffer)
{
	// 标准化tick数据地址 storage/his/ticks/CFFEX/CFFEX.IF_HOT.dsb
	std::stringstream ss;
	ss << _base_dir << "his/ticks/" << exchg << "/" << uDate << "/" << code << ".dsb";
	std::string filename = ss.str();
	if (!StdFile::exists(filename.c_str()))
	{
		pipe_btreader_log(_sink, LL_WARN, "Back tick data file {} not exists", filename);
		return false;
	}
	// 读取数据并处理
	StdFile::read_file_content(filename.c_str(), buffer);
	bool bSucc = proc_block_data(buffer, false, false);
	if (!bSucc)
		pipe_btreader_log(_sink, LL_ERROR, "Processing back tick data from file {} failed", filename);

	return bSucc;
}
```

## WtBtDtReader.cpp

```cpp
#include "WtBtDtReader.h"

#include "../Includes/WTSVariant.hpp"
#include "../Share/TimeUtils.hpp"
#include "../Share/CodeHelper.hpp"
#include "../Share/DLLHelper.hpp"

#include "../Includes/WTSContractInfo.hpp"
#include "../Includes/IBaseDataMgr.h"
#include "../Includes/IHotMgr.h"
#include "../Includes/WTSDataDef.hpp"
#include "../Includes/WTSSessionInfo.hpp"

#include "../WTSUtils/WTSCmpHelper.hpp"
#include "../WTSUtils/WTSCfgLoader.h"

#include <rapidjson/document.h>
namespace rj = rapidjson;

//By Wesley @ 2022.01.05
#include "../Share/fmtlib.h"

// 回测数据读取日志输出
template<typename... Args>
inline void pipe_btreader_log(IBtDtReaderSink* sink, WTSLogLevel ll, const char* format, const Args&... args)
{
	if (sink == NULL)
		return;

	static thread_local char buffer[512] = { 0 };
	memset(buffer, 0, 512);
	fmt::format_to(buffer, format, args...);

	sink->reader_log(ll, buffer);
}

extern bool proc_block_data(std::string& content, bool isBar, bool bKeepHead = true);

extern "C"
{
	EXPORT_FLAG IBtDtReader* createBtDtReader()
	{
		IBtDtReader* ret = new WtBtDtReader();
		return ret;
	}

	EXPORT_FLAG void deleteBtDtReader(IBtDtReader* reader)
	{
		if (reader != NULL)
			delete reader;
	}
};

/*
 *	处理块数据
 */
extern bool proc_block_data(std::string& content, bool isBar, bool bKeepHead);

WtBtDtReader::WtBtDtReader()
{
}


WtBtDtReader::~WtBtDtReader()
{

}

void WtBtDtReader::init(WTSVariant* cfg, IBtDtReaderSink* sink)
{
	_sink = sink;
	if (cfg == NULL)
		return;
	// 获取数据文件
	_base_dir = cfg->getCString("path");
	_base_dir = StrUtil::standardisePath(_base_dir);

	pipe_btreader_log(_sink, LL_INFO, "WtBtDtReader initialized, root data dir is {}", _base_dir);
}

// 读取bar数据
bool WtBtDtReader::read_raw_bars(const char* exchg, const char* code, WTSKlinePeriod period, std::string& buffer)
{
	// 标准化bar数据地址 storage/his/min5/CFFEX/CFFEX.IF_HOT.dsb
	std::stringstream ss;
	ss << _base_dir << "his/" << PERIOD_NAME[period] << "/" << exchg << "/" << code << ".dsb";
	std::string filename = ss.str();
	if (!StdFile::exists(filename.c_str()))
	{
		pipe_btreader_log(_sink, LL_WARN, "Back {} data file {} not exists", PERIOD_NAME[period], filename);
		return false;
	}

	pipe_btreader_log(_sink, LL_DEBUG, "Reading back {} bars from file {}...", PERIOD_NAME[period], filename);
	// 读取文件内容
	StdFile::read_file_content(filename.c_str(), buffer);
	// 处理块数据
	bool bSucc = proc_block_data(buffer, true, false);
	if(!bSucc)
		pipe_btreader_log(_sink, LL_ERROR, "Processing back {} data from file {} failed", PERIOD_NAME[period], filename);

	return bSucc;
}

bool WtBtDtReader::read_raw_ticks(const char* exchg, const char* code, uint32_t uDate, std::string& buffer)
{
	// 标准化tick数据地址 storage/his/ticks/CFFEX/CFFEX.IF_HOT.dsb
	std::stringstream ss;
	ss << _base_dir << "his/ticks/" << exchg << "/" << uDate << "/" << code << ".dsb";
	std::string filename = ss.str();
	if (!StdFile::exists(filename.c_str()))
	{
		pipe_btreader_log(_sink, LL_WARN, "Back tick data file {} not exists", filename);
		return false;
	}
	// 读取数据并处理
	StdFile::read_file_content(filename.c_str(), buffer);
	bool bSucc = proc_block_data(buffer, false, false);
	if (!bSucc)
		pipe_btreader_log(_sink, LL_ERROR, "Processing back tick data from file {} failed", filename);

	return bSucc;
}
```