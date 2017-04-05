## Bazel理解及构建测试
--------------------------------
### 1. Bazel 简介
Bazel是一个构建工具，即一个可以运行编译和测试来组装软件的工具，跟Make、Ant、Gradle、Buck、Pants和Maven一样。
### 2. Bazel特性
>支持多语言（java、object-c、c++）、可扩展用来支持任意编程语言；
多平台的构建；
重现性（reproducibility），BUILD文件中，每个库，测试程序，二进制文件必须明确完整地指定直接依赖。当修改源代码文件后，Bazel使用这个依赖信息就可以知道哪些必须重新构建，哪些任务可以并行执行。这意味者所有的构建都是增量形式的并能够每次都生成相同的结果。
伸缩性[Scalability]：Bazel可以处理巨大的构建；在Google，一个服务器端程序超过100k的源码是常有的事情，如果没有文件被改动，构建过程需要大约200ms

### 3. Bazel同make、ant等比较
> Make,Ninja: 通过这些工具都能够控制执行哪些命令来构建文件，但是需要用户书写正确的规则。
用户跟Bazel在更高级别上交互。例如，它有内置的"Java test", "C++ binary"的规则[rule]，有例如“目标平台”[target platform],“主机平台"[host platform]这种标记。这些规则都经历了充分的测试，不会出错。
Ant和Maven：Ant和Maven主要是面向Java，而Bazel可以处理多种语言。Bazel鼓励把代码库的内容划分成小的，可复用的单元，并且只重新构建需要重新构建的文件。这会提高在庞大的代码库上开发的速度。

### 4. 为什么用 Bazel
> Bazel可以成倍提高构建速度，因为它只重新编译需要重新编译的文件。类似的，它会跳过没有被改变的测试。
Bazel产出确定的结果。这消除了增量和干净构建，开发机器和持续集成之间的构建结果的差异。
Bazel可以使用同一个工程下的相同的工具来构建不同的客户端和服务器端应用程序。例如，你可以在一次提交里修改一个客户端/服务器协议，然后测试更新后的手机程序和服务器端程序能够正常工作，构建时使用的是同样的工具，利用的都是上面提到的Bazel的特性。

### 5. Bazel构建java示例
Bazel构建文件目录必须包含WORKSPACE文件

**Bazel构建java示例**
创建示例工程目录~/gitroot/my-project-java/，并创建一个空的WORKSPACE文件，用来告诉Bazel当前工程的根目录位置。
- **创建一个java示例工程：**
  ```sh
  # If you're not already there, move to your workspace directory.
  cd ~/gitroot/my-project-java
  mkdir -p src/main/java/com/example
  cat > src/main/java/com/example/ProjectRunner.java <<'EOF'
  package com.example;

  public class ProjectRunner {
      public static void main(String args[]) {
          Greeting.sayHi();
      }
  }
  EOF

  cat > src/main/java/com/example/Greeting.java <<'EOF'
  package com.example;

  public class Greeting {
      public static void sayHi() {
          System.out.println("Hi!");
      }
  }
  EOF
  ```
- **在根目录下创建~/gitroot/my-project-java/BUILD文件：**
  ```sh
  # ~/gitroot/my-project-java/BUILD
  java_binary(
      name = "my-runner",
      srcs = glob(["**/*.java"]),
      main_class = "com.example.ProjectRunner",
  )
  ```
- **Bazel构建java二进制文件**
  ```sh
  cd ~/gitroot/my-project-java
  bazel build //:my-runner
  ```
  构建完成后运行如下命令：
  ```sh
  bazel-bin/my-runner
  # you will see
  Hi!
  ```
- **添加依赖**
  当工程文件很大的时候，有必要细分成多个独立的库文件，通过添加依赖的方法，当部分文件修改时避免重新编译整个工程，因此可以对     BUILD文件进行如下修改：
  ```sh
  java_binary(
      name = "my-other-runner",
      srcs = ["src/main/java/com/example/ProjectRunner.java"],
      main_class = "com.example.ProjectRunner",
      deps = [":greeter"],
  )
  
  java_library(
      name = "greeter",
      srcs = ["src/main/java/com/example/Greeting.java"],
  )
  ```
  这样Bazel就会首先构建Greeter库，再构建my-other-runner
  ```sh
  bazel build //:my-other-runner
  ```
