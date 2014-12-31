---
layout: post
title: Google开源基准测试库benchmark
---

最早看到google benchmark是从Jeff Dean分享的PPT里，针对Protobuf做的基准测试。
后来在golang的testing包又看到, golang中的benchmark功能和google benchmark库的功能一脉相承。

题外话，golang中profiler工具/内部内存池都跟google现有的成熟库一脉相承的。

Google benchmark库可以在[这里下载](https://github.com/google/benchmark)

安装的时候可能会碰到下载gtest出错，可以修改CMakeLists.txt中googletest的URL，替换成本地已经下载的gtest-1.7.0.zip。

google benchmark支持的特性:

1. 容易跟gtest整合
2. 自定义参数
3. 暂停/恢复统计，剔除初始化数据的构建
4. 支持多线程
5. 可以选择性地执行基准测试 

##简单的例子

测试std::string创建和复制的性能:

	static void BM_StringCreation(benchmark::State& state) {
	while(state.KeepRunning())
		std::string empty_string;
	}
	
	BENCHMARK(BM_StringCreation);
	
	static void BM_StringCopy(benchmark::State& state) {
		std::string x = "hello";
  		while (state.KeepRunning())
    	std::string copy(x);
	}
	BENCHMARK(BM_StringCopy);
	
	int main(int arg, const char* argv[]) {
		benchmark::Initialize(&argc, argv);
		benchmark::RunSpecifiedBenchmarks();
		return 0;
	}

通过命令行参数指定执行匹配的基准测试，例如:

默认执行所有的基准测试 ./my_unittest

等同于 ./my_unittest --benchmark_filter=all

指定运行BM_StringCreation ./my_unittest --benchmark_filter=BM_StringCreation

也可以使用正则匹配 ./my_unittest --benchmark_filter='Copy|Creation'

##测试参数

有时在一个函数中实现一组基准测试，通过额外的参数来指定测试内容。比如说，下面代码定义了一组针对不同长度memcpy()性能的测试:

	static void BM_memcpy(benchmark::State& state) {
		char *src = new char[state.range_x()];
		char *dst = new char[state.range_x()];
		memset(src, 'x', state.range_x());
		while(state.KeepRunning())
			memcpy(dst, src, state.range_x());
		state.SetBytesProcessed(int64_t(state.iterations)*int64_t(state.range_x()));
		delete[] src;
		delete[] dst;
	}
	BENCHMARK(BM_memcpy)->Arg(8)->Arg(64)->Arg(512)->Arg(1<<10)->Arg(8<<10);
	
上面代码中指定参数的方式不够优雅，可以用Range替代，会从区间中选一系列参数作为基准测试的参数。
	
	BENCHMARK(BM_memcpy)->Range(8, 8<<10);
	
如果测试依赖两个参数呢？

例如，下面代码定义一组针对set插入速度的测试。

	static void BM_SetInsert(benchmark::State& state) {
		while(state.KeepRunning()) {
			state.PauseTiming();
			std::set<int> data = ConstructRandomSet(state.range_x());
			state.ResumeTiming();
			for(int j = 0; j < state.rangeY; ++j) {
				data.insert(RandomNumber());
			}
		}
	}
	BENCHMARK(BM_SetInsert)
		->ArgPair(1<<10, 1)
		->ArgPair(1<<10, 8)
		->ArgPair(1<<10, 64)
		->ArgPair(1<<10, 512)
		->ArgPair(8<<10, 1)
		->ArgPair(8<<10, 8)
		->ArgPair(8<<10, 64)
		->ArgPair(8<<10, 512);

和上面一样，指定参数的方式不够优雅，可以用RangePair替代，RangePair会从两个指定的区间中选择合适的参数，产生一组基准测试的参数。

	BENCHMARK(BM_SetInsert)->RangePair(1<<10, 8<<10, 1, 512);
	

对于一些更复杂参数呢？

可以定义一个自定义函数，如下：

	static benchmark::internal::Benchmark* CustomArguments(benchmark::internal::Benchmark* b) {
		for(int i = 0; i <= 10; ++i) {
			for(int j = 32; j <= 1024*1024; j *= 8) {
				b = b->ArgPair(i, j);
			}
		}
		return b;
	}
	BENCHMARK(BM_SetInsert)->Apply(CustomArguments);
	
## 模板化的测试

模板化的微基准以相同方式工作的, 下面代码测试WaitQueue的吞吐量:

	template<class Q> int BM_Sequential(benchmark::State& state) {
		Q q;
		typename Q::value_type v;
		while(state.KeepRunning()) {
			for(int i = state.range_x(); i--;)
				q.push(v);
			for(int e = state.range_x(); e--;)
				q.Wait(&v);
		}
		
		state.SetBytesProcessed(static_cast<int64_t>(state.iterations())*state.range_x());
	}
	

	BENCHMARK_TEMPLATE(BM_Sequential, WaitQueue<int>)->Range(1<<0, 1<<10);

## 多线程
在多线程测试中，benchmark库能保证等所有线程都调用了KeepRunning()才开始，直到KeepRunning()返回false才结束。全局的setup或者teardown，可以通过线程索引来识别。


	static void BM_MultiThreaded(benchmar::State& state) {
		if(state.thread_index == 0) {
			// setup代码
		}
		while(state.KeepRunning()) {
			// 测试代码
		}
		if(state.thread_index == 0) {
			// teardown
		}
	}
	BENCHMARK(BM_MultiThreaded)->Threads(2);