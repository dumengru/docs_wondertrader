# 2. 集合组件

source: `{{ page.path }}`

## WTSCollection.hpp

该文件下 WTSMap 和 WTSHashMap 一模一样, 似乎多余

### WTSArray

数组容器

- 内部使用vector实现, 数据使用WTSObject指针对象, 所有WTSObject的派生类都可以使用

<details>

```C++
class WTSArray : public WTSObject
{
public:
	// 数组迭代器
	typedef std::vector<WTSObject*>::iterator Iterator;
	typedef std::vector<WTSObject*>::const_iterator ConstIterator;

	typedef std::vector<WTSObject*>::reverse_iterator ReverseIterator;
	typedef std::vector<WTSObject*>::const_reverse_iterator ConstReverseIterator;

	typedef std::function<bool(WTSObject*, WTSObject*)>	SortFunc;

	// 创建数组对象
	static WTSArray* create()
	{
		WTSArray* pRet = new WTSArray();
		return pRet;
	}

	// 读取数组长度
	uint32_t size() const{ return (uint32_t)_vec.size(); }

	// 清空数组, 并重新分配空间, 用 NULL 填充
	void resize(uint32_t _size)
	{
		if(!_vec.empty())
			clear();
		_vec.resize(_size, NULL);
	}

	// 读取数组指定位置的数据, 不增加数据的引用计数
	WTSObject* at(uint32_t idx)
	{
		if(idx <0 || idx >= _vec.size())
			return NULL;
		WTSObject* pRet = _vec.at(idx);
		return pRet;
	}

    // 查询元素索引, 失败返回 -1
	uint32_t idxOf(WTSObject* obj)
	{
		if (obj == NULL)
			return -1;

		uint32_t idx = 0;
		auto it = _vec.begin();
		for (; it != _vec.end(); it++, idx++)
		{
			if (obj == (*it))
				return idx;
		}
		return -1;
	}

	// 读取数组指定位置的数据, 不增加数据的引用计数
	template<typename T> 
	T* at(uint32_t idx)
	{
		if(idx <0 || idx >= _vec.size())
			return NULL;

		WTSObject* pRet = _vec.at(idx);
		return static_cast<T*>(pRet);
	}

	// []操作符重载, 用法同at函数
	WTSObject* operator [](uint32_t idx)
	{
		if(idx <0 || idx >= _vec.size())
			return NULL;

		WTSObject* pRet = _vec.at(idx);
		return pRet;
	}

	// 读取数组指定位置的数据, 增加引用计数
	WTSObject*	grab(uint32_t idx)
	{
		if(idx <0 || idx >= _vec.size())
			return NULL;

		WTSObject* pRet = _vec.at(idx);
		if (pRet)
			pRet->retain();
		return pRet;
	}

	// 数组末尾追加数据, 数据自动增加引用计数
	void append(WTSObject* obj, bool bAutoRetain = true)
	{
		if (bAutoRetain && obj)
			obj->retain();

		_vec.emplace_back(obj);
	}

	// 设置指定位置的数据, 如果该位置已有数据,则释放掉, 新数据引用计数增加
	void set(uint32_t idx, WTSObject* obj, bool bAutoRetain = true)
	{
		if(idx >= _vec.size() || obj == NULL)
			return;

		if(bAutoRetain)
			obj->retain();

		WTSObject* oldObj = _vec.at(idx);
		if(oldObj)
			oldObj->release();

		_vec[idx] = obj;
	}

    // 在末尾追加 WTSArray
	void append(WTSArray* ay)
	{
		if(ay == NULL)
			return;

		_vec.insert(_vec.end(), ay->_vec.begin(), ay->_vec.end());
		ay->_vec.clear();
	}

	// 数组清空, 数组内所有数据释放引用
	void clear()
	{
		{
			std::vector<WTSObject*>::iterator it = _vec.begin();
			for (; it != _vec.end(); it++)
			{
				WTSObject* obj = (*it);
				if (obj)
					obj->release();
			}
		}
		
		_vec.clear();
	}

	// 释放数组对象,用法如WTSObject, 不同的是, 如果引用计数为1时, 释放所有数据
	virtual void release()
	{
		if (m_uRefs == 0)
			return;

		try
		{
			m_uRefs--;
			if (m_uRefs == 0)
			{
				clear();
				delete this;
			}
		}
		catch(...)
		{}
	}

	// 取得数组对象起始位置的迭代器
	Iterator begin()
	{
		return _vec.begin();
	}

	ConstIterator begin() const
	{
		return _vec.begin();
	}

	ReverseIterator rbegin()
	{
		return _vec.rbegin();
	}

	ConstReverseIterator rbegin() const
	{
		return _vec.rbegin();
	}

	// 取得数组对象末尾位置的迭代器
	Iterator end()
	{
		return _vec.end();
	}

	ConstIterator end() const
	{
		return _vec.end();
	}

	ReverseIterator rend()
	{
		return _vec.rend();
	}

	ConstReverseIterator rend() const
	{
		return _vec.rend();
	}

    // 自定义数组排序
	void sort(SortFunc func)
	{
		std::sort(_vec.begin(), _vec.end(), func);
	}

protected:
	WTSArray():_holding(false){}
	virtual ~WTSArray(){}

	std::vector<WTSObject*>	_vec;
	std::atomic<bool>		_holding;
};

```

