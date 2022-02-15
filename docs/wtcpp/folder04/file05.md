# WTSLogger.h

source: `{{ page.path }}`

```cpp
/*!
 * \file WTSLogger.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 日志模块定义
 */
#pragma once
#include "../Includes/WTSTypes.h"
#include "../Includes/WTSCollection.hpp"

#include <memory>
#include <sstream>
#include <thread>
#include <set>

// 包装 Spdlog 处理日志
//By Wesley @ 2022.01.05
//spdlog升级到1.9.2
//同时使用外部的fmt 8.1.0
#include <spdlog/spdlog.h>

typedef std::shared_ptr<spdlog::logger> SpdLoggerPtr;

NS_WTP_BEGIN
class ILogHandler;
class WTSVariant;
NS_WTP_END

USING_NS_WTP;

#define MAX_LOG_BUF_SIZE 2048

#include <spdlog/fmt/bundled/printf.h>
template<typename... Args>
inline void fmt_print_impl(char* buf, const char* format, const Args&... args)
{
	static std::string s;
	s = std::move(fmt::sprintf(format, args...));
	strcpy(buf, s.c_str());
	buf[s.size()] = '\0';
}

template<typename... Args>
inline void fmt_format_impl(char* buf, const char* format, const Args&... args)
{
	memset(buf, 0, MAX_LOG_BUF_SIZE);
	fmt::format_to(buf, format, args...);
}


class WTSLogger
{
private:
	static void debug_imp(SpdLoggerPtr logger, const char* message);
	static void info_imp(SpdLoggerPtr logger, const char* message);
	static void warn_imp(SpdLoggerPtr logger, const char* message);
	static void error_imp(SpdLoggerPtr logger, const char* message);
	static void fatal_imp(SpdLoggerPtr logger, const char* message);

	static void initLogger(const char* catName, WTSVariant* cfgLogger);
	static SpdLoggerPtr getLogger(const char* logger, const char* pattern = "");

	static void print_message(const char* buffer);

public:
	/*
	 *	直接输出
	 */
	static void log_raw(WTSLogLevel ll, const char* message);

	/*
	 *	分类输出
	 */
	static void log_raw_by_cat(const char* catName, WTSLogLevel ll, const char* message);

	/*
	 *	动态分类输出
	 */
	static void log_dyn_raw(const char* patttern, const char* catName, WTSLogLevel ll, const char* message);

//printf风格接口
#pragma region "printf style apis"
public:
	template<typename... Args>
	static void debug(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_DEBUG || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		debug_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void info(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_INFO || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		info_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void warn(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_WARN || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		warn_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void error(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_ERROR || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		error_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void fatal(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_FATAL || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		fatal_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void log(WTSLogLevel ll, const char* format, const Args& ...args)
	{
		if (m_logLevel > ll || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		log_raw(ll, m_buffer);
	}

	template<typename... Args>
	static void log_by_cat(const char* catName, WTSLogLevel ll, const char* format, const Args& ...args)
	{
		if (m_logLevel > ll || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		log_raw_by_cat(catName, ll, m_buffer);
	}

	template<typename... Args>
	static void log_dyn(const char* patttern, const char* catName, WTSLogLevel ll, const char* format, const Args& ...args)
	{
		if (m_logLevel > ll || m_bStopped)
			return;

		fmt_print_impl(m_buffer, format, args...);

		log_dyn_raw(patttern, catName, ll, m_buffer);
	}
#pragma endregion "printf style apis"

//fmt::format风格接口
#pragma region "format style apis"
public:
	template<typename... Args>
	static void debug_f(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_DEBUG || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		debug_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void info_f(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_INFO || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		info_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void warn_f(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_WARN || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		warn_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void error_f(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_ERROR || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		error_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void fatal_f(const char* format, const Args& ...args)
	{
		if (m_logLevel > LL_FATAL || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		if (!m_bInited)
		{
			print_message(m_buffer);
			return;
		}

		fatal_imp(m_rootLogger, m_buffer);
	}

	template<typename... Args>
	static void log_f(WTSLogLevel ll, const char* format, const Args& ...args)
	{
		if (m_logLevel > ll || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		log_raw(ll, m_buffer);
	}

	template<typename... Args>
	static void log_by_cat_f(const char* catName, WTSLogLevel ll, const char* format, const Args& ...args)
	{
		if (m_logLevel > ll || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		log_raw_by_cat(catName, ll, m_buffer);
	}

	template<typename... Args>
	static void log_dyn_f(const char* patttern, const char* catName, WTSLogLevel ll, const char* format, const Args& ...args)
	{
		if (m_logLevel > ll || m_bStopped)
			return;

		fmt_format_impl(m_buffer, format, args...);

		log_dyn_raw(patttern, catName, ll, m_buffer);
	}
#pragma endregion "format style apis"
public:
	static void init(const char* propFile = "logcfg.json", bool isFile = true, ILogHandler* handler = NULL, WTSLogLevel logLevel = LL_DEBUG);

	static void registerHandler(ILogHandler* handler = NULL, WTSLogLevel logLevel = LL_DEBUG);

	static void stop();

	static void freeAllDynLoggers();

private:
	static bool					m_bInited;
	static bool					m_bTpInited;
	static bool					m_bStopped;
	static ILogHandler*			m_logHandler;
	static WTSLogLevel			m_logLevel;

	static SpdLoggerPtr			m_rootLogger;

	typedef WTSHashMap<std::string>	LogPatterns;
	static LogPatterns*			m_mapPatterns;
	static std::set<std::string>	m_setDynLoggers;

	static thread_local char	m_buffer[MAX_LOG_BUF_SIZE];
};
```

