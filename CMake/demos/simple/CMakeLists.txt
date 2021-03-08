cmake_minimum_required(VERSION 3.10)
#set the project name 
project(hello)


set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

add_executable(hello hello.c)
add_library(dynamic_hello SHARED hello.c)
add_library(static_hello STATIC hello.c)

link_directories(lib)