- **工程中包含多个package包**
  对于大型的工程，往往包含了多个package包，根据BUILD中//path/to/directory:target-name语法去编译。比如，该示例中src/main/java  /com/example/包含了一个cmdline的子目录：
  ```sh
  mkdir -p src/main/java/com/example/cmdline
  cat > src/main/java/com/example/cmdline/Runner.java <<'EOF'
  package com.example.cmdline;
  
  import com.example.Greeting;
  
  public class Runner {
      public static void main(String args[]) {
          Greeting.sayHi();
      }
  }
  EOF
  ```
  此时，Runner.java依赖于com.example.Greeting，因此该子目录下的BUILD文件为：
  ```sh
  # ~/gitroot/my-project-java/src/main/java/com/example/cmdline/BUILD
  java_binary(
      name = "runner",
      srcs = ["Runner.java"],
      main_class = "com.example.cmdline.Runner",
      deps = ["//:greeter"]
  )
  ```
  同时修改~/gitroot/my-project-java/BUILD文件中的java_library的visibility，使得com.example.Greeting对com.example..cmdline.ru   nner可见：
  ```sh
  java_library(
      name = "greeter",
      srcs = ["src/main/java/com/example/Greeting.java"],
      visibility = ["//src/main/java/com/example/cmdline:__pkg__"],
  )
  ```
  然后构建运行即可：
  ```sh
  bazel run //src/main/java/com/example/cmdline:runner
  ```
- **部署 .jar**
  当前bazel-bin/src/main/java/com/example/cmdline/runner.jar文件内容只包含Runner.class，而不包含依赖库Greeting.class：
  ```sh
  jar tf bazel-bin/src/main/java/com/example/cmdline/runner.jar
  # you will see
  META-INF/
  META-INF/MANIFEST.MF
  com/
  com/example/
  com/example/cmdline/
  com/example/cmdline/Runner.class
  ```
   因此该 .jar 文件只能在本地运行，不能部署到其他机器上，用以下命令即可：
   ```sh
   bazel build //src/main/java/com/example/cmdline:runner_deploy.jar # more generally, <target-name>_deploy.jar
   ```

### 6. Bazel构建c++示例
- **同理，创建工程目录，以及WORKSPACE文件**
  当前工程目录的文件结构大致为：
  ```
  └── my-project
    ├── lib
    │   ├── BUILD
    │   ├── hello-greet.cc
    │   └── hello-greet.h
    ├── main
    │   ├── BUILD
    │   ├── hello-time.cc
    │   ├── hello-time.h
    │   └── hello-world.cc
    └── WORKSPACE
  ```
- **创建工程所需源文件**
  ```sh
    # If you're not already there, move to your workspace directory.
    cd ~/gitroot/my-project-cpp
    mkdir ./main
    cat > main/hello-world.cc <<'EOF'
    
    #include "lib/hello-greet.h"
    #include "main/hello-time.h"
    #include <iostream>
    #include <string>
    
    int main(int argc, char** argv) {
      std::string who = "world";
      if (argc > 1) {
        who = argv[1];
      }
      std::cout << get_greet(who) <<std::endl;
      print_localtime();
      return 0;
    }
    EOF
    
    cat > main/hello-time.h <<'EOF'
    
    #ifndef MAIN_HELLO_TIME_H_
    #define MAIN_HELLO_TIME_H_
    
    void print_localtime();
    
    #endif
    EOF
    
    cat > main/hello-time.cc <<'EOF'
    
    #include "main/hello-time.h"
    #include <ctime>
    #include <iostream>
    
    void print_localtime() {
      std::time_t result = std::time(nullptr);
      std::cout << std::asctime(std::localtime(&result));
    }
    EOF
    
    mkdir ./lib
    cat > lib/hello-greet.h <<'EOF'
    
    #ifndef LIB_HELLO_GREET_H_
    #define LIB_HELLO_GREET_H_
    
    #include <string>
    
    std::string get_greet(const std::string &thing);
    
    #endif
    EOF
    
    cat > lib/hello-greet.cc <<'EOF'
    
    #include "lib/hello-greet.h"
    #include <string>
    
    std::string get_greet(const std::string& who) {
      return "Hello " + who;
    }
    EOF
  ```
