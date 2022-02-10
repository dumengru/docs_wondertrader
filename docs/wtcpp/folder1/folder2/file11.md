# 报错数据

source: `{{ page.path }}`

## WTSError.hpp



```cpp
class WTSError : public WTSObject
{
protected:
	WTSError():m_errCode(WEC_NONE),m_strMsg(""){}
	virtual ~WTSError(){}

public:
	// 创建报错对象
	static WTSError* create(WTSErroCode ec, const char* errmsg)
	{
		WTSError* pRet = new WTSError;
		pRet->m_errCode = ec;
		pRet->m_strMsg = errmsg;
		return pRet;
	}

	// 获取报错属性
	const char*		getMessage() const{return m_strMsg.c_str();}
	WTSErroCode		getErrorCode() const{return m_errCode;}

protected:
	WTSErroCode		m_errCode;	// 报错代码
	std::string		m_strMsg;	// 报错信息
};
```