## WTSLogger.cpp

实现逻辑

```cpp
/*!
 * \file WTSLogger.cpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#include <stdio.h>
#include <iostream>
#include <sys/timeb.h>
#ifdef _MSC_VER
#include <time.h>
#else
#include <sys/time.h>
#endif

#include "WTSLogger.h"
#include "../WTSUtils/WTSCfgLoader.h"
#include "../Includes/ILogHandler.h"
#include "../Includes/WTSVariant.hpp"
#include "../Share/StdUtils.hpp"
#include "../Share/StrUtil.hpp"
#include "../Share/TimeUtils.hpp"

#include <boost/filesystem.hpp>

#include <spdlog/sinks/daily_file_sink.h>
#include <spdlog/sinks/basic_file_sink.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <spdlog/sinks/ostream_sink.h>
#include <spdlog/async.h>

const char* DYN_PATTERN = "dyn_pattern";

ILogHandler*		WTSLogger::m_logHandler	= NULL;			// 日志句柄
WTSLogLevel			WTSLogger::m_logLevel	= LL_ALL;		// 日志级别
bool				WTSLogger::m_bStopped = false;
bool				WTSLogger::m_bInited = false;
bool				WTSLogger::m_bTpInited = false;
SpdLoggerPtr		WTSLogger::m_rootLogger = NULL;
WTSLogger::LogPatterns*	WTSLogger::m_mapPatterns = NULL;
thread_local char	WTSLogger::m_buffer[];
std::set<std::string>	WTSLogger::m_setDynLoggers;

// 根据输入的字符串获取对应的日志级别
inline spdlog::level::level_enum str_to_level( const char* slvl)
{
	if(wt_stricmp(slvl, "debug") == 0)
	{
		return spdlog::level::debug;
	}
	else if (wt_stricmp(slvl, "info") == 0)
	{
		return spdlog::level::info;
	}
	else if (wt_stricmp(slvl, "warn") == 0)
	{
		return spdlog::level::warn;
	}
	else if (wt_stricmp(slvl, "error") == 0)
	{
		return spdlog::level::err;
	}
	else if (wt_stricmp(slvl, "fatal") == 0)
	{
		return spdlog::level::critical;
	}
	else
	{
		return spdlog::level::off;
	}
}

// 检查日志文件路径, 没有就新建
inline void checkDirs(const char* filename)
{
	std::string s = StrUtil::standardisePath(filename, false);
	std::size_t pos = s.find_last_of('/');

	if (pos == std::string::npos)
		return;

	pos++;

	if (!StdFile::exists(s.substr(0, pos).c_str()))
		boost::filesystem::create_directories(s.substr(0, pos).c_str());
}

// 打印时间标签
static inline void print_timetag(bool bWithSpace = true)
{
	timeb now;
	ftime(&now);

	tm * tNow = localtime(&(now.time));
	printf("[%d.%02d.%02d %02d:%02d:%02d]", tNow->tm_year + 1900, tNow->tm_mon + 1, tNow->tm_mday, tNow->tm_hour, tNow->tm_min, tNow->tm_sec);
	if (bWithSpace)
		printf(" ");
}

void WTSLogger::print_message(const char* buffer)
{
	print_timetag(true);
	printf(buffer);
	printf("\r\n");
}

// 从解析logcfg.json的WTSVariant变量中初始化日志对象
void WTSLogger::initLogger(const char* catName, WTSVariant* cfgLogger)
{
	bool bAsync = cfgLogger->getBoolean("async");
	const char* level = cfgLogger->getCString("level");

	WTSVariant* cfgSinks = cfgLogger->get("sinks");
	std::vector<spdlog::sink_ptr> sinks;
	for (uint32_t idx = 0; idx < cfgSinks->size(); idx++)
	{
		WTSVariant* cfgSink = cfgSinks->get(idx);
		const char* type = cfgSink->getCString("type");
		if (strcmp(type, "daily_file_sink") == 0)
		{
			std::string filename = cfgSink->getString("filename");
			StrUtil::replace(filename, "%s", catName);
			checkDirs(filename.c_str());
			auto sink = std::make_shared<spdlog::sinks::daily_file_sink_mt>(filename, 0, 0);
			sink->set_pattern(cfgSink->getCString("pattern"));
			sinks.emplace_back(sink);
		}
		else if (strcmp(type, "basic_file_sink") == 0)
		{
			std::string filename = cfgSink->getString("filename");
			StrUtil::replace(filename, "%s", catName);
			checkDirs(filename.c_str());
			auto sink = std::make_shared<spdlog::sinks::basic_file_sink_mt>(filename, cfgSink->getBoolean("truncate"));
			sink->set_pattern(cfgSink->getCString("pattern"));
			sinks.emplace_back(sink);
		}
		else if (strcmp(type, "console_sink") == 0)
		{
			auto sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
			sink->set_pattern(cfgSink->getCString("pattern"));
			sinks.emplace_back(sink);
		}
		else if (strcmp(type, "ostream_sink") == 0)
		{
			auto sink = std::make_shared<spdlog::sinks::ostream_sink_mt>(std::cout, true);
			sink->set_pattern(cfgSink->getCString("pattern"));
			sinks.emplace_back(sink);
		}
	}

	if (!bAsync)
	{
		auto logger = std::make_shared<spdlog::logger>(catName, sinks.begin(), sinks.end());
		logger->set_level(str_to_level(cfgLogger->getCString("level")));
		spdlog::register_logger(logger);
	}
	else
	{
		if(!m_bTpInited)
		{
			spdlog::init_thread_pool(8192, 2);
			m_bTpInited = true;
		}

		auto logger = std::make_shared<spdlog::async_logger>(catName, sinks.begin(), sinks.end(), spdlog::thread_pool(), spdlog::async_overflow_policy::block);
		logger->set_level(str_to_level(cfgLogger->getCString("level")));
		spdlog::register_logger(logger);
	}
}

// 加载 logcfg.json 文件初始化日志对象
void WTSLogger::init(const char* propFile /* = "logcfg.json" */, bool isFile /* = true */, ILogHandler* handler /* = NULL */, WTSLogLevel logLevel /* = LL_INFO */)
{
	if (m_bInited)
		return;

	if (isFile && !StdFile::exists(propFile))
		return;

	WTSVariant* cfg = isFile ? WTSCfgLoader::load_from_file(propFile, true) : WTSCfgLoader::load_from_content(propFile, false, true);
	if (cfg == NULL)
		return;

	auto keys = cfg->memberNames();
	for (std::string& key : keys)
	{
		WTSVariant* cfgItem = cfg->get(key.c_str());
		if (key == DYN_PATTERN)
		{
			auto pkeys = cfgItem->memberNames();
			for(std::string& pkey : pkeys)
			{
				WTSVariant* cfgPattern = cfgItem->get(pkey.c_str());
				if (m_mapPatterns == NULL)
					m_mapPatterns = LogPatterns::create();

				m_mapPatterns->add(pkey.c_str(), cfgPattern, true);
			}
			continue;
		}

		initLogger(key.c_str(), cfgItem);
	}

	m_rootLogger = getLogger("root");
	spdlog::set_default_logger(m_rootLogger);
	spdlog::flush_every(std::chrono::seconds(2));

	m_logHandler = handler;
	m_logLevel = logLevel;

	m_bInited = true;
}

// 注册日志句柄
void WTSLogger::registerHandler(ILogHandler* handler /* = NULL */, WTSLogLevel logLevel /* = LL_ALL */)
{
	m_logHandler = handler;
	m_logLevel = logLevel;
}

// 关闭日志
void WTSLogger::stop()
{
	m_bStopped = true;
	if (m_mapPatterns)
		m_mapPatterns->release();
	spdlog::shutdown();
}

// 处理不同级别日志
void WTSLogger::debug_imp(SpdLoggerPtr logger, const char* message)
{
	if (logger)
		logger->debug(message);

	if (logger != m_rootLogger)
		m_rootLogger->debug(message);

	if (m_logHandler)
		m_logHandler->handleLogAppend(LL_DEBUG, message);
}

void WTSLogger::info_imp(SpdLoggerPtr logger, const char* message)
{
	if (logger)
		logger->info(message);

	if (logger != m_rootLogger)
		m_rootLogger->info(message);

	if (m_logHandler)
		m_logHandler->handleLogAppend(LL_INFO, message);
}

void WTSLogger::warn_imp(SpdLoggerPtr logger, const char* message)
{
	if (logger)
		logger->warn(message);

	if (logger != m_rootLogger)
		m_rootLogger->warn(message);

	if (m_logHandler)
		m_logHandler->handleLogAppend(LL_WARN, message);
}

void WTSLogger::error_imp(SpdLoggerPtr logger, const char* message)
{
	if (logger)
		logger->error(message);

	if (logger != m_rootLogger)
		m_rootLogger->error(message);

	if (m_logHandler)
		m_logHandler->handleLogAppend(LL_ERROR, message);
}

void WTSLogger::fatal_imp(SpdLoggerPtr logger, const char* message)
{
	if (logger)
		logger->critical(message);

	if (logger != m_rootLogger)
		m_rootLogger->critical(message);

	if (m_logHandler)
		m_logHandler->handleLogAppend(LL_FATAL, message);
}

void WTSLogger::log_raw(WTSLogLevel ll, const char* message)
{
	if (m_logLevel > ll || m_bStopped)
		return;

	if (!m_bInited)
	{
		print_message(message);
		return;
	}

	auto logger = m_rootLogger;

	if (logger)
	{
		switch (ll)
		{
		case LL_DEBUG:
			debug_imp(logger, message); break;
		case LL_INFO:
			info_imp(logger, message); break;
		case LL_WARN:
			warn_imp(logger, message); break;
		case LL_ERROR:
			error_imp(logger, message); break;
		case LL_FATAL:
			fatal_imp(logger, message); break;
		default:
			break;
		}
	}
}

void WTSLogger::log_raw_by_cat(const char* catName, WTSLogLevel ll, const char* message)
{
	if (m_logLevel > ll || m_bStopped)
		return;

	auto logger = getLogger(catName);

	if (!m_bInited)
	{
		print_timetag(true);
		printf(message);
		printf("\r\n");
		return;
	}

	if (logger)
	{
		switch (ll)
		{
		case LL_DEBUG:
			debug_imp(logger, message);
			break;
		case LL_INFO:
			info_imp(logger, message);
			break;
		case LL_WARN:
			warn_imp(logger, message);
			break;
		case LL_ERROR:
			error_imp(logger, message);
			break;
		case LL_FATAL:
			fatal_imp(logger, message);
			break;
		default:
			break;
		}
	}	
}

void WTSLogger::log_dyn_raw(const char* patttern, const char* catName, WTSLogLevel ll, const char* message)
{
	if (m_logLevel > ll || m_bStopped)
		return;

	auto logger = getLogger(catName, patttern);

	if (!m_bInited)
	{
		print_timetag(true);
		printf(m_buffer);
		printf("\r\n");
		return;
	}

	if (logger)
	{
		switch (ll)
		{
		case LL_DEBUG:
			debug_imp(logger, message);
			break;
		case LL_INFO:
			info_imp(logger, message);
			break;
		case LL_WARN:
			warn_imp(logger, message);
			break;
		case LL_ERROR:
			error_imp(logger, message);
			break;
		case LL_FATAL:
			fatal_imp(logger, message);
			break;
		default:
			break;
		}
	}
}


SpdLoggerPtr WTSLogger::getLogger(const char* logger, const char* pattern /* = "" */)
{
	SpdLoggerPtr ret = spdlog::get(logger);
	if (ret == NULL && strlen(pattern) > 0)
	{
		//当成动态的日志来处理
		if (m_mapPatterns == NULL)
			return SpdLoggerPtr();

		WTSVariant* cfg = (WTSVariant*)m_mapPatterns->get(pattern);
		if (cfg == NULL)
			return SpdLoggerPtr();

		initLogger(logger, cfg);

		m_setDynLoggers.insert(logger);

		return spdlog::get(logger);
	}

	return ret;
}

void WTSLogger::freeAllDynLoggers()
{
	for(const std::string& logger : m_setDynLoggers)
	{
		auto loggerPtr = spdlog::get(logger);
		if(!loggerPtr)
			continue;

		spdlog::drop(logger);
	}
}
```