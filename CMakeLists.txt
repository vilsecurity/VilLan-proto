cmake_minimum_required(VERSION 3.10)

project(villan_proto)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

find_package(Protobuf REQUIRED)
find_package(Grpc REQUIRED)

file(GLOB source ${CMAKE_CURRENT_SOURCE_DIR}/*.proto)

protobuf_generate_cpp(proto_gen_source proto_gen_headers ${source})
grpc_generate_cpp(rpc_gen_source rpc_gen_headers ${source})

add_library(villan_proto ${proto_gen_source} ${rpc_gen_source})

target_include_directories(villan_proto
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)

target_link_libraries(villan_proto
  PUBLIC
  gRPC::grpc++_reflection
  gRPC::grpc++
  gRPC::gpr
)

set_target_properties(villan_proto PROPERTIES EXPORT_NAME VilSecurity::VilLan::Proto)

add_subdirectory(internal)
