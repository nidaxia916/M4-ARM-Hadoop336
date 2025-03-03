M4-ARM-Hadoop336 : build source code to genenrate native lib

## 1. 下载hadoop336源码，解压

```sh
# 在/opt/software目录下进行编译
cd /opt/software
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6-src.tar.gz
tar -zxvf hadoop-3.3.6-src.tar.gz
cd hadoop-3.3.6-src
```

## 2.  下载编译hadoop native 相关依赖

参照hadoop源码中的BUILDINF.txt文件，找到 `Building on macOS` 相关部分

```sh
# Install Xcode Command Line Tools
xcode-select --install

# Install Homebrew
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

# Install OpenJDK 8
brew tap AdoptOpenJDK/openjdk
brew cask install adoptopenjdk8

# Install maven and tools
brew install maven autoconf automake cmake wget

# Mac 电脑自带了openssl 和 protocl ，这一点要注意一下
# Install native libraries, only openssl is required to compile native code,you may optionally install zlib, lz4, etc.
brew install zlib lz4 snappy zstd
# 参考 这个jira，https://issues.apache.org/jira/browse/HADOOP-16647， 显示hadoop 2 supports OpenSSL 1.0.2， Hadoop 3 supports OpenSSL 1.1.0 。所以这里我们需要安装1.1.0版本的openssl
# 现在openssl 1.1.0版本的openssl ，brew已经不支持了，所以需要我们手动源码安装
# 1.1 下载openssl 1.1.0版本源码，声明一下，openssl 1.1.0版本不支持mac arm 编译，需参考如下链接进行编译：https://blog.andrewmadsen.com/2020/06/22/building-openssl-for.html


# Protocol Buffers 3.7.1 (required to compile native code)
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.7.1/protobuf-java-3.7.1.tar.gz
mkdir -p protobuf-3.7 && tar zxvf protobuf-java-3.7.1.tar.gz --strip-components 1 -C protobuf-3.7
cd protobuf-3.7
./configure
make
make check
make install
protoc --version

# 执行编译命令 （绝大概率是会报错的，在hadoop-common编译时）
mvn package -Pdist,native -DskipTests -Dmaven.javadoc.skip \
-Dopenssl.prefix=/opt/openssl
```

## 3. hadoop-common相关报错

#### 3.1 Could NOT find ZLIB (missing: ZLIB_LIBRARY) (found version "1.2.12"), 详细报错信息如下

