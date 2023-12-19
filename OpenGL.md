## 配置OpenGL和GLEW
- 下载指定库：GLFW和GLEW
  - GLFW：老版openGL，需要使用该库创建openGL上下文
  - GLEW：新版openGL，在上述上下文中才能够正常进行操作
- 将库复制到指定的solution文件夹(含有.sln的那个文件夹)中，并放入新建的include文件夹中便于后续管理
  - VS中选择项目名称并打开属性，在c++通用以及链接器通用中加入需要使用的头文件所在路径
  - ![img](./OpenGL_img/属性.png)
```
#include<GL/glew.h> //由于内部设置，必须将该头文件置于顶部
#include <GLFW/glfw3.h>

#include<iostream>


int main(void)
{
    GLFWwindow* window;

    /* Initialize the library */
    if (!glfwInit())
        return -1;
    
    /* Create a windowed mode window and its OpenGL context */
    window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
    if (!window)
    {
        glfwTerminate();
        return -1;
    }

    /* Make the window's context current */
    glfwMakeContextCurrent(window);

    //glew中的函数必须要处在已经建立完成的openGL的上下文中才能够正常运行
    if (glewInit() != GLEW_OK)
        std::cout << "Error" << std::endl;

    // 打印glew版本及所用驱动
    std::cout << glGetString(GL_VERSION) << std::endl;

    /* Loop until the user closes the window */
    while (!glfwWindowShouldClose(window))
    {
        /* Render here */
        glClear(GL_COLOR_BUFFER_BIT);

        //标准语法，画一个三角形并指定其三个顶点
        glBegin(GL_TRIANGLES);
        glVertex2f(-0.5f, -0.5f);
        glVertex2f(1.0f, 0.5f);
        glVertex2f(-0.5f, 1.0f);
        glEnd();


        /* Swap front and back buffers */
        glfwSwapBuffers(window);

        /* Poll for and process events */
        glfwPollEvents();
    }

    glfwTerminate();
    return 0;
}
```

##使用GLEW绘制三角形
- 需要使用**顶点缓冲区**(一系列字节存放在VRAM中)以及**着色器**(一套在GPU上执行的程序)
- 顶点缓冲区
```
    //定义顶点缓冲区 静态 由于glGenBuffers是返回void的，所以函数中还需要一个指针来指向
    //创建的缓冲区，在后续函数运行中将该指针传入，相当于该缓冲区的id
    float position[6] = {
        -0.5f, -0.5f,
         0.0f,  0.5f,
         0.5f, -0.5f
    };
    unsigned int buffer = 1;
    glGenBuffers(1, &buffer);
    glBindBuffer(GL_ARRAY_BUFFER, buffer); //声明只是一个数组
    glBufferData(GL_ARRAY_BUFFER, sizeof(position), position, GL_STATIC_DRAW); //指定缓冲区大小
    //启动顶点
    glEnableVertexAttribArray(0);
                                                                              
    //指定顶点的属性，假设顶点有位置和纹理两个属性，每个属性占8个字节
    //0：开始的位置 index， 2：位置占两个float(与后面指定类型有关)，GL_FLOAT：指定类型
    //GL_FALSE：是否标准化(映射到0-1), sizeof(float)*2：指明从当前节点到下一个节点需要经过的字节数
    //0：需要指定的属性的起点；如果指定属性是纹理，则需要改为8
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, sizeof(float) * 2, 0);
```
- 着色器
  - 顶点着色器：每一个顶点调用一次，决定顶点在屏幕上的位置
  - 片段(像素)着色器:每一个像素调用一次(光栅化)
```
static unsigned int CompileShader(unsigned int type, const std::string& source)
{
    unsigned int id = glCreateShader(type);
    const char* src = source.c_str();
    glShaderSource(id, 1, &src, nullptr);
    glCompileShader(id);

    //错误处理
    int result;
    glGetShaderiv(id, GL_COMPILE_STATUS, &result);
    if (result == GL_FALSE)
    {
        int length;
        glGetShaderiv(id, GL_INFO_LOG_LENGTH, &length);
        char* message = (char*)alloca(length * sizeof(char));
        glGetShaderInfoLog(id, length, &length, message);
        std::cout << "Failed to complie" << (type == GL_VERTEX_SHADER ? "vertex" : "fragment") << "shader" << std::endl;
        std::cout << message << std::endl;
        glDeleteShader(id);
        return 0;
    }

    return id;
}

static unsigned int CreateShader(const std::string& vertexshader, const std::string& fragmentshader)
{
    unsigned int program = glCreateProgram();
    unsigned int vs = CompileShader(GL_VERTEX_SHADER, vertexshader);
    unsigned int fs = CompileShader(GL_FRAGMENT_SHADER, fragmentshader);

    //附加与链接
    glAttachShader(program, vs);
    glAttachShader(program, fs);
    glLinkProgram(program);
    glValidateProgram(program);

    //删除着色器
    glDeleteShader(vs);
    glDeleteShader(fs);

    return program;
}

...在openGL的while循环前编写简单着色器，指定位置以及颜色
//location=0与前面glVertexAttribPointer的第一个0相对应
//哪怕提取出来的是二维的 应该写成vec2但是由于下面gl_Position只接受vec4，所以不如提前提取成vec4的形式
std::string vertexShader =
        "#version 330 core\n"
        "\n"
        "layout(location = 0) in vec4 position;\n"
        "\n"
        "void main()\n"
        "{\n"
        "   gl_Position = position;\n"
        "}\n";

    std::string fragmentShader =
        "#version 330 core\n"
        "\n"
        "layout(location = 0) out vec4 color;\n"
        "\n"
        "void main()\n"
        "{\n"
        "   color = vec4(1.0, 0.0, 0.0, 1.0);\n"
        "}\n";

    unsigned int shader = CreateShader(vertexShader, fragmentShader);
    glUseProgram(shader);
```