- **添加各所需BUILD文件**
  lib/BUILD:
  ```sh
  cc_library(
      name = "hello-greet",
      srcs = ["hello-greet.cc"],
      hdrs = ["hello-greet.h"],
      visibility = ["//main:__pkg__"],
  )
  ```
  main/BUILD
  ```sh
  cc_library(
      name = "hello-time",
      srcs = ["hello-time.cc"],
      hdrs = ["hello-time.h"],
  )

  cc_binary(
      name = "hello-world",
      srcs = ["hello-world.cc"],
      deps = [
          ":hello-time",
          "//lib:hello-greet",
      ],
  )
  ```
- **构建hello-world**
  ```sh
  bazel build main:hello-world
  ```
- **运行示例**
  ```sh
  ./bazel-bin/main/hello-world
  #Then you will see
  Hello world
  Wed Apr  5 14:33:08 2017
  
  #or you can run with a command parameter
  ./bazel-bin/main/hello-world bazel
  #Then you will see
  Hello bazel
  Wed Apr  5 14:34:18 2017
  ```
- **添加include包含路径**
  此时，文件目录结构类似于 ：
  ```
  └── my-project
    ├── third_party
    │   └── some_lib
    │       ├── BUILD
    │       ├── include
    │       │   └── some_lib.h
    │       └── some_lib.cc
    └── WORKSPACE
  ```
  third_party/some_lib/BUILD的内容为：
  ```sh
  cc_library(
      name = "some_lib",
      srcs = ["some_lib.cc"],
      hdrs = ["some_lib.h"],
      copts = ["-Ithird_party/some_lib"],
  )
  ```
- **添加外部依赖库**
  需要在WORKSPACE文件中添加外部依赖库的repository，以Google Test举例，使得新的知识库对当前工程有效
  ```sh
  new_http_archive(
      name = "gtest",
      url = "https://github.com/google/googletest/archive/release-1.7.0.zip",
      sha256 = "b58cb7547a28b2c718d1e38aee18a3659c9e3ff52440297e965f5edffe34b6d0",
      build_file = "gtest.BUILD",
      strip_prefix = "googletest-release-1.7.0",
  )
  ```
  这个时候需要在工程根目录下创建gtest.BUILD文件，用来编译Google Test:
  ```sh
  cc_library(
      name = "main",
      srcs = glob(
          ["src/*.cc"],
          exclude = ["src/gtest-all.cc"]
      ),
      hdrs = glob([
          "include/**/*.h",
          "src/*.h"
      ]),
      copts = ["-Iexternal/gtest/include"],
      linkopts = ["-pthread"],
      visibility = ["//visibility:public"],
  )
  ```
  然后，我们创建一个test子目录用来测试外部依赖库的有效性：
  ./test/hello-test.cc：
  ```sh
  #include "gtest/gtest.h"
  #include "lib/hello-greet.h"
  
  TEST(HelloTest, GetGreet) {
    EXPECT_EQ(get_greet("Bazel"), "Hello Bazel");
  }
  ```
  ./test/BUILD:
  ```sh
  cc_test(
      name = "hello-test",
      srcs = ["hello-test.cc"],
      copts = ["-Iexternal/gtest/include"],
      deps = [
          "@gtest//:main",
          "//lib:hello-greet",
      ],
  )
  ```
  这里，注意到./test/hello-test.cc依赖//lib:hello-greet，因此需要在lib/BUILD添加对其可见性："//test:__pkg__",         加入到visibility属性中
