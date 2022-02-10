# 策略运行demo

source: `{{ page.path }}`

## WtBtRunner.cpp

```cpp

int main()
{
	// 通过 logcfg.json 初始化日志对象
	WTSLogger::init("logcfg.json");

	// 将 config.json 解析为 WTSVariant 对象
	std::string filename = "config.json";
	std::string content;
	StdFile::read_file_content(filename.c_str(), content);
	rj::Document root;
	if (root.Parse(content.c_str()).HasParseError())
	{
		WTSLogger::info("Parsing configuration file failed");
		return -1;
	}
	WTSVariant* cfg = WTSVariant::createObject();
	jsonToVariant(root, cfg);

	// 初始化回测引擎
	HisDataReplayer replayer;
	replayer.init(cfg->get("replayer"));

	// 通过配置文件初始化 cta 调度器
	WTSVariant* cfgEnv = cfg->get("env");
	const char* mode = cfgEnv->getCString("mocker");
	int32_t slippage = cfgEnv->getInt32("slippage");
	if (strcmp(mode, "cta") == 0)
	{
		CtaMocker* mocker = new CtaMocker(&replayer, "cta", slippage);
		mocker->init_cta_factory(cfg->get("cta"));
		replayer.register_sink(mocker, "cta");
	}
	else if (strcmp(mode, "hft") == 0)
	{
		HftMocker* mocker = new HftMocker(&replayer, "hft");
		mocker->init_hft_factory(cfg->get("hft"));
		replayer.register_sink(mocker, "hft");
	}
	else if (strcmp(mode, "sel") == 0)
	{
		SelMocker* mocker = new SelMocker(&replayer, "sel", slippage);
		mocker->init_sel_factory(cfg->get("cta"));
		replayer.register_sink(mocker, "sel");
	}
	else if (strcmp(mode, "exec") == 0)
	{
		ExecMocker* mocker = new ExecMocker(&replayer);
		mocker->init(cfg->get("exec"));
		replayer.register_sink(mocker, "exec");
	}

	// 准备回测
	replayer.prepare();
	// 执行回测
	replayer.run();

	printf("press enter key to exit\r\n");
	getchar();

	WTSLogger::stop();
}
```