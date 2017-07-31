---
layout:     post
title:      "C++ 读取文件与字符串类型装换"
subtitle:   " \"C++\""
date:       2016-09-21
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
---

> 简单记录 C++ 读取文件以及字符串转换的一些操作

### 读取文件

#### 文件操作头文件以及类

```
#include <fstream>  
ofstream         //文件写入流，内存写入存储设备   
ifstream         //文件读入流，存储设备读区到内存中  
fstream          //文件读写流，对打开的文件可进行读写操作
```

他们和其他流类之间的派生关系如下：

<img src="http://leiyiming.com/img/in-post/post-c++/filestream/1.png"/>


#### 打开、关闭文件

打开文件可以使用 `open` 函数，也可以直接调用构造函数，两种方法的主要参数相同。

`void open ( const char * filename, ios_base::openmode mode = ios_base::in | ios_base::out );`

* *filename* ：操作文件名。
* *mode* ：打开文件方式，有如下几种，并且能够以“或”运算 `|` 的方式进行组合使用

|        选项        |           意义            |
| :--------------: | :---------------------: |
|   ios_base::in   |       为输入(读)而打开文件       |
|  ios_base::out   |       为输出(写)而打开文件       |
|  ios_base::ate   |        初始位置：文件尾         |
|  ios_base::app   |       所有输出附加在文件末尾       |
| ios_base::trunc  |      如果文件已存在则覆盖写入       |
|  ios::nocreate   | 文件不存在时产生错误，常和in或app联合使用 |
|  ios::noreplace  |   文件存在时产生错误，常和out联合使用   |
| ios_base::binary |    以二进制格式打开（默认为文本格式）    |

检测文件是否正确打开可以使用 `is_open()` 函数：

```
if(file.is_open())
{
	...
}
```

关闭文件可以使用 `close()` 函数，例如： `file.close();` 

#### 文件指针

文件读写过程中，有一个流指针来控制读写的位置，这个指针就是文件指针。

参照位置有以下三种：

|      指针       |               意义               |
| :-----------: | :----------------------------: |
| ios_base::beg |    beginning of the stream     |
| ios_base::cur | current position in the stream |
| ios_base::end |       end of the stream        |

可以使用以下函数来操作文件指针：

```
//输入流（istream）操作
seekg(绝对位置);　　　　　　//绝对移动，　　　　
seekg(相对位置,参照位置);　 //相对移动  
tellg();　　　　　　　　　　//返回当前指针位置  

//输出流（ostream）操作
seekp(绝对位置);　　　　　　//绝对移动　　　　
seekp(相对位置,参照位置);　 //相对移动　　　  
tellp();　　　　　　　　　　//返回当前指针位置  
```

下面是通过移动指针来获得文件长度的一个例子：

```
std::ifstream is ("test.txt", std::ifstream::binary);
is.seekg (0, is.end);     //移动到文件末尾（移动到末尾位置的0偏移处 ）
int length = is.tellg();  //得到文件长度
is.seekg (0, is.beg);     //回到文件开头
```

#### 读写操作

`ifstream` 和 `ofstream` 可以通过 `<<` 或者 `>>` 来进行读写操作，通过这种方式读写的时候都会以空格或者回车为分界读写文件。例如：

```
double d;  
char c;  
char s[20];  
out_file >> i >> d >> c;　
```

`ifstream` 可以通过 `get()` 成员函数来读取一个字符，通过 `get(char *,int n,char delim)` 来读取多个字符，也可以通过 `getline(char*)` 来直接读取一行的内容。例如：

```
char c;
while ((c=fin.get()) != EOF)
	cout << c;

char a[80];
while(fin.get(a, 80, '\0')!=NULL) //以 '\0' 为终止 
	cout << a;
```

`ofstream` 可以通过 `put(char c)` 成员函数来写入一个字符，也可以通过 ` write (const char* s, int n)` 写入多个字符。

#### 一次性将文件内容读入char*中

```C++
    std::ifstream fin(imagePath, std::ios::in | std::ios::binary);
    if (fin.is_open())
    {
        std::istreambuf_iterator<char> beg(fin), end;
        _image = std::string(beg, end);
        fin.close();
    }
```



### 字符串、数字类型转换

#### string --> char *

```C++
string str("OK");
char * p = str.c_str();
```

#### char * -->string

```C++
char *p = "OK";
string str(p);
```

#### string --> int, long, double

```C++
int a = atoi(s.c_str());
long l = atol(s.c_str());
double d = atof(s.c_str());
```

#### int, long, float, double --> string

```C++
string to_string (int val);
string to_string (long val);
string to_string (long long val);
string to_string (unsigned val);
string to_string (unsigned long val);
string to_string (unsigned long long val);
string to_string (float val);
string to_string (double val);
string to_string (long double val);
```

#### 将字符串按不同进制转换为数字

```
long int strtol(const char *nptr, char **endptr, int base)
```

* nptr ：输入字符串

* endptr ：不为 `NULL` 时，会将不符合转换条件的字符串存入其中。

* base : 范围从2至36，或0，代表采用的进制方式，当为 0 时会使用十进制转换，但遇到 "0x" 前置字符时会以十六进制转换。

```
char a[] = "100";
char b[] = "100";
char c[] = "ffff";
printf("a = %d\n", strtol(a, NULL, 10));   //以十进制转换a，结果100
printf("b = %d\n", strtol(b, NULL, 2));    //以二进制转换b，结果4
printf("c = %d\n", strtol(c, NULL, 16));   //以十六进制转换c，结果65535
```