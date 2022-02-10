# 数据压缩 

source: `{{ page.path }}`

## WTSCmpHelper.hpp

数据转存储需要压缩及解压.dsb文件

### WTSCmpHelper

```cpp
class WTSCmpHelper
{
public:
	// 压缩
	static std::string compress_data(const void* data, uint32_t dataLen, uint32_t uLevel = 1)
	{
		std::string desBuf;
		std::size_t const desLen = ZSTD_compressBound(dataLen);
		desBuf.resize(desLen, 0);
		size_t const cSize = ZSTD_compress((void*)desBuf.data(), desLen, data, dataLen, uLevel);
		desBuf.resize(cSize);
		return desBuf;
	}

	// 解压
	static std::string uncompress_data(const void* data, uint32_t dataLen)
	{
		std::string desBuf;
		unsigned long long const desLen = ZSTD_getFrameContentSize(data, dataLen);
		desBuf.resize((std::size_t)desLen, 0);
		size_t const dSize = ZSTD_decompress((void*)desBuf.data(), (size_t)desLen, data, dataLen);
		if (dSize != desLen)
			throw std::runtime_error("uncompressed data size does not match calculated data size");
		return desBuf;
	}
};
```