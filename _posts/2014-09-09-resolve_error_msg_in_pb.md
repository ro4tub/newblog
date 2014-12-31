---
layout: post
title: 解决PB库打印的错误日志
---

后台程序时不时在标准错误打印如下错误日志:

	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when serializing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when serializing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when parsing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when parsing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when serializing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when serializing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when parsing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.
	[libprotobuf ERROR google/protobuf/wire_format.cc:1053] String field contains invalid UTF-8 data when parsing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes.

打开wire_format.cc:1053

	void WireFormat::VerifyUTF8StringFallback(const char* data, int size, Operation op) {
	if (!IsStructurallyValidUTF8(data, size)) {
	    const char* operation_str = NULL;
	    switch (op) {
	      case PARSE:
	        operation_str = "parsing";
	        break;
	      case SERIALIZE:
	        operation_str = "serializing";
	        break;
	      // no default case: have the compiler warn if a case is not covered.
	    }
	    GOOGLE_LOG(ERROR) << "String field contains invalid UTF-8 data when "
	               << operation_str
	               << " a protocol buffer. Use the 'bytes' type if you intend to "
	                  "send raw bytes.";
	  } 
	} 


调用VerifyUTF8StringFallback的地方只有:

	inline void WireFormat::VerifyUTF8String(const char* data, int size,
	    WireFormat::Operation op) {
	#ifdef GOOGLE_PROTOBUF_UTF8_VALIDATION_ENABLED
	  WireFormat::VerifyUTF8StringFallback(data, size, op);
	#else 
	  // Avoid the compiler warning about unsued variables.
	  (void)data; (void)size; (void)op;
	#endif
	}

而调用VerifyUTF8String的地方很多，非lite版本的proto生成文件，对于每个string类型字段都会调用VerifyUTF8String。

所以想在日志输出的地方，打印函数调用堆栈。查看GOOGLE_LOG宏，发现在LogMessage::Finish中实现比较好，代码如下：

	void LogMessage::Finish() {    
	  bool suppress = false;       
	              
	  if (level_ != LOGLEVEL_FATAL) { 
	    InitLogSilencerCountOnce();
	    MutexLock lock(log_silencer_count_mutex_);
	    suppress = log_silencer_count_ > 0;
	  }           
	              
	  if (!suppress) {
	    log_handler_(level_, filename_, line_, message_);
	//#ifndef NDEBUG
	    if (level_ >= LOGLEVEL_ERROR) { 
	        // print stack added by HKF     
	        int j, nptrs;
	#define SIZE 100
	        void *buffer[100];
	        char **strings;
	              
	        nptrs = backtrace(buffer, SIZE);
	              
	        /* The call backtrace_symbols_fd(buffer, nptrs, STDOUT_FILENO)
	         *               would produce similar output to the following: */
	              
	        strings = backtrace_symbols(buffer, nptrs);
	        if (strings == NULL) {
	            perror("backtrace_symbols");    
	        }     
	              
	        string result;
	        for (j = 0; j < nptrs; j++) {   
	              
	            result.append(strings[j]).append("\n");
	        }     
	        log_handler_(level_, filename_, line_, result.c_str());
	        free(strings);
	    }         
	//#endif      
	              
	  }           
	  if (level_ == LOGLEVEL_FATAL) { 
	#if PROTOBUF_USE_EXCEPTIONS    
	    throw FatalException(filename_, line_, message_);
	#else         
	    abort();  
	#endif
	  }
	}


编译pb库，又出现-fPIC错误:

	libprotobuf.a(common.o): relocation R_X86_64_32S against `std::basic_string<char, std::char_traits<char>, std::allocator<char> >::_Rep::_S_empty_rep_storage' can not be used when making a shared object; recompile with -fPIC

修改configure文件，对于CFLAGS和CXXFLAGS默认设置-fPIC（可以grep ac_cv_env_CFLAGS_set和ac_cv_env_CXXFLAGS_set）。再执行./configure --disable-shared && make -j4

测试通过，可以成功看到堆栈。
Joel Spolsky曾说：
> All non-trivial abstraction, to some degree, are leaky.
