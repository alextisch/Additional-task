cmake_minimum_required(VERSION 3.10)
find_package(SFML 2.5 COMPONENTS system window graphics REQUIRED)
project(Xonix)

find_package(SFML REQUIRED network audio graphics window system)

file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/images" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/")

add_executable(xonix main.cpp)

target_link_libraries(xonix PRIVATE sfml-system sfml-window sfml-graphics sfml-network sfml-audio)