```verilog
[INFO] Running cmake /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src -DCUSTOM_OPENSSL_PREFIX=/opt/openssl -DGENERATED_JAVAH=/opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/target/native/javah -DJVM_ARCH_DATA_MODEL=64 -DREQUIRE_BZIP2=false -DREQUIRE_ISAL=false -DREQUIRE_OPENSSL=false -DREQUIRE_ZSTD=false -G Unix Makefiles
[INFO] with extra environment variables {}
[WARNING] CMake Warning (dev) in CMakeLists.txt:
[WARNING]   No project() command is present.  The top-level CMakeLists.txt file must
[WARNING]   contain a literal, direct call to the project() command.  Add a line of
[WARNING]   code such as
[WARNING]
[WARNING]     project(ProjectName)
[WARNING]
[WARNING]   near the top of the file, but after cmake_minimum_required().
[WARNING]
[WARNING]   CMake is pretending there is a "project(Project)" command on the first
[WARNING]   line.
[WARNING] This warning is for project developers.  Use -Wno-dev to suppress it.
[WARNING]
[WARNING] CMake Warning (dev) in CMakeLists.txt:
[WARNING]   cmake_minimum_required() should be called prior to this top-level project()
[WARNING]   call.  Please see the cmake-commands(7) manual for usage documentation of
[WARNING]   both commands.
[WARNING] This warning is for project developers.  Use -Wno-dev to suppress it.
[WARNING]
[WARNING] -- The C compiler identification is AppleClang 16.0.0.16000026
[WARNING] -- The CXX compiler identification is AppleClang 16.0.0.16000026
[WARNING] -- Detecting C compiler ABI info
[WARNING] -- Detecting C compiler ABI info - done
[WARNING] -- Check for working C compiler: /Library/Developer/CommandLineTools/usr/bin/cc - skipped
[WARNING] -- Detecting C compile features
[WARNING] -- Detecting C compile features - done
[WARNING] -- Detecting CXX compiler ABI info
[WARNING] -- Detecting CXX compiler ABI info - done
[WARNING] -- Check for working CXX compiler: /Library/Developer/CommandLineTools/usr/bin/c++ - skipped
[WARNING] -- Detecting CXX compile features
[WARNING] -- Detecting CXX compile features - done
[WARNING] CMake Deprecation Warning at CMakeLists.txt:23 (cmake_minimum_required):
[WARNING]   Compatibility with CMake < 3.10 will be removed from a future version of
[WARNING]   CMake.
[WARNING]
[WARNING]   Update the VERSION argument <min> value.  Or, use the <min>...<max> syntax
[WARNING]   to tell CMake that the project requires at least <min> but has been updated
[WARNING]   to work with policies introduced by <max> or earlier.
[WARNING]
[WARNING]
[WARNING] -- Performing Test CMAKE_HAVE_LIBC_PTHREAD
[WARNING] -- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
[WARNING] -- Found Threads: TRUE
[WARNING] -- Found Java: /opt/software/jdk1.8/Contents/Home/bin/java (found version "1.8.0.421")
[WARNING] -- Found JNI: /opt/software/jdk1.8/Contents/Home/include  found components: AWT JVM
[WARNING] CMake Error at /opt/homebrew/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
[WARNING]   Could NOT find ZLIB (missing: ZLIB_LIBRARY) (found version "1.2.12")
[WARNING] Call Stack (most recent call first):
[WARNING]   /opt/homebrew/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:603 (_FPHSA_FAILURE_MESSAGE)
[WARNING]   /opt/homebrew/share/cmake/Modules/FindZLIB.cmake:202 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
[WARNING]   CMakeLists.txt:47 (find_package)
```

这段日志表明在执行cmake命令时，找不到ZLIB_LIBRARY参数。

这是因为在hadoop-common项目src目录下的CMakeLists.txt文件中显示zlib是必须的，但是我们执行mvn命令时并没有提供这个参数，导致找到的是系统自带的zlib，版本是1.2.12 。不是我们通过brew进行安装的。

解决这个问题，我们需要手动指定zlib的相关路径参数。这里指出一点，传递给cmake的-D参数是有mvn -D参数传递的，具体的参数名在mvn pom文件中能看到，例如hadoop-common项目的pom文件中profile不分，关于相关参数的定义：

```shell
vim hadoop-common-project/hadoop-common/pom.xml
```