</details>

### WTSMap

map容器
- 内部采用`std::map`实现, 模版类型为key类型, 数据使用WTSObject指针对象, 所有WTSObject的派生类都适用

<details>

```C++
template <typename T>
class WTSHashMap : public WTSObject
{
public:
	// 容器迭代器的定义
	typedef tsl::robin_map<T, WTSObject*>		_MyType;
	typedef typename _MyType::const_iterator	ConstIterator;

	// 创建map容器
	static WTSHashMap<T>*	create()
	{
		WTSHashMap<T>* pRet = new WTSHashMap<T>();
		return pRet;
	}

	// 返回map容器的大小
	uint32_t size() const{return (uint32_t)_map.size();}

	// 读取指定key对应的数据, 不增加数据的引用计数, 没有则返回NULL
	WTSObject* get(const T &_key)
	{
		auto it = _map.find(_key);
		if(it == _map.end())
			return NULL;

		WTSObject* pRet = it->second;
		return pRet;
	}

	// 读取指定key对应的数据, 增加数据的引用计数, 没有则返回NULL
	WTSObject* grab(const T &_key)
	{
		auto it = _map.find(_key);
		if(it == _map.end())
			return NULL;

		WTSObject* pRet = it->second;
		pRet->retain();
		return pRet;
	}

	// 新增一个数据, 并增加数据引用计数, 如果key存在, 则将原有数据释放
	void add(const T &_key, WTSObject* obj, bool bAutoRetain = true)
	{
		if (bAutoRetain && obj)
			obj->retain();

		WTSObject* pOldObj = NULL;
		auto it = _map.find(_key);
		if (it != _map.end())
		{
			pOldObj = it->second;
		}
		_map[_key] = obj;
		if (pOldObj) pOldObj->release();
	}

	// 根据key删除一个数据, 如果key存在, 则对应数据引用计数-1
	void remove(const T &_key)
	{
		auto it = _map.find(_key);
		if(it != _map.end())
		{
			it->second->release();
			_map.erase(it);
		}
	}

	// 获取容器起始位置的迭代器
	ConstIterator begin() const
	{
		return _map.begin();
	}

	// 获取末尾位置的迭代器
	ConstIterator end() const
	{
		return _map.end();
	}
	// 获取指定key对应的迭代器
	ConstIterator find(const T& key) const
	{
		return _map.find(key);
	}

	// 清空容器, 容器内所有数据引用计数-1
	void clear()
	{
		ConstIterator it = _map.begin();
		for(; it != _map.end(); it++)
		{
			it->second->release();
		}
		_map.clear();
	}

	// 释放容器对象, 如果容器引用计数为1, 则清空所有数据
	virtual void release()
	{
		if (m_uRefs == 0)
			return;
		try
		{
			m_uRefs--;
			if (m_uRefs == 0)
			{
				clear();
				delete this;
			}
		}
		catch (...)
		{}
	}

protected:
	WTSHashMap(){}
	virtual ~WTSHashMap(){}

	tsl::robin_map<T, WTSObject*>	_map;
};
```

</details>

### WTSHashMap

map容器
- 内部采用`std::map`实现, 模版类型为key类型, 数据使用WTSObject指针对象, 所有WTSObject的派生类都适用

<details>

