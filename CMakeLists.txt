set(LIBNAME nanotrack)
file(GLOB all_src_files *.cpp)
add_library(${LIBNAME} SHARED ${all_src_files})


option(BUILD_QRB5165 "Build the qrb5165 binary" OFF)

if(BUILD_QRB5165)
    add_definitions(-DBUILD_QRB5165)
endif()

# Enable compile optimizations
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -fsee -fomit-frame-pointer -fno-signed-zeros -fno-math-errno -funroll-loops")

# Enable debug flags (use if you want to debug in gdb)
# set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g3 -Wall -Wuninitialized -Wmaybe-uninitialized -fno-omit-frame-pointer")

# tell the linker not to worry about missing symbols in libraries
set(CMAKE_C_FLAGS   "-Wl,--unresolved-symbols=ignore-in-shared-libs -Wno-pointer-arith ${CMAKE_C_FLAGS}")
set(CMAKE_CXX_FLAGS "-Wl,--unresolved-symbols=ignore-in-shared-libs -Wno-pointer-arith ${CMAKE_CXX_FLAGS}")

# Build from all source files
include_directories(/usr/include/opencv4/ include/ include/ncnn)
file(GLOB OPENCV_LIBS "/usr/lib64/libopencv*.so")

option(HAS_CVP "if hardware supports CVP" false)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^aarch64")
	set(LIB_INSTALL_DIR /usr/lib64)
	find_library(LIBMODAL_PIPE_SO NAMES libmodal_pipe.so HINTS /usr/lib64 )
	find_library(LIBMODAL_JSON_SO NAMES libmodal_json.so HINTS /usr/lib64 )
	find_library(LIBVOXL_COMMON_CONFIG_SO NAMES libvoxl_common_config.so HINTS /usr/lib64 )
	find_library(LIBRC_MATH_SO NAMES librc_math.so HINTS /usr/lib64 )
	find_library(LIBMODAL_JOURNAL_SO NAMES libmodal_journal.so HINTS /usr/lib64 )

elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm")
	message( "configure for ARMv7-A")
	set(LIB_INSTALL_DIR /usr/lib)
	find_library(LIBMODAL_PIPE_SO NAMES libmodal_pipe.so HINTS /usr/lib )
	find_library(LIBMODAL_JSON_SO NAMES libmodal_json.so HINTS /usr/lib )
	find_library(LIBVOXL_COMMON_CONFIG_SO NAMES libvoxl_common_config.so HINTS /usr/lib )
	find_library(LIBRC_MATH_SO NAMES librc_math.so HINTS /usr/lib )
        find_library(LIBMODAL_JOURNAL_SO NAMES libmodal_journal.so HINTS /usr/lib )
	set(CMAKE_CXX_FLAGS "-std=c++11 -pthread -march=armv7-a -mfloat-abi=softfp -mfpu=neon -Wno-unused-function -DENABLE_MAVLINK_SUPPORT")
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O3 -g -pthread -Wall -fPIC")
	set(CMAKE_C_FLAGS "-march=armv7-a -mfloat-abi=softfp -mfpu=neon -pthread -Wno-unused-function")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS} -O3 -g -pthread -Wall -fPIC" )
else()
	set(LIB_INSTALL_DIR /usr/lib)
	find_library(LIBMODAL_PIPE_SO NAMES libmodal_pipe.so HINTS /usr/lib )
	find_library(LIBMODAL_JSON_SO NAMES libmodal_json.so HINTS /usr/lib )
	find_library(LIBVOXL_COMMON_CONFIG_SO NAMES libvoxl_common_config.so HINTS /usr/lib /usr/lib64 )
	find_library(LIBRC_MATH_SO NAMES librc_math.so HINTS /usr/lib )
	find_library(LIBMODAL_JOURNAL_SO NAMES libmodal_journal.so HINTS /usr/lib )
endif()

find_package(OpenCV 4 REQUIRED)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)


add_library(ncnn_static STATIC IMPORTED)
set_target_properties(ncnn_static PROPERTIES
	IMPORTED_LOCATION "${CMAKE_CURRENT_LIST_DIR}/lib/libncnn.a"
	IMPORTED_LINK_INTERFACE_LANGUAGES "C;CXX"
)

target_link_libraries(${LIBNAME} PRIVATE Threads::Threads)

target_link_libraries(${LIBNAME} LINK_PUBLIC 
	m 
	${LIBMODAL_JSON_SO} 
	${LIBMODAL_PIPE_SO} 
	${LIBVOXL_COMMON_CONFIG_SO} 
	${LIBRC_MATH_SO} 
	${OpenCV_LIBRARIES} 
	pthread 
	ncnn_static 
	gomp
	${LIBMODAL_JOURNAL_SO} )

target_include_directories(${LIBNAME} PUBLIC include )

# make sure everything is installed where we want
# LIB_INSTALL_DIR comes from the parent cmake file
install(
	TARGETS			${LIBNAME}
	LIBRARY			DESTINATION ${LIB_INSTALL_DIR}
	RUNTIME			DESTINATION /usr/bin
	PUBLIC_HEADER	DESTINATION /usr/include
)

