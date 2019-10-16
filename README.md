## Breakpad编译so并实现native异常捕获

#### 1.编译
 1.1 下载[breakpad](https://chromium.googlesource.com/breakpad/breakpad)</br>
 1.2  新建jni目录将下载的breakpad里面的src拷进jni目录里面</br>
 1.3 添加自己的中间代码，编译成so之后如果不处理会有比较多的头文件</br>
 头文件
 ```c
 //
// Created by TIAN FENG on 2019/10/16.
//

#ifndef BREAKPAD_DEMO_BREAKPAD_INIT_H
#define BREAKPAD_DEMO_BREAKPAD_INIT_H

void breakpad_init(const char* savePath,void(*callback)(bool isSuccess));

#endif //BREAKPAD_DEMO_BREAKPAD_INIT_H
 ```
 实现文件，注意为了统一编译方式这里使用.cc的后缀文件,此段代码在官方提供的sample_app里面可以找到，这里做了一点封装
 ```c++
#include "breakpad_init.h"
#include "src/client/linux/handler/minidump_descriptor.h"
#include "src/client/linux/handler/exception_handler.h"


void(*call)(bool isSuccess);

bool DumpCallback(const google_breakpad::MinidumpDescriptor &descriptor, void *context, bool succeeded) {
    call(succeeded);
    return succeeded;
}

void breakpad_init(const char* savePath,void(*callback)(bool isSuccess)){
    call = callback;
    google_breakpad::MinidumpDescriptor descriptor(savePath);
    static google_breakpad::ExceptionHandler eh(descriptor, NULL, DumpCallback, NULL, true, -1);
}

 ```

 1.4 将下载下来的breakpad里面android/google_breakpad/Android.mk拷贝进jni目录里面并修改
 ```mk
#LOCAL_PATH := $(call my-dir)/../..  这里是原来的 修改了文件路径这里把 /../.. 去掉
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

#名字改下 随意
LOCAL_MODULE := breakpad

LOCAL_CPP_EXTENSION := .cc
LOCAL_ARM_MODE := arm
LOCAL_SRC_FILES := \
    src/client/linux/crash_generation/crash_generation_client.cc \
    src/client/linux/dump_writer_common/thread_info.cc \
    src/client/linux/dump_writer_common/ucontext_reader.cc \
    src/client/linux/handler/exception_handler.cc \
    src/client/linux/handler/minidump_descriptor.cc \
    src/client/linux/log/log.cc \
    src/client/linux/microdump_writer/microdump_writer.cc \
    src/client/linux/minidump_writer/linux_dumper.cc \
    src/client/linux/minidump_writer/linux_ptrace_dumper.cc \
    src/client/linux/minidump_writer/minidump_writer.cc \
    src/client/minidump_file_writer.cc \
    src/common/android/breakpad_getcontext.S \
    src/common/convert_UTF.cc \
    src/common/md5.cc \
    src/common/string_conversion.cc \
    src/common/linux/elfutils.cc \
    src/common/linux/file_id.cc \
    src/common/linux/guid_creator.cc \
    src/common/linux/linux_libc_support.cc \
    src/common/linux/memory_mapped_file.cc \
    src/breakpad_init.cc \
    src/common/linux/safe_readlink.cc

LOCAL_C_INCLUDES        := $(LOCAL_PATH)/src/common/android/include \
                           $(LOCAL_PATH)/src

LOCAL_EXPORT_C_INCLUDES := $(LOCAL_C_INCLUDES)
LOCAL_EXPORT_LDLIBS     := -llog

#include $(BUILD_STATIC_LIBRARY)
#动态库
include $(BUILD_SHARED_LIBRARY)
 ```
 1.5 jni目录下添加Application.mk文件,如果不添加Application.mk某些情况下可能会编译成功，如果失败，可以根据提示搜索一下原因，然后添加相关字段
 ```mk
APP_ABI := all
APP_STL    := gnustl_static
APP_CFLAGS += -std=gnu++11
APP_OPTIM  := debug
APP_CPPFLAGS := -std=gnu++11 -D__STDC_LIMIT_MACROS
LOCAL_CPP_FEATURES += exceptions
NDK_TOOLCHAIN_VERSION := 4.9
 ```
 1.6 配置ndk环境（mac或者linux，不要在windows上编译）我这里使用的ubuntu + ndk-r13d 编译</br>
 1.7 在jni同级目录下执行ndk-build等待编译</br>
 1.8 编译可能出现的问题
 ```java
 jni/src/client/linux/minidump_writer/linux_dumper.cc: In member function 'void google_breakpad::LinuxDumper::ParseLoadedElfProgramHeaders(Elf32_Ehdr*, uintptr_t, uintptr_t*, uintptr_t*, size_t*)':
jni/src/client/linux/minidump_writer/linux_dumper.cc:698:30: error: 'UINTPTR_MAX' was not declared in this scope
   const uintptr_t max_addr = UINTPTR_MAX;
 这是工具链编译器引起的，指定APP_STL APP_CFLAGS APP_CPPFLAGS
 ```
 1.9编译结束后，会在jni同级目录下生成libs文件夹和obj文件夹，libs就是我们需要的so

 #### 2.使用
 2.1 配置studio native环境</br>
 2.2 新建native空白项目</br>
 2.3 main目录下新建jniLibs文件夹并将相应cpu的so考入</br>
 2.4 cpp目录下cpp目录下导入breakpad_init.h文件</br>
 2.4 编写CMakeLists.txt</br>
 ```cmake
 cmake_minimum_required(VERSION 3.4.1)

#自己的库
add_library(native-lib SHARED native-lib.cpp)

find_library(
        log-lib
        log)

#声明外部导入库
add_library(breakpad
        SHARED
        IMPORTED)
#设置到入库的路径
set_target_properties(breakpad
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libbreakpad.so)


target_link_libraries(
        native-lib
        breakpad
        log)
 ```
  2.4 修改build.gradle</br>
  ```gradle
  android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                cppFlags ""
                // 根据平台添加
                abiFilters "armeabi"
            }
        }
    }

  }
  ```

   2.5 编写java代码和native代码
  java
  ```java
  public class MainActivity extends Activity {

    private static String sPath = Environment.getExternalStorageDirectory().getAbsolutePath() + "/dumpPath";

    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initPath();

        initPath(sPath);

        findViewById(R.id.sample_text).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                crash();
            }
        });
    }

    private void initPath() {
        File dir = new File(sPath);
        if (!dir.exists()) {
            dir.mkdirs();
        }
    }

    private native void crash() ;


    public native void initPath(String path);

}

  ```
  native
  ```c++
  #include "breakpad_init.h"
#include <jni.h>
#include <string>
#include <android/log.h>
#define TAG "JNI_LOG" // 这个是自定义的LOG的标识
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,TAG ,__VA_ARGS__)


JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }
    return JNI_VERSION_1_6;
}

void callback(bool isSuccess){
    LOGE(" save crash %s", isSuccess ? "TRUE" : "FALSE");
}

extern "C"
JNIEXPORT void JNICALL
Java_com_breakpad_MainActivity_initPath(JNIEnv *env, jobject instance, jstring path_) {
    const char *path = env->GetStringUTFChars(path_, 0);
    breakpad_init(path,callback);
    env->ReleaseStringUTFChars(path_, path);

}


extern "C"
JNIEXPORT void JNICALL
Java_com_breakpad_MainActivity_crash(JNIEnv *env, jobject instance) {
    LOGE(" in crash ");
    int *a = NULL;
    *a = 10;
}
  ```

 运行成功后点击按钮便会将native异常信息保存到对应的文件夹，最后使用addr2line便可分析native异常位置


 ##### 第二种方式
 将src下的所有文件拷贝到jni目录下编写CMakeLists.txt，将所有相关的资源文件添加也就是Android.mk编译相关的文件，便可在studio下运行生成so，但是第一次编译时间较长，具体方式根据自身条件便可

 ##### 这里将所有so都放在了Breakpad_so 但是使用必须要有breakpad_init.h头文件