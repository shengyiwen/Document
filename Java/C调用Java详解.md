### 1. C代码

```
#include <stdio.h>
#include <stdlib.h>
#include "jni.h"

JavaVM *bootJvm() {
    JavaVM *jvm;
    JNIEnv *env;

    JavaVMInitArgs jvm_args;
    JavaVMOption options[4];

    // 定制JVM启动参数，通过这种方式启动只能通过-Djava.class.path来指定classpath
    options[0].optionString = "-Djava.class.path= -Dfoo=bar";
    options[1].optionString = "-Xmx1g";
    options[2].optionString = "-Xms1g";
    options[3].optionString = "-Xmn265m";

    jvm_args.options = options;
    jvm_args.nOptions = sizeof(options) / sizeof(struct JavaVMOption);
    jvm_args.version = JNI_VERSION_1_8;
    jvm_args.ignoreUnrecognized = JNI_FALSE;

    JavaVMAttachArgs attachArgs;
    attachArgs.version = JNI_VERSION_1_8;
    attachArgs.name = "TODO";
    attachArgs.group = NULL;

    JNI_CreateJavaVM(&jvm, (void **) &env, &jvm_args);
    // 此处env对我们已经没有用了，所有detach掉
    // 否则默认情况下create完JVM，会自动将当前线程Attach上去
    (*jvm)->DetachCurrentThread(jvm);
    return jvm;
}

int main() {
    JavaVM *jvm = bootJvm();
    JNIEnv *env;

    // 将当前线程Attach上去
    if ((*jvm)->AttachCurrentThread(jvm, (void **)&env, NULL) != JNI_OK) {
        printf("AttachCurrentThread Error \n");
        exit(1);
    }

    // 获取java/lang/String、Object、Integer的类
    jclass StringClass = (*env)->FindClass(env, "java/lang/String");
    jclass ObjectClass = (*env)->FindClass(env, "java/lang/Object");
    jclass IntegerClass = (*env)->FindClass(env, "java/lang/Integer");

    // 获取String中format静态函数以及Integer的构造函数
    jmethodID formatMethod = (*env)->GetStaticMethodID(env, StringClass, "format", "(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;");
    jmethodID integerConstructor = (*env)->GetMethodID(env, IntegerClass, "<init>", "(I)V");

    // 初始化三个变量以及变量数组
    jstring arg0 = (*env)->NewStringUTF(env, "world");
    jstring arg1 = (*env)->NewStringUTF(env, "haha");
    jobject arg2 = (*env)->NewObject(env, IntegerClass, integerConstructor, 2);
    jobjectArray args = (*env)->NewObjectArray(env, 3, ObjectClass, NULL);

    // 将三个变量存放到变量数组中，并删除掉三个变量的本地引用
    (*env)->SetObjectArrayElement(env, args, 0, arg0);
    (*env)->SetObjectArrayElement(env, args, 1, arg1);
    (*env)->SetObjectArrayElement(env, args, 2, arg2);
    (*env)->DeleteLocalRef(env, arg0);
    (*env)->DeleteLocalRef(env, arg1);
    (*env)->DeleteLocalRef(env, arg2);

    // 调用java的静态方法
    jstring format = (*env)->NewStringUTF(env, "hello %s %s %d");
    jobject result = (*env)->CallStaticObjectMethod(env, StringClass, formatMethod, format, args);
    (*env)->DeleteLocalRef(env, format);

    // 判断是否发生了异常
    if ((*env)->ExceptionCheck(env)) {
        (*env)->ExceptionDescribe(env);
        printf("Exception Check \n");
        exit(-1);
    }

    // 获取结果的字符串长度，并分配相应的内存，并且将值存放到该地址
    jint resultLength = (*env)->GetStringUTFLength(env, result);
    char *cResult = malloc(resultLength + 1);
    cResult[resultLength] = 0;
    (*env)->GetStringUTFRegion(env, result, 0, resultLength, cResult);
    (*env)->DeleteLocalRef(env, result);

    printf("Java Result: %s\n", cResult);
    free(cResult);

    (*env)->DeleteLocalRef(env, args);
    if ((*jvm)->DetachCurrentThread(jvm) != JNI_OK) {
        printf("AttachCurrentThread Error\n");
        exit(1);
    }

    printf("Done\n");

    return 0;
}
```

### 2.CMake配置

此配置是MACOS下的配置，如果在不同的系统下，需要修改不同的jdk地址以及jre地址

其中project以及executable地址也要根据具体的自己项目环境配置

```
cmake_minimum_required(VERSION 3.17)
project(learning C)

set(CMAKE_C_STANDARD 11)

# 引入头文件环境变量目录，cpp或者h文件中的#include会自动搜索这些头文件
include_directories(/Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home/include)
# 引入dylib、ddl、so文件等，自动搜索该目录下的dylib so ddl文件等
link_libraries(/Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home/jre/lib/server/libjvm.dylib)
#link_directories(/Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home/jre/lib/server/libjvm.dylib)

add_executable(learning jni_invoke.c)
```