```xml
<profile>
      <id>native</id>
      <activation>
        <activeByDefault>false</activeByDefault>
      </activation>
  		<!-- 这是通过mvn -D传递的参数-->
      <properties>
        <require.bzip2>false</require.bzip2>
        <zstd.prefix></zstd.prefix>
        <zstd.lib></zstd.lib>
        <zstd.include></zstd.include>
        <require.zstd>false</require.zstd>
        <openssl.prefix></openssl.prefix>
        <openssl.lib></openssl.lib>
        <openssl.include></openssl.include>
        <require.isal>false</require.isal>
        <isal.prefix></isal.prefix>
        <isal.lib></isal.lib>
        <require.openssl>false</require.openssl>
        <runningWithNative>true</runningWithNative>
        <bundle.openssl.in.bin>false</bundle.openssl.in.bin>
        <extra.libhadoop.rpath></extra.libhadoop.rpath>
      </properties>
      <build>
        <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-enforcer-plugin</artifactId>
            <executions>
              <execution>
                <id>enforce-os</id>
                <goals>
                  <goal>enforce</goal>
                </goals>
                <configuration>
                  <rules>
                    <requireOS>
                      <family>mac</family>
                      <family>unix</family>
                      <message>native build only supported on Mac or Unix</message>
                    </requireOS>
                  </rules>
                  <fail>true</fail>
                </configuration>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-maven-plugins</artifactId>
            <executions>
              <execution>
                <id>cmake-compile</id>
                <phase>compile</phase>
                <goals><goal>cmake-compile</goal></goals>
                <configuration>
                  <source>${basedir}/src</source>
                  <!-- 这是maven 解析完传递给cmake的参数-->
                  <vars>
                    <GENERATED_JAVAH>${project.build.directory}/native/javah</GENERATED_JAVAH>
                    <JVM_ARCH_DATA_MODEL>${sun.arch.data.model}</JVM_ARCH_DATA_MODEL>
                    <REQUIRE_BZIP2>${require.bzip2}</REQUIRE_BZIP2>
                    <REQUIRE_ZSTD>${require.zstd}</REQUIRE_ZSTD>
                    <CUSTOM_ZSTD_PREFIX>${zstd.prefix}</CUSTOM_ZSTD_PREFIX>
                    <CUSTOM_ZSTD_LIB>${zstd.lib} </CUSTOM_ZSTD_LIB>
                    <CUSTOM_ZSTD_INCLUDE>${zstd.include} </CUSTOM_ZSTD_INCLUDE>
                    <REQUIRE_ISAL>${require.isal} </REQUIRE_ISAL>
                    <CUSTOM_ISAL_PREFIX>${isal.prefix} </CUSTOM_ISAL_PREFIX>
                    <CUSTOM_ISAL_LIB>${isal.lib} </CUSTOM_ISAL_LIB>
                    <REQUIRE_PMDK>${require.pmdk}</REQUIRE_PMDK>
                    <CUSTOM_PMDK_LIB>${pmdk.lib}</CUSTOM_PMDK_LIB>
                    <REQUIRE_OPENSSL>${require.openssl} </REQUIRE_OPENSSL>
                    <CUSTOM_OPENSSL_PREFIX>${openssl.prefix} </CUSTOM_OPENSSL_PREFIX>
                    <CUSTOM_OPENSSL_LIB>${openssl.lib} </CUSTOM_OPENSSL_LIB>
                    <CUSTOM_OPENSSL_INCLUDE>${openssl.include} </CUSTOM_OPENSSL_INCLUDE>
                    <EXTRA_LIBHADOOP_RPATH>${extra.libhadoop.rpath}</EXTRA_LIBHADOOP_RPATH>
                  </vars>
                </configuration>
              </execution>
              <execution>
                <id>test_bulk_crc32</id>
                <goals><goal>cmake-test</goal></goals>
                <phase>test</phase>
                <configuration>
                  <binary>${project.build.directory}/native/test_bulk_crc32</binary>
                  <timeout>1200</timeout>
                  <results>${project.build.directory}/native-results</results>
                </configuration>
              </execution>
              <execution>
                <id>erasure_code_test</id>
                <goals><goal>cmake-test</goal></goals>
                <phase>test</phase>
                <configuration>
                  <binary>${project.build.directory}/native/erasure_code_test</binary>
                  <timeout>300</timeout>
                  <results>${project.build.directory}/native-results</results>
                  <skipIfMissing>true</skipIfMissing>
                  <env>
                    <LD_LIBRARY_PATH>${LD_LIBRARY_PATH}:${isal.lib}:${isal.prefix}:/usr/lib</LD_LIBRARY_PATH>
                  </env>
                </configuration>
              </execution>
            </executions>
          </plugin>
        </plugins>
      </build>
    </profile>
```

从上面可以看出在hadoop-common项目下并没有解析zlib相关参数

我们可以打开

vim hadoop-3.3.6-src/hadoop-common-project/hadoop-common/target/native/CMakeCache.txt

搜索ZLIB，可以看到cmake 拿到的ZLIB相关参数值，是找的系统自带的zlib路径，且没有找到ZLIB_LIBRART相关

```txt
ZLIB_INCLUDE_DIR:PATH=/Library/Developer/CommandLineTools/SDKs/MacOSX15.2.sdk/usr/include
ZLIB_LIBRARY_DEBUG:FILEPATH=ZLIB_LIBRARY_DEBUG-NOTFOUND
ZLIB_LIBRARY_RELEASE:FILEPATH=ZLIB_LIBRARY_RELEASE-NOTFOUND
//ADVANCED property for variable: ZLIB_INCLUDE_DIR
ZLIB_INCLUDE_DIR-ADVANCED:INTERNAL=1
//ADVANCED property for variable: ZLIB_LIBRARY_DEBUG
ZLIB_LIBRARY_DEBUG-ADVANCED:INTERNAL=1
//ADVANCED property for variable: ZLIB_LIBRARY_RELEASE
ZLIB_LIBRARY_RELEASE-ADVANCED:INTERNAL=1
```

为了解决这个问题，我们需要手动修改hadoop-common项目下的pom文件，增加对ZLIB相关概念参数的解析。

```xml
<!-- 这是通过mvn -D传递的参数-->
<properties>
  <require.bzip2>false</require.bzip2>
  <zstd.prefix></zstd.prefix>
  <zstd.lib></zstd.lib>
  <zstd.include></zstd.include>
  <require.zstd>false</require.zstd>
  <openssl.prefix></openssl.prefix>
  <openssl.lib></openssl.lib>
  <openssl.include></openssl.include>
  <require.isal>false</require.isal>
  <isal.prefix></isal.prefix>
  <isal.lib></isal.lib>
  <require.openssl>false</require.openssl>
  <runningWithNative>true</runningWithNative>
  <bundle.openssl.in.bin>false</bundle.openssl.in.bin>
  <extra.libhadoop.rpath></extra.libhadoop.rpath>
  <!-- 新增zlib相关参数， 通过mvn -D参数来指定-->
  <require.zlib></require.zlib>  <!-- 指定zlib是必须的-->
  <zlib.include.dir></zlib.include.dir> <!-- 指定zlib头文件目录-->
  <zlib.library></zlib.library> <!--指定zlib lib目录-->
</properties>
```

```xml
<!-- 这是maven 解析完传递给cmake的参数-->
<vars>
  <GENERATED_JAVAH>${project.build.directory}/native/javah</GENERATED_JAVAH>
  <JVM_ARCH_DATA_MODEL>${sun.arch.data.model}</JVM_ARCH_DATA_MODEL>
  <REQUIRE_BZIP2>${require.bzip2}</REQUIRE_BZIP2>
  <REQUIRE_ZSTD>${require.zstd}</REQUIRE_ZSTD>
  <CUSTOM_ZSTD_PREFIX>${zstd.prefix}</CUSTOM_ZSTD_PREFIX>
  <CUSTOM_ZSTD_LIB>${zstd.lib} </CUSTOM_ZSTD_LIB>
  <CUSTOM_ZSTD_INCLUDE>${zstd.include} </CUSTOM_ZSTD_INCLUDE>
  <REQUIRE_ISAL>${require.isal} </REQUIRE_ISAL>
  <CUSTOM_ISAL_PREFIX>${isal.prefix} </CUSTOM_ISAL_PREFIX>
  <CUSTOM_ISAL_LIB>${isal.lib} </CUSTOM_ISAL_LIB>
  <REQUIRE_PMDK>${require.pmdk}</REQUIRE_PMDK>
  <CUSTOM_PMDK_LIB>${pmdk.lib}</CUSTOM_PMDK_LIB>
  <REQUIRE_OPENSSL>${require.openssl} </REQUIRE_OPENSSL>
  <CUSTOM_OPENSSL_PREFIX>${openssl.prefix} </CUSTOM_OPENSSL_PREFIX>
  <CUSTOM_OPENSSL_LIB>${openssl.lib} </CUSTOM_OPENSSL_LIB>
  <CUSTOM_OPENSSL_INCLUDE>${openssl.include} </CUSTOM_OPENSSL_INCLUDE>
  <EXTRA_LIBHADOOP_RPATH>${extra.libhadoop.rpath}</EXTRA_LIBHADOOP_RPATH>
  <!-- 新增zlib相关参数， 通过mvn -D参数来指定-->
  <REQUIRE_ZLIB>${require.zlib}</REQUIRE_ZLIB>
  <ZLIB_INCLUDE_DIR>${zlib.include.dir}</ZLIB_INCLUDE_DIR>
  <ZLIB_LIBRARY>${zlib.library}</ZLIB_LIBRARY>
</vars>
```

然后返回hadoop-3.3.6源码目录，继续执行mvn命令

