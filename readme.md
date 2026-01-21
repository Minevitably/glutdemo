# 在vscode中使用vcpkg+cmake配置freeglut项目

[微软官方文档](https://learn.microsoft.com/en-us/vcpkg/get_started/get-started-vscode?pivots=shell-powershell)

[源码](https://github.com/Minevitably/glutdemo)

# 1 使用vcpkg安装freeglut

安装`freeglut`（这一步可有可无，毕竟项目编译时本就会从`vcpkg`源拉取`freeglut`源码并在`build`目录下进行编译，这里保留该步骤仅用于展示下述两种配置方案）

```bash
vcpkg install freeglut
```

![image](assets/image-20260119171215-ih0jor6.png)

在这里可以看到`freeglut`​提供两种`cmake`项目的配置方案

第一种为默认配置

```cmake
find_package(FreeGLUT CONFIG REQUIRED)
target_link_libraries(main PRIVATE $<IF:$<TARGET_EXISTS:FreeGLUT::freeglut>,FreeGLUT::freeglut,FreeGLUT::freeglut_static>)
```

第二种为兼容模式

```cmake
find_package(GLUT REQUIRED)
target_link_libraries(main PRIVATE GLUT::GLUT)
```

# 2 配置方案

## 方案1 使用默认配置

按照文档新建空项目`glutdemo`

![image](assets/image-20260119172546-oy402wk.png)

在项目根目录执行

```bash
vcpkg new --application
```

根目录下会生成一个清单文件（包含项目中要用到的库，默认为空）`vcpkg.json`​，以及一个包管理器配置文件（配置包的来源，以及版本锁定，方便团队协作）`vcpkg-configuration.json`

通过`vcpkg-configuration.json`​配置后的项目即使在完全更换开发环境后（不同版本的`vcpkg`，不同的操作系统）也能确保其他人下载这些代码到本地后编译时使用的包和发布者创建项目时用的包版本相同。

项目使用的包完全存放在`glutdemo/build`​下，而不是使用`vcpkg`全局安装的包，能有效避免版本冲突，起到环境隔离的作用。

‍

声明项目用到的`freeglut`依赖

```bash
vcpkg add port freeglut
```

此时`vcpkg.json`会发生变化

```json
{
  "dependencies": [
    "freeglut"
  ]
}
```

创建`CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.10)

project(glutdemo)

set(CMAKE_CXX_STANDARD 17)

find_package(OpenGL REQUIRED)
find_package(FreeGLUT CONFIG REQUIRED)

add_executable(${PROJECT_NAME} main.cpp)


target_link_libraries(${PROJECT_NAME} PRIVATE
        OpenGL::GL
        $<IF:$<TARGET_EXISTS:FreeGLUT::freeglut>,FreeGLUT::freeglut,FreeGLUT::freeglut_static>

)
```

添加源文件`main.cpp`

```cmake
#include <GL/freeglut.h>

// Display callback function
void display() {
    glClear(GL_COLOR_BUFFER_BIT);
    
    // Draw a red triangle
    glBegin(GL_TRIANGLES);
    glColor3f(1.0f, 0.0f, 0.0f);  // Red
    glVertex2f(-0.5f, -0.5f);
    glVertex2f(0.5f, -0.5f);
    glVertex2f(0.0f, 0.5f);
    glEnd();
    
    glutSwapBuffers();
}

// Main function
int main(int argc, char** argv) {
    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB);
    glutInitWindowSize(400, 400);
    glutCreateWindow("FreeGLUT Test");
    
    // Set background color to black
    glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    
    glutDisplayFunc(display);
    glutMainLoop();
    
    return 0;
}
```

为了使`cmake`​识别`vcpkg`​还需要创建`CMakePresets.json`​设置`vcpkg`工具链文件

> 这里如果系统用的不是`msvc`编译器在实际项目中可能还是会报错找不到`glu`系列函数，对于`Visual Studio 2022`以及之前的版本可能还需要把源码改为`gb2312`编码。

```json
{
  "version": 2,
  "configurePresets": [
    {
      "name": "vcpkg",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build",
      "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
      }
    }
  ]
}
```

最后创建`CMakeUserPresets.json`​配置本地`vcpkg`路径，该文件仅供本地使用（如果只想通过环境变量配置一个全局的`vcpkg`那么该文件也可以不用创建，后续重启`vscode`后预设使用上述`CMakePresets.json`的`vcpkg`预设即可）

```cmake
{
  "version": 2,
  "configurePresets": [
    {
      "name": "default",
      "inherits": "vcpkg",
      "environment": {
        "VCPKG_ROOT": "<path to vcpkg>"
      }
    }
  ]
}
```

​`VCPKG_ROOT`​对应的值为本地`vcpkg`​的路径，需要修改为自己的`vcpkg`路径

![image](assets/image-20260119175259-w6ldzbj.png)

重启`vscode`​，选择`default`预设

> 若没有弹窗则参考官方文档按下`shift + ctrl + p`​后选择`cmake: build`

![image](assets/image-20260119175147-ae4t2b9.png)

构建完成后会在项目根目录下生成`build`​文件夹以及编译好的`exe`​文件，但是此时直接运行`glutdemo.exe`可能​会报错，通过将`glutdemo/build/vcpkg_installed/x64-windows/debug/bin`​下的`freeglutd.dll`​复制到`build/`下即可

> 如果通过`msvc`编译器编译则不会有这样的问题，它会自动将`dll`复制到`build/`下

![image](assets/image-20260119175856-tsokac9.png)

![image](assets/image-20260119180118-wvijbaa.png)

运行效果如下

![image](assets/image-20260119180156-ptaplm2.png)

## 方案2 使用兼容模式

参考方案1，只需修改`CMakeLists.txt`即可

```cmake
cmake_minimum_required(VERSION 3.10)

project(robert)

set(CMAKE_CXX_STANDARD 17)

find_package(OpenGL REQUIRED)
find_package(GLUT REQUIRED)

add_executable(${PROJECT_NAME} main.cpp)


target_link_libraries(${PROJECT_NAME} PRIVATE
        OpenGL::GL
        GLUT::GLUT

)
```

通过这种方式配置的项目会自动将`freeglut.dll`​复制到`build/`下，所以不会出现前面报错的情况

‍

‍
