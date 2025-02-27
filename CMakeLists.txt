project(jmi)
cmake_minimum_required(VERSION 3.0)
option(BUILD_TESTS "build tests" OFF)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17) # TODO: option
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(POLICY CMP0063) # visibility. since 3.3
  cmake_policy(SET CMP0063 NEW)
endif()
set(CMAKE_CXX_VISIBILITY_PRESET hidden) #use with -fdata-sections -ffunction-sections to reduce target size
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

set(JAVA_AWT_LIBRARY NotNeeded)
set(JAVA_JVM_LIBRARY NotNeeded)
#set(JAVA_INCLUDE_PATH2 NotNeeded) # jni_md.h, required by jni.h
set(JAVA_AWT_INCLUDE_PATH NotNeeded)
find_package(Java COMPONENTS Development)
include(UseJava)

message("java=${Java_JAVA_EXECUTABLE}")
message("javac=${Java_JAVAC_EXECUTABLE}")
if(ANDROID)
else()
  find_package(JNI REQUIRED)
  include_directories(${JNI_INCLUDE_DIRS})
  message("JNI_INCLUDE_DIRS: ${JNI_INCLUDE_DIRS}")
  enable_testing()
endif()
add_library(jmi STATIC jmi.cpp)
target_include_directories(jmi PUBLIC ${CMAKE_CURRENT_LIST_DIR})
if(NOT WIN32 AND NOT APPLE AND NOT ANDROID)
  target_link_libraries(jmi PUBLIC pthread) # linux
endif()

if(BUILD_TESTS AND NOT CMAKE_CROSSCOMPILING)
  add_executable(test_signature test/signature.cpp)
  target_link_libraries(test_signature PRIVATE jmi)

  add_library(JMITest SHARED test/JMITest.cpp)
  target_link_libraries(JMITest PRIVATE jmi)
  add_jar(test_jmi test/JMITest.java)
  get_target_property(jar_path test_jmi JAR_FILE)
  get_target_property(class_dir test_jmi CLASSDIR)
  message(STATUS "Jar file: ${jar_path}")
  message(STATUS "Class compiled to: ${class_dir}")
  #add_test(NAME signature_test COMMAND )
  # -Djava.library.path=. required on linux if libJMITest.so can not be found in LD_LIBRARY_PATH
  add_test(NAME jmitest COMMAND ${Java_JAVA_EXECUTABLE} -cp ${jar_path} -Djava.library.path=. JMITest)
  if(ANDROID)
    target_link_libraries(test_signature PRIVATE -landroid -llog)
    target_link_libraries(JMITest PRIVATE -landroid -llog)
  endif()
endif()