```shell
mvn clean package -e -Pdist,native \
-DskipTests \
-Dmaven.javadoc.skip=true \
-DCMAKE_PREFIX_PATH=/opt/homebrew/opt/cmake \
-Dopenssl.prefix=/opt/openssl \
-Drequire.zlib=true \
-Dzlib.include.dir=/opt/homebrew/opt/zlib/include \
-Dzlib.library=/opt/homebrew/opt/zlib/lib \
-DCMAKE_OSX_ARCHITECTURES=arm64
```

#### 3.2 function-like macro '__GLIBC_PREREQ' is not defined

这个估计是因为mac os 和openssl版本的相关问题，得问问deepseek

这个问题可以在这里找到相关信息：https://issues.apache.org/jira/browse/HADOOP-19170

```
/xxxxx/hadoop/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:114:50: error: function-like macro '__GLIBC_PREREQ' is not defined
#if defined(__sun) || defined(__GLIBC_PREREQ) && __GLIBC_PREREQ(2, 32)
```

现在来修复这个问题， 参考这个**pr**：https://github.com/apache/hadoop/pull/6822/files

### 3.3 dlsym_CRYPTO_num_locks 相关问题

```verilog
WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:250:33: error: call to undeclared function 'dlsym_CRYPTO_num_locks'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   250 |   lock_cs = dlsym_CRYPTO_malloc(dlsym_CRYPTO_num_locks() *  \
[WARNING]       |                                 ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:257:3: error: call to undeclared function 'dlsym_CRYPTO_set_id_callback'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   257 |   dlsym_CRYPTO_set_id_callback((unsigned long (*)())pthreads_thread_id);
[WARNING]       |   ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:258:3: error: call to undeclared function 'dlsym_CRYPTO_set_locking_callback'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   258 |   dlsym_CRYPTO_set_locking_callback((void (*)())pthreads_locking_callback);
[WARNING]       |   ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:264:3: error: call to undeclared function 'dlsym_CRYPTO_set_locking_callback'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   264 |   dlsym_CRYPTO_set_locking_callback(NULL);
[WARNING]       |   ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:266:19: error: call to undeclared function 'dlsym_CRYPTO_num_locks'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   266 |   for (i = 0; i < dlsym_CRYPTO_num_locks(); i++) {
[WARNING]       |                   ^
[WARNING] 5 errors generated.
[WARNING] make[2]: *** [CMakeFiles/hadoop_static.dir/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c.o] Error 1
[WARNING] make[2]: *** Waiting for unfinished jobs....
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:250:33: error: call to undeclared function 'dlsym_CRYPTO_num_locks'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   250 |   lock_cs = dlsym_CRYPTO_malloc(dlsym_CRYPTO_num_locks() *  \
[WARNING]       |                                 ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:257:3: error: call to undeclared function 'dlsym_CRYPTO_set_id_callback'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   257 |   dlsym_CRYPTO_set_id_callback((unsigned long (*)())pthreads_thread_id);
[WARNING]       |   ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:258:3: error: call to undeclared function 'dlsym_CRYPTO_set_locking_callback'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   258 |   dlsym_CRYPTO_set_locking_callback((void (*)())pthreads_locking_callback);
[WARNING]       |   ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:264:3: error: call to undeclared function 'dlsym_CRYPTO_set_locking_callback'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   264 |   dlsym_CRYPTO_set_locking_callback(NULL);
[WARNING]       |   ^
[WARNING] /opt/software/hadoop-3.3.6-src/hadoop-common-project/hadoop-common/src/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c:266:19: error: call to undeclared function 'dlsym_CRYPTO_num_locks'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
[WARNING]   266 |   for (i = 0; i < dlsym_CRYPTO_num_locks(); i++) {
[WARNING]       |                   ^
[WARNING] 5 errors generated.
[WARNING] make[2]: *** [CMakeFiles/hadoop.dir/main/native/src/org/apache/hadoop/crypto/random/OpensslSecureRandom.c.o] Error 1
[WARNING] make[2]: *** Waiting for unfinished jobs....
[WARNING] make[1]: *** [CMakeFiles/hadoop_static.dir/all] Error 2
[WARNING] make[1]: *** Waiting for unfinished jobs....
[WARNING] make[1]: *** [CMakeFiles/hadoop.dir/all] Error 2
[WARNING] make: *** [all] Error 2
```

我手动编译安装了openssl 1.1.0版本，但是还是出现了这个问题，还得请教一下deepseek
