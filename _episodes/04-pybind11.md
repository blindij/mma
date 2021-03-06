---
layout: episode
title: Hands-on example using PyBind11
teaching: 40
exercises: 0
questions:
  - Write me.
objectives:
  - Unit testing and prototyping of C++ code.
keypoints:
  - Write me.
---

## Section

Assume we are starting a project on  Phonetic Algorithms. We have received
source code for the Soundex algorithm which is one Phonetic Algorithm
(The code used here is from  [Jeff Langr's](http://langrsoft.com/about/) book
[Modern C++ Programming with Test-Driven Development](http://pragprog.com/book/lotdd/modern-c-programming-with-test-driven-development)

Our ambition is to implement a Phonetic Algorithm class where Soundex is one
several encoding algorithm

Other Phonetic Algorithms are [Daitch-Mokotoff Soundex](https://en.wikipedia.org/wiki/Daitch–Mokotoff_Soundex),
[Cologne phonetics](https://en.wikipedia.org/wiki/Cologne_phonetics) or [NYIIS](https://en.wikipedia.org/wiki/New_York_State_Identification_and_Intelligence_System)

## The Soundex Source code

 Here is source code for a Phonetic Algorithm  [Soundex](https://en.wikipedia.org/wiki/Soundex) . We want to make use of Soundex in Python with help of Pybind11


```cpp
#ifndef SOUNDEX_H
#define SOUNDEX_H
#include <string>
#include <unordered_map>
class Soundex {
    static const size_t MaxCodeLength{4};
public:
    std::string encode(const std::string& word) const {
        return zeroPad(upperFront(head(word))+tail(encodedDigits(word)));
    }

    std::string encodedDigit(char letter) const {
        const std::unordered_map<char,std::string> encodings{
                {'b',"1"},{'f',"1"},{'p',"1"},{'v',"1"},
                {'c',"2"},{'g',"2"},{'j',"2"},{'k',"2"},{'q',"2"},{'s',"2"},{'x',"2"},{'z',"2"},
                {'d',"3"},{'t',"3"},
                {'l',"4"},
                {'m',"5"},{'n',"5"},
                {'r',"6"}

        };
        auto it = encodings.find(lower(letter));
        return it == encodings.end() ? NotADigit: it->second;
    }

private:
    const std::string NotADigit{"*"};

    char lower(char c) const {
        return std::tolower(static_cast<unsigned char>(c));
    }

    std::string head(const std::string& word) const {
        return word.substr(0,1);
    }

    std::string tail(const std::string& word) const {
        return word.substr(1);
    }

    std::string encodedDigits(const std::string& word) const{
        std::string encoding;
        encodeHead(encoding, word);
        encodeTail(encoding,word);
        return encoding;
    }

    void encodeHead(std::string& encoding, const std::string& word) const {
        encoding += encodedDigit(word.front());
    }

    void encodeTail(std::string& encoding, const std::string& word) const{
        for (auto i=1u; i < word.length();i++)  {
            if (!isComplete(encoding))
                encodeLetter(encoding,word[i],word[i-1]);
        }
    }

    void encodeLetter(std::string& encoding, char letter, char lastLetter) const {
        auto digit = encodedDigit(letter);
        if ( digit != NotADigit && ( digit != lastDigit(encoding) || isVowel(lastLetter)))
            encoding += digit;
    }

    bool isComplete(const std::string& encoding) const{
        return encoding.length() == MaxCodeLength;
    }

    std::string zeroPad(const std::string& word) const {
        auto zerosNeeded = MaxCodeLength - word.length();
        return word + std::string(zerosNeeded,'0');
    }

    std::string lastDigit(const std::string& encoding) const {
        if (encoding.empty()) return NotADigit;
        return std::string(1,encoding.back());

    }

    std::string upperFront(const std::string& string) const {
        return std::string(1, std::toupper(static_cast<unsigned char>(string.front())));
    }

    bool isVowel(char letter) const {
        return std::string("aeiouy").find(lower(letter)) != std::string::npos;
    }
};

#endif //SOUNDEX_H
```
The source code implements the [Soundex algorithm](https://en.wikipedia.org/wiki/Soundex) which according to Wikipedia maps a name or word to its' first letter
followed by three numerical digits. Outlined the algorithm goes like:
 1. Retain the first letter of the name and drop all other occurrences of a,e,i,o,u,y,h,w
 2. Replace consonants with digits as follows (after the first letter)
    * b,f,p,v -> 1
    * c,g,j,k,q,s,x,z, -> 2
    * d,t -> 3
    * m,n -> 5
    * r -> 6

 3. If two or more letters with the same number are adjacent in the original name
 (before step 1), only retain the first letter; also two letters with the same
 number separated by 'h' or 'w' are coded as a single number, whereas such letters
 separated by a vowel are coded twice. This rule also applies to the first letter.
 4. If you have too few letters in your word that you can't assign three numbers,
 append with zeros until there are three numbers. If you have more than 3 letters,
 just retain the first 3 numbers.


### Building the library using CMake
We will make a subdirectory pybind11 with a *src* subdirectory where put the source code
In addition there will be a CMakeLists.txt file under pybind11 and a CMakeLists.txt
under src:

```shell
pybind11 | -  CMakeLists.txt
         |
	 |
	 | -  src | -  CMakeLists.txt 
	          |
	          | -  Soundex.h
	          |
	          | -  Py11Soundex.cpp

```

Let us create the subdirectory setup and the files:

```shell
(pybind11-example) [lynx@lille-login2~]$ mkdir -p pybind11/src
(pybind11-example) [lynx@lille-login2~]$ cd pybind11/src
(pybind11-example) [lynx@lille-login2~]$ cat > Soundex.h
Create the Soundex.h file by pasting in the source code either by using cat
or your favorite editor.
```
The contents of Py11Soundex.cpp:

```cpp
// File: Py11Soundex.cpp
#include <pybind11/pybind11.h>
#include "Soundex.h"

namespace py = pybind11;

PYBIND11_PLUGIN(soundex) {
  py::module m("soundex", "pybind11 soundex plugin");

  py::class_<Soundex>(m,"soundex")
    .def(py::init<>())
    .def("encode", &Soundex::encode);
}

``` 
Create the Py11Soundex.cpp file in the *src* subdirectory:

```shell
(pybind11-example) [lynx@lille-login2src]$ cat > Py11Soundex.cpp
Cat the contents of Py11Soundex.cpp to Py11Soundex.cpp or use
an editor

```
Create also the CMakeLists.txt file in the *src* subdirectory.
Here is the CMakeLists.txt file, just one line:

```cmake
pybind11_add_module(soundex Py11Soundex.cpp)
```
Change to the directory above after you have made the file:

```shell
(pybind11-example) [lynx@lille-login2src]$ cat > CMakeLists.txt
pybind11_add_module(soundex Py11soundex.cpp)
<Ctrl>-d
(pybind11-example) [lynx@lille-login2src]$ cd ..
(pybind11-example) [lynx@lille-login2py11bind]$ 

```

Create the top level CMakeLists.txt in this directory. Here is the contents:

```cmake
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)
project(soundex)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
  ${CMAKE_BINARY_DIR}/lib
  )

find_package(pybind11 REQUIRED )

add_subdirectory(src)
```
After you have made the CMakeLists.txt, make a build subdirectory, *cd* into it
generate the build files by running cmake, and execute make

```shell
(pybind11-example) [lynx@lille-login2src]$ cat > CMakeLists.txt
Paste the contents of the CMakeLists.txt and press <Ctrl>-d
(pybind11-example) [lynx@lille-login2src]$ mkdir build
(pybind11-example) [lynx@lille-login2src]$ cd build
(pybind11-example) [lynx@lille-login2build]$  cmake ..
(pybind11-example) [lynx@lille-login2build]$  make

```
After the make, we cd into the lib subdirectory and load the soundex library:

```shell
(pybind11-example) [lynx@lille-login2 build]$ cmake ..
-- The C compiler identification is GNU 6.3.0
-- The CXX compiler identification is GNU 6.3.0
-- Check for working C compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/gcc
-- Check for working C compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/gcc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working CXX compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/g++
-- Check for working CXX compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/g++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Found PythonInterp: /home/lynx/anaconda2/envs/pybind11-example/bin/python (found version "2.7.13") 
-- Found PythonLibs: /home/lynx/anaconda2/envs/pybind11-example/lib/libpython2.7.so
-- Performing Test HAS_CPP14_FLAG
-- Performing Test HAS_CPP14_FLAG - Success
-- Performing Test HAS_CPP11_FLAG
-- Performing Test HAS_CPP11_FLAG - Success
-- Performing Test HAS_FLTO
-- Performing Test HAS_FLTO - Success
-- LTO enabled
-- Configuring done
-- Generating done
-- Build files have been written to: /home/lynx/src/c++/encodings/soundex/pybind11/build
(pybind11-example) [lynx@lille-login2 build]$ make
Scanning dependencies of target soundex
[100%] Building CXX object src/CMakeFiles/soundex.dir/Py11Soundex.cpp.o
Linking CXX shared module ../lib/soundex.so
[100%] Built target soundex
(pybind11-example) [lynx@lille-login2 build]$ cd lib
(pybind11-example) [lynxy@lille-login2 lib]$ python
Python 2.7.13 |Continuum Analytics, Inc.| (default, Dec 20 2016, 23:09:15) 
[GCC 4.4.7 20120313 (Red Hat 4.4.7-1)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
Anaconda is brought to you by Continuum Analytics.
Please check out: http://continuum.io/thanks and https://anaconda.org
>>> import soundex
>>> p = soundex.soundex()
>>> p.encode('Allison')
u'A425'
>>> 
```

### Inheritance
Let us take introduce inheritance by creating a parent class PhoneticAlgorithms.
The Soundex class will inherit this class. The Pybind11 directory will contain
the following files (before we again run CMake and Make):

```
├── CMakeLists.txt
└── src
    ├── CMakeLists.txt
    ├── PhoneticAlgorithm.h
    ├── Py11PhoneAlg.cpp
    └── Soundex.h
```

The contents of the top level CMakeLists.txt. It is only the project name
which has changed:

```cmake
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)
project(phonetic)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
  ${CMAKE_BINARY_DIR}/lib
  )

find_package(pybind11 REQUIRED )

add_subdirectory(src)

```

The contents of the CMakeLists.txt in the *src*-directory:

```cmake
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)
project(phonetic)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
  ${CMAKE_BINARY_DIR}/lib
  )

find_package(pybind11 REQUIRED )

add_subdirectory(src)

```

The contents of the CMakeLists.txt in the *src*-directory. Here the library
is given a new name and it is a new source file:

```cmake
pybind11_add_module(phonetic Py11PhoneAlg.cpp)
```

The contents of file PhoneticAlgorithms.h:

```c++
// file: PhoneticAlgorithms.h
#ifndef PHONETICALGORITHMS_H
#define PHONETICALGORITHMS_H
#include <string>
class PhoneticAlgorithm{
 public:
  std::string encode(const std::string& word) {
    return word + " do not encode\n";
  };

  
};

#endif //PHONETICALGORITHMS_H
```

Contents of Soundex.h. It includes PhoneticAlgorithms.h and states that
Soundex inherits class PhoneticAlgorithm:

```cpp
// file: Soundex.h
#ifndef SOUNDEX_SOUNDEX_H
#define SOUNDEX_SOUNDEX_H
#include <string>
#include <unordered_map>

#include "PhoneticAlgorithm.h"

// Soundex inherit PhoneticAlgorithm

class Soundex : public PhoneticAlgorithm { 
    static const size_t MaxCodeLength{4};
public:
    std::string encode(const std::string& word) const {
        return zeroPad(upperFront(head(word))+tail(encodedDigits(word)));
    }

    std::string encodedDigit(char letter) const {
        const std::unordered_map<char,std::string> encodings{
                {'b',"1"},{'f',"1"},{'p',"1"},{'v',"1"},
                {'c',"2"},{'g',"2"},{'j',"2"},{'k',"2"},{'q',"2"},{'s',"2"},{'x',"2"},{'z',"2"},
                {'d',"3"},{'t',"3"},
                {'l',"4"},
                {'m',"5"},{'n',"5"},
                {'r',"6"}

        };
        auto it = encodings.find(lower(letter));
        return it == encodings.end() ? NotADigit: it->second;
    }

private:
    const std::string NotADigit{"*"};

    char lower(char c) const {
        return std::tolower(static_cast<unsigned char>(c));
    }

    std::string head(const std::string& word) const {
        return word.substr(0,1);
    }

    std::string tail(const std::string& word) const {
        return word.substr(1);
    }

    std::string encodedDigits(const std::string& word) const{
        std::string encoding;
        encodeHead(encoding, word);
        encodeTail(encoding,word);
        return encoding;
    }

    void encodeHead(std::string& encoding, const std::string& word) const {
        encoding += encodedDigit(word.front());
    }

    void encodeTail(std::string& encoding, const std::string& word) const{
        for (auto i=1u; i < word.length();i++)  {
            if (!isComplete(encoding))
                encodeLetter(encoding,word[i],word[i-1]);
        }
    }

    void encodeLetter(std::string& encoding, char letter, char lastLetter) const {
        auto digit = encodedDigit(letter);
        if ( digit != NotADigit && ( digit != lastDigit(encoding) || isVowel(lastLetter)))
            encoding += digit;
    }

    bool isComplete(const std::string& encoding) const{
        return encoding.length() == MaxCodeLength;
    }

    std::string zeroPad(const std::string& word) const {
        auto zerosNeeded = MaxCodeLength - word.length();
        return word + std::string(zerosNeeded,'0');
    }

    std::string lastDigit(const std::string& encoding) const {
        if (encoding.empty()) return NotADigit;
        return std::string(1,encoding.back());

    }

    std::string upperFront(const std::string& string) const {
        return std::string(1, std::toupper(static_cast<unsigned char>(string.front())));
    }

    bool isVowel(char letter) const {
        return std::string("aeiouy").find(lower(letter)) != std::string::npos;
    }
};

#endif //SOUNDEX_SOUNDEX_H
```

The Pybind11-cpp file Py11PhoneAlg.cpp is a new:

```cpp
#include <pybind11/pybind11.h>
#include "PhoneticAlgorithm.h"
#include "Soundex.h"

namespace py = pybind11;

  
PYBIND11_PLUGIN(phonetic) {
  py::module m("phonetic", "pybind11 phonetic plugin");

  py::class_<PhoneticAlgorithm>(m,"PhoneticAlgorithm")
    .def(py::init<>())
    .def("encode", &PhoneticAlgorithm::encode);

  py::class_<Soundex,PhoneticAlgorithm /* specify C++ parent */>(m,"soundex")
    .def(py::init<>())
    .def("encode", &Soundex::encode);

  return m.ptr();

}
```

We generate the builds files in a subdirectory. Run make and load the library
into our python interpreter:

```shell
(pybind11-example) [bjornlin@lille-login2 build]$ cmake ..
-- The C compiler identification is GNU 6.3.0
-- The CXX compiler identification is GNU 6.3.0
-- Check for working C compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/gcc
-- Check for working C compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/gcc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working CXX compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/g++
-- Check for working CXX compiler: /share/apps/modulessoftware/Core/gcc/6.3.0/bin/g++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Found PythonInterp: /home/bjornlin/anaconda2/envs/pybind11-example/bin/python (found version "2.7.13") 
-- Found PythonLibs: /home/bjornlin/anaconda2/envs/pybind11-example/lib/libpython2.7.so
-- Performing Test HAS_CPP14_FLAG
-- Performing Test HAS_CPP14_FLAG - Success
-- Performing Test HAS_CPP11_FLAG
-- Performing Test HAS_CPP11_FLAG - Success
-- Performing Test HAS_FLTO
-- Performing Test HAS_FLTO - Success
-- LTO enabled
-- Configuring done
-- Generating done
-- Build files have been written to: /home/bjornlin/src/c++/encodings/phoneticA/build
(pybind11-example) [bjornlin@lille-login2 build]$ make
Scanning dependencies of target phonetic
[100%] Building CXX object src/CMakeFiles/phonetic.dir/Py11PhoneAlg.cpp.o
Linking CXX shared module ../lib/phonetic.so
[100%] Built target phonetic
(pybind11-example) [bjornlin@lille-login2 build]$ cd lib
(pybind11-example) [bjornlin@lille-login2 lib]$ python
Python 2.7.13 |Continuum Analytics, Inc.| (default, Dec 20 2016, 23:09:15) 
[GCC 4.4.7 20120313 (Red Hat 4.4.7-1)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
Anaconda is brought to you by Continuum Analytics.
Please check out: http://continuum.io/thanks and https://anaconda.org
```
In python we import the library and try it out:

```python
>>> import phonetic
>>> help(phonetic)

>>> s = phonetic.PhoneticAlgorithm()
>>> s.encode('Allison')
u'Allison do not encode\n'
>>> t = phonetic.soundex()
>>> t.encode('Allison')
u'A425'
>>> 
```

### Extending the class by prototyping

In Python you can add new attributes to a class dynamically. As the Phonetic/Algorithm
and Soundex class are now, they can not get new attributes dynamically.
This feature can be added to the parent (PhoneticAlgorithm), at the same time
we add a new class we want to implement, the ReverseSoundex.

We add the file ReverseSoundex.h with following contens to the *src*-subdirectory:

```cpp
#ifndef REVERSESOUNDEX_H
#define REVERSESOUNDEX_H
#include  "PhoneticAlgorithm.h"

class ReverseSoundex : public PhoneticAlgorithm {

};
#endif  // REVERSESOUNDEX
```

The Py11PhoneAlg.cpp is changed to:

```cpp
#include <pybind11/pybind11.h>
#include "PhoneticAlgorithm.h"
#include "Soundex.h"
#include "ReverseSoundex.h"

namespace py = pybind11;

  
PYBIND11_PLUGIN(phonetic) {
  py::module m("phonetic", "pybind11 phonetic plugin");

  py::class_<PhoneticAlgorithm>(m,"PhoneticAlgorithm",py::dynamic_attr())
    .def(py::init<>())
    .def("encode", &PhoneticAlgorithm::encode);

  py::class_<Soundex,PhoneticAlgorithm /* specify C++ parent */>(m,"soundex")
    .def(py::init<>())
    .def("encode", &Soundex::encode);

  py::class_<ReverseSoundex,PhoneticAlgorithm /* specify C++ parent */>(m,"reversesoundex")
    .def(py::init<>());

  return m.ptr();

}
```

We are now starting to implement a new class, but we have not implemented any
methods or attributes. When have we done the cmake/make build, we can start
 the python interpreter and import the library:
 
```python
>>> import phonetic
>>> t = phonetic.PhoneticAlgorithm()
>>> s = phonetic.soundex()
>>> r = phonetic.reversesoundex()
>>> t.__dict__
{}
>>> s.__dict__
{}
>>> r.__dict__
{}
```

The subclass has inherited the dynamic feature from the parent class Phonetic Algorithm.
Let us prototype the encode method for ReverseSoundex classL

``` python
>>> def rencode(word):
...   x = s.encode(word)
...   return x[1:]+word[:1]
... 
>>> r.encode=rencode
>>> r.__dict__
{'encode': <function rencode at 0x7f25e7e3ca28>}
>>> r.encode('Allison')
u'425A'
>>> 
```

LocalWords:  Soundex Pybind11
