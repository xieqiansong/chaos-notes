# Mysql UDF 实现sm4加解密（mysql 插件)

> 场景：业务侧希望直接在 SQL 里对字段做国密 SM4 加解密（如脱敏存储、存量数据迁移），又不想改应用层代码、或数据库本身不支持 SM4 函数时，用 C 写个 UDF 插件挂到 MySQL，提供 `encrypt_sm4()/decrypt_sm4()` 两个函数即可。
>
> 版本：`MySQL 5.7` + C UDF（`mysql.h` 接口），SM4 实现引用 [NEWPLAN/SMx](https://github.com/NEWPLAN/SMx/tree/master/SM4/Windows/SM4/src) 的 `sm4.c`（ECB 模式）。

先说结论：

- **UDF 是「函数即 .so/.dll」**：编译出动态库丢进 `plugin_dir`，`CREATE FUNCTION ... SONAME` 注册即可，无需重启 MySQL。
- **密钥写死在代码里只是 demo**：示例 `key = "0000000000000000"`，真实场景应外置（配置表/环境变量/由传入参数决定），否则等于没加密。
- **ECB 模式不安全**：同明文同密文，仅适合学习；生产换 CBC/GCM 并带随机 IV。
- **长度是最容易翻车的地方**：`args->args[0]` 在一次 SQL 执行中是复用的缓冲区，必须用 `args->lengths[0]` 显式拷贝，且出参 `max_length` 要给足（示例 65535）。

## 1. 插件源码

udf_sm4.c:

```cpp
/**
 * mysql udf sm4
 *
 * @author xieqiansong
 * @date 2024-05-26
 */
#include <mysql.h>
#include <mysql_com.h>
#include <stdlib.h>
#include <string.h>
 
#include "sm4.c" // https://github.com/NEWPLAN/SMx/tree/master/SM4/Windows/SM4/src
 
my_bool encrypt_sm4_init(UDF_INIT *initid, UDF_ARGS *args, char *message);
 
my_bool decrypt_sm4_init(UDF_INIT *initid, UDF_ARGS *args, char *message);
 
char *encrypt_sm4(UDF_INIT *initid, UDF_ARGS *args, char *result, unsigned long *length, char *is_null, char *error);
 
char *decrypt_sm4(UDF_INIT *initid, UDF_ARGS *args, char *result, unsigned long *length, char *is_null, char *error);
 
void encrypt_sm4_deinit(UDF_INIT *initid);
 
void decrypt_sm4_deinit(UDF_INIT *initid);
 
my_bool encrypt_sm4_init(UDF_INIT *initid, UDF_ARGS *args, char *message) {
    // 检查参数个数
    if (args->arg_count != 1) {
        strcpy(message, "wrong number of arguments: encrypt_sm4() requires one argument");
        return 1;
    }
 
    // 检查参数类型
    if (args->arg_type[0] != STRING_RESULT) {
        strcpy(message, "encrypt_sm4() requires a string argument");
        return 1;
    }
 
    initid->maybe_null = 1;
    initid->max_length = 65535;
    return 0;
 
}
 
my_bool decrypt_sm4_init(UDF_INIT *initid, UDF_ARGS *args, char *message) {
    // 检查参数个数
    if (args->arg_count != 1) {
        strcpy(message, "wrong number of arguments: decrypt_sm4() requires one argument");
        return 1;
    }
    // 检查参数类型
    if (args->arg_type[0] != STRING_RESULT) {
        strcpy(message, "decrypt_sm4() requires a string argument");
        return 1;
    }
 
    initid->maybe_null = 1;
    initid->max_length = 65535;
    return 0;
}
 
 
char *do_crypt(UDF_INIT *inisid, UDF_ARGS *args, char *result, unsigned long *length, char *is_null, char *error,
               int mode) {
    if (args->args[0] == NULL) {
        *is_null = 1;
        return NULL;
    }
    // 入参以及入参长度
    char *text = args->args[0];
    int input_length = args->lengths[0];
 
    // 构建加解密入参和出参
    int max_len = input_length + 20;
    char *input = (char *) malloc(max_len * sizeof(char));
    char *output = (char *) malloc(max_len * sizeof(char));
 
    if (input == NULL || output == NULL) {
        *is_null = 1;
        return NULL;
    }
    memset(input, 0, max_len * sizeof(char));
    memset(output, 0, max_len * sizeof(char));
 
    // args->args[0] 执行一次sql时是复用的，所以必须通过 input_length 来复制
    memcpy(input, text, input_length);
 
    // --------------------------------  加解密  Start --------------------------------
    sm4_context ctx;
    char *key = "0000000000000000"; // 密钥好像要求16个字符
    int encrypt_len = (input_length + 15) & ~15;
 
    if (mode == 1) {
        sm4_setkey_enc(&ctx, (unsigned char *) key);
        sm4_crypt_ecb(&ctx, mode, encrypt_len, (unsigned char *) input, (unsigned char *) output);
        *length = encrypt_len;
    } else {
        sm4_setkey_dec(&ctx, (unsigned char *) key);
        sm4_crypt_ecb(&ctx, mode, encrypt_len, (unsigned char *) input, (unsigned char *) output);
        *length = strlen(output);
    }
    // --------------------------------  加解密 End --------------------------------
 
    // *length 出参长度
    // *length = mode == 1 ? encrypt_len : strlen(output);
    *is_null = 0;
    return output;
}
 
char *encrypt_sm4(UDF_INIT *initid, UDF_ARGS *args, char *result, unsigned long *length, char *is_null, char *error) {
    return do_crypt(initid, args, result, length, is_null, error, 1);
}
 
char *decrypt_sm4(UDF_INIT *initid, UDF_ARGS *args, char *result, unsigned long *length, char *is_null, char *error) {
    return do_crypt(initid, args, result, length, is_null, error, 0);
}
 
void encrypt_sm4_deinit(UDF_INIT *initid) {
}
 
void decrypt_sm4_deinit(UDF_INIT *initid) {
}
```

## 2. 编译（CMake / gcc）

CMakeLists.txt:

```bash
cmake_minimum_required(VERSION 3.27)
project(mysql_udf C)
set(CMAKE_C_STANDARD 11)
 
# 添加MySQL头文件夹路径（替换为本机 MySQL include 目录）
include_directories(<MYSQL_INCLUDE_DIR>)
 
# 添加库文件并打包为DLL
add_library(mysql_udf SHARED udf_sm4.c)
 
# 在 Windows 平台上添加额外属性
if(WIN32)
    set_target_properties(mysql_udf PROPERTIES
            WIN32_EXECUTABLE TRUE
            EXPORT_NAME mysql_udf)
endif()
```

Windows 下我用的 Clion 编译；Linux 直接 `g++` 一步出 `.so`：

```bash
g++ -I /usr/include/mysql -shared -fPIC -o udf_sm4.so udf_sm4.c
cp udf_sm4.so /usr/lib64/mysql/plugin/
```

## 3. 注册与验证

把编译出的 `dll`/`so` 复制到 `plugin_dir`，再用 `CREATE FUNCTION ... SONAME` 注册（**无需重启 MySQL**）：

```sql
 
-- 复制到这个目录下
show variables like "%plugin_dir%";
 
DROP FUNCTION IF EXISTS encrypt_sm4;
DROP FUNCTION IF EXISTS decrypt_sm4;
CREATE FUNCTION encrypt_sm4 RETURNS string SONAME 'libmysql_udf.dll';
CREATE FUNCTION decrypt_sm4 RETURNS string SONAME 'libmysql_udf.dll';
 
 
-- 测试UDF 没有结果应该就是成功了
select description,
       encrypt_sm4(description)                            en_description,
       length(encrypt_sm4(description))                    en_description_len,
       decrypt_sm4(encrypt_sm4(description))               de_description,
       cast(decrypt_sm4(encrypt_sm4(description)) as char) de_description_char,
       length(decrypt_sm4(encrypt_sm4(description)))       de_description_len
from mysql.help_topic
where description != cast(decrypt_sm4(encrypt_sm4(description)) as char);
 
```

## 几个坑

- **编译前置**：机器上必须已装 MySQL（要引入 `mysql.h` 头文件），`include_directories` 指向本机 include 目录（Windows 用 CMake，Linux 用 `-I /usr/include/mysql`）。
- **入参/出参长度**：`args->args[0]` 在一次 SQL 执行中是复用缓冲区，必须用 `args->lengths[0]` 显式 `memcpy`，否则串数据；出参 `initid->max_length` 要给足（示例 65535），否则长字段被截断。
- **密钥写死**：示例 `key` 硬编码在 C 里，仅演示；生产应外置或作为参数传入，否则等于明文。
- **ECB 局限**：同明文出同密文、无 IV，不适合直接保护敏感数据；生产换带 IV 的分组模式（CBC/GCM）。
- **`deinit` 空实现**：示例 `deinit` 没释放 `malloc` 的内存（`do_crypt` 里 `input/output` 每次调用都分配却未释放），反复调用会内存泄漏，生产需补 `free`。

## 参考来源

- [NEWPLAN/SMx — SM4 实现](https://github.com/NEWPLAN/SMx/tree/master/SM4/Windows/SM4/src)（示例中 `sm4.c` 来源）
- MySQL 官方文档：Adding a User-Defined Function（UDF）