```C++
template <typename T>
class WTSHashMap : public WTSObject
{
public:
	// 容器迭代器的定义
	typedef tsl::robin_map<T, WTSObject*>		_MyType;
	typedef typename _MyType::const_iterator	ConstIterator;

	// 创建map容器
	static WTSHashMap<T>*	create()
	{
		WTSHashMap<T>* pRet = new WTSHashMap<T>();
		return pRet;
	}

	// 返回map容器的大小
	uint32_t size() const{return (uint32_t)_map.size();}

	// 读取指定key对应的数据, 不增加数据的引用计数, 没有则返回NULL
	WTSObject* get(const T &_key)
	{
		auto it = _map.find(_key);
		if(it == _map.end())
			return NULL;

		WTSObject* pRet = it->second;
		return pRet;
	}

	// 读取指定key对应的数据, 增加数据的引用计数, 没有则返回NULL
	WTSObject* grab(const T &_key)
	{
		auto it = _map.find(_key);
		if(it == _map.end())
			return NULL;

		WTSObject* pRet = it->second;
		pRet->retain();
		return pRet;
	}

	// 新增一个数据,并增加数据引用计数, 如果key存在, 则将原有数据释放
	void add(const T &_key, WTSObject* obj, bool bAutoRetain = true)
	{
		if (bAutoRetain && obj)
			obj->retain();

		WTSObject* pOldObj = NULL;
		auto it = _map.find(_key);
		if (it != _map.end())
		{
			pOldObj = it->second;
		}

		_map[_key] = obj;

		if (pOldObj) pOldObj->release();
	}

	// 根据key删除一个数据, 如果key存在,则对应数据引用计数-1
	void remove(const T &_key)
	{
		auto it = _map.find(_key);
		if(it != _map.end())
		{
			it->second->release();
			_map.erase(it);
		}
	}

	// 获取容器起始位置的迭代器
	ConstIterator begin() const
	{
		return _map.begin();
	}

	// 获取容器末尾位置的迭代器
	ConstIterator end() const
	{
		return _map.end();
	}

	// 获取容器指定key对应的迭代器
	ConstIterator find(const T& key) const
	{
		return _map.find(key);
	}

	// 清空容器, 容器内所有数据引用计数-1
	void clear()
	{
		ConstIterator it = _map.begin();
		for(; it != _map.end(); it++)
		{
			it->second->release();
		}
		_map.clear();
	}

	// 释放容器对象, 如果容器引用计数为1, 则清空所有数据
	virtual void release()
	{
		if (m_uRefs == 0)
			return;

		try
		{
			m_uRefs--;
			if (m_uRefs == 0)
			{
				clear();
				delete this;
			}
		}
		catch (...)
		{}
	}

protected:
	WTSHashMap(){}
	virtual ~WTSHashMap(){}

	tsl::robin_map<T, WTSObject*>	_map;
};
```

</details>

### WTSQueue

队列容器
- 内部采用`std::deque`实现

<details>

```C++
class WTSQueue : public WTSObject
{
public:
	typedef std::deque<WTSObject*>::iterator Iterator;
	typedef std::deque<WTSObject*>::const_iterator ConstIterator;

	// 创建队列容器
	static WTSQueue* create()
	{
		WTSQueue* pRet = new WTSQueue();
		return pRet;
	}

	// 头部删除一个元素
	void pop()
	{
		_queue.pop_front();
	}

	// 末尾添加一个元素, 引用计数+1
	void push(WTSObject* obj, bool bAutoRetain = true)
	{
		if (obj && bAutoRetain)
			obj->retain();

		_queue.emplace_back(obj);
	}

	// 获取首元素的引用
	WTSObject* front(bool bRetain = true)
	{
		if(_queue.empty())
			return NULL;

		WTSObject* obj = _queue.front();
		if(bRetain)
			obj->retain();

		return obj;
	}

	// 获取尾元素的引用
	WTSObject* back(bool bRetain = true)
	{
		if(_queue.empty())
			return NULL;

		WTSObject* obj = _queue.back();
		if(bRetain)
			obj->retain();

		return obj;
	}

	// 获取容器大小
	uint32_t size() const{ return (uint32_t)_queue.size(); }

	// 判断容器是否为空
	bool	empty() const{return _queue.empty();}

	// 释放容器对象, 如果容器引用计数为1, 则清空所有数据
	void release()
	{
		if (m_uRefs == 0)
			return;

		try
		{
			m_uRefs--;
			if (m_uRefs == 0)
			{
				clear();
				delete this;
			}
		}
		catch (...)
		{}
	}

	// 清空容器, 容器内所有数据引用计数-1
	void clear()
	{
		Iterator it = begin();
		for(; it != end(); it++)
		{
			(*it)->release();
		}
		_queue.clear();
	}

	// 取得数组对象起始位置的迭代器
	Iterator begin()
	{
		return _queue.begin();
	}

	ConstIterator begin() const
	{
		return _queue.begin();
	}

	// 交换容器数据
	void swap(WTSQueue* right)
	{
		_queue.swap(right->_queue);
	}

	// 取得数组对象末尾位置的迭代器
	Iterator end()
	{
		return _queue.end();
	}

	ConstIterator end() const
	{
		return _queue.end();
	}

protected:
	WTSQueue(){}
	virtual ~WTSQueue(){}

	std::deque<WTSObject*>	_queue;
};
```

</details>