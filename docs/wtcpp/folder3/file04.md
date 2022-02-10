# 读取CSV文件

source: `{{ page.path }}`

## CsvHelper.h

读取CSV文件, 内容比较简单: 加载文件 -> 获取列名 -> 逐行读取 -> 数据类型转换

### CsvReader

```cpp
class CsvReader
{
public:
	CsvReader(const char* item_splitter = ",");

public:
	bool	load_from_file(const char* filename);

public:
	inline uint32_t	col_count() { return _fields_map.size(); }

	int32_t		get_int32(int32_t col);
	uint32_t	get_uint32(int32_t col);

	int64_t		get_int64(int32_t col);
	uint64_t	get_uint64(int32_t col);

	double		get_double(int32_t col);

	const char*	get_string(int32_t col);

	int32_t		get_int32(const char* field);
	uint32_t	get_uint32(const char* field);

	int64_t		get_int64(const char* field);
	uint64_t	get_uint64(const char* field);

	double		get_double(const char* field);

	const char*	get_string(const char* field);

	bool		next_row();

	const char* fields() const 
	{ 
		static std::string s;
		if(s.empty())
		{
			std::stringstream ss;
			for (auto item : _fields_map)
				ss << item.first << ",";

			s = ss.str();
			s = s.substr(0, s.size() - 1);
		}

		return s.c_str();
	}

private:
	bool		check_cell(int32_t col);
	int32_t		get_col_by_filed(const char* field);

private:
	std::ifstream	_ifs;				// 文件流
	char			_buffer[1024];
	std::string		_item_splitter;		// 文件分隔符

	std::unordered_map<std::string, int32_t> _fields_map;						// 列名字典
	std::vector<std::string> _current_cells;	// 每列数据字段
};
```

## CsvHelper.cpp

```cpp
CsvReader::CsvReader(const char* item_splitter /* = "," */)
	: _item_splitter(item_splitter)
{}

// 加载CSV文件, 保存列名字典到 _fields_map 中
bool CsvReader::load_from_file(const char* filename)
{
	if (!StdFile::exists(filename))
		return false;

	_ifs.open(filename);

	_ifs.getline(_buffer, 1024);
	// 判断是不是UTF-8BOM 编码
	static char flag[] = { (char)0xEF, (char)0xBB, (char)0xBF };
	char* buf = _buffer;
	if (memcmp(_buffer, flag, sizeof(char) * 3) == 0)
		buf += 3;

	std::string row = buf;

	//替换掉一些字段的特殊符号
	StrUtil::replace(row, "<", "");
	StrUtil::replace(row, ">", "");
	StrUtil::replace(row, "\"", "");
	StrUtil::replace(row, "'", "");

	// 将字段名转成小写
	StrUtil::toLowerCase(row);

	StringVector fields = StrUtil::split(row, _item_splitter.c_str());
	for (uint32_t i = 0; i < fields.size(); i++)
	{
		std::string field = StrUtil::trim(fields[i].c_str(), " ");
		if (field.empty())
			break;

		_fields_map[field] = i;
	}
	return true;
}

// 获取下一列数据字段保存到 _current_cells 中
bool CsvReader::next_row()
{
	if (_ifs.eof())
		return false;

	while (!_ifs.eof())
	{
		_ifs.getline(_buffer, 1024);
		if(strlen(_buffer) == 0)
			continue;
		else
			break;
	} 
	
	if (strlen(_buffer) == 0)
		return false;
	_current_cells.clear();
	StrUtil::split(_buffer, _current_cells, _item_splitter.c_str());
	return true;
}

// 将数据字段转为对应的类型
int32_t CsvReader::get_int32(int32_t col)
{
	if (!check_cell(col))
		return 0;

	return strtol(_current_cells[col].c_str(), NULL, 10);
}

uint32_t CsvReader::get_uint32(int32_t col)
{
	if (!check_cell(col))
		return 0;

	return strtoul(_current_cells[col].c_str(), NULL, 10);
}

int64_t CsvReader::get_int64(int32_t col)
{
	if (!check_cell(col))
		return 0;

	return strtoll(_current_cells[col].c_str(), NULL, 10);
}

uint64_t CsvReader::get_uint64(int32_t col)
{
	if (!check_cell(col))
		return 0;

	return strtoull(_current_cells[col].c_str(), NULL, 10);
}

// 将数据字段转为double
double CsvReader::get_double(int32_t col)
{
	if (!check_cell(col))
		return 0;

	return strtod(_current_cells[col].c_str(), NULL);
}

// 将字段数据转为字符串
const char* CsvReader::get_string(int32_t col)
{
	if (!check_cell(col))
		return "";

	return _current_cells[col].c_str();
}

// 通过列名获取对应字段并做类型转换
int32_t CsvReader::get_int32(const char* field)
{
	int32_t col = get_col_by_filed(field);
	return get_int32(col);
}

uint32_t CsvReader::get_uint32(const char* field)
{
	int32_t col = get_col_by_filed(field);
	return get_uint32(col);
}

int64_t CsvReader::get_int64(const char* field)
{
	int32_t col = get_col_by_filed(field);
	return get_int64(col);
}

uint64_t CsvReader::get_uint64(const char* field)
{
	int32_t col = get_col_by_filed(field);
	return get_uint64(col);
}

double CsvReader::get_double(const char* field)
{
	int32_t col = get_col_by_filed(field);
	return get_double(col);
}

const char* CsvReader::get_string(const char* field)
{
	int32_t col = get_col_by_filed(field);
	return get_string(col);
}

bool CsvReader::check_cell(int32_t col)
{
	if (col == INT_MAX )
		return false;

	if (col < 0 || col >= (int32_t)_fields_map.size())
		return false;

	return true;
}

// 找到列名对应索引
int32_t CsvReader::get_col_by_filed(const char* field)
{
	auto it = _fields_map.find(field);
	if (it == _fields_map.end())
		return INT_MAX;

	return it->second;
}
```