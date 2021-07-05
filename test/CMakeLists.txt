cmake_minimum_required(VERSION 3.20)

enable_language(CXX)

option(BUILD_CUDA_TESTS "Include tests for use of the printf library with CUDA host-side and device-side code"  OFF)

set(test_targets autotest)
if (NOT ALIAS_STANDARD_FUNCTION_NAMES)
	list(APPEND test_targets test_suite)
	set(targets_needing_config_h test_suite)
endif()


option(TEST_WITH_NON_STANDARD_FORMAT_STRINGS "Include tests using non-standard-compliant format strings?" ON)
# ... don't worry, we'll suppress the compiler warnings for those.

if(BUILD_CUDA_TESTS)
	enable_language(CUDA)
	list(APPEND CMAKE_CUDA_FLAGS "--extended-lambda --expt-relaxed-constexpr -Xcudafe --display_error_number")
	if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.18)
	    cmake_policy(SET CMP0104 OLD)
	endif()
	include(FindCUDA/select_compute_arch)
	list(APPEND cuda_test_targets cuda_test_suite_host cuda_test_suite_device)
	if((NOT DEFINED CUDA_ARCH_FLAGS) OR ("${CUDA_ARCH_FLAGS}" STREQUAL ""))
		cuda_select_nvcc_arch_flags(CUDA_ARCH_FLAGS_1 Auto)
		set(CUDA_ARCH_FLAGS ${CUDA_ARCH_FLAGS} CACHE STRING "CUDA -gencode parameters")
		string(REPLACE ";" " " CUDA_ARCH_FLAGS_STR "${CUDA_ARCH_FLAGS}")
	else()
		set(CUDA_ARCH_FLAGS_STR "${CUDA_ARCH_FLAGS}")
	endif()
	set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} ${CUDA_ARCH_FLAGS_STR}")

	foreach(tgt ${cuda_test_targets})
		string(REPLACE "cuda_" "cuda/" source_file_prefix ${tgt})
		add_executable(${tgt} "${source_file_prefix}.cu")
		target_include_directories(${tgt} PRIVATE "$<BUILD_INTERFACE:${GENERATED_INCLUDE_DIR}>")
		set_target_properties(
			${tgt}
			PROPERTIES
			CUDA_STANDARD 11
			CUDA_STANDARD_REQUIRED YES
			CUDA_EXTENSIONS NO
		)
		if (TEST_WITH_NON_STANDARD_FORMAT_STRINGS)
			target_compile_definitions(${tgt} PRIVATE TEST_WITH_NON_STANDARD_FORMAT_STRINGS)
		endif()
	endforeach()
endif()

if(BUILD_CUDA_TESTS)
	if (NOT ALIAS_STANDARD_FUNCTION_NAMES)
		target_link_libraries(cuda_test_suite_device printf_cuda)
		set_target_properties(cuda_test_suite_device PROPERTIES CUDA_SEPARABLE_COMPILATION YES)
		list(APPEND targets_needing_config_h cuda_test_suite_host)
#        target_compile_options(cuda_test_suite_device PRIVATE -G -g)
	endif()
endif()

foreach(tgt ${test_targets})
	add_executable(${tgt} "${tgt}.cpp")
	set_target_properties(
		${tgt}
		PROPERTIES
		CXX_STANDARD 11
		CXX_STANDARD_REQUIRED YES
		CXX_EXTENSIONS NO
	)

	if (TEST_WITH_NON_STANDARD_FORMAT_STRINGS)
		target_compile_definitions(${tgt} PRIVATE TEST_WITH_NON_STANDARD_FORMAT_STRINGS)
	endif()
endforeach()

add_executable(aliasing "aliasing.c")
set_target_properties(
	${tgt}
	PROPERTIES
	C_STANDARD 11
	C_STANDARD_REQUIRED YES
	C_EXTENSIONS NO
)
list(APPEND test_targets aliasing)

foreach(tgt ${test_targets})
	get_target_property(tgt_lang ${tgt} LANGUAGE)
	if (tgt_lang EQUAL "CXX")
		set(tgt_compiler_id ${CMAKE_CXX_COMPILER_ID})
	else() # it's C
		set(tgt_compiler_id ${CMAKE_C_COMPILER_ID})
	endif()

	if (tgt_compiler_id STREQUAL "MSVC")
		target_compile_options(${tgt} PRIVATE /W4)
	elseif (tgt_compiler_id STREQUAL "GNU" OR
			tgt_compiler_id STREQUAL "Clang")
		# lots of warnings and all warnings as errors
		target_compile_options(${tgt} PRIVATE
			-g
			-Wall
			-Wextra
			-pedantic
			-Wundef
			-Wsign-conversion
			-Wuninitialized
			-Wshadow
			-Wunreachable-code
			-Wswitch-default
			-Wswitch
			-Wcast-align
			-Wmissing-include-dirs
			-Winit-self
			-Wdouble-promotion
			-gdwarf-2
			-fno-exceptions
			-ffunction-sections
			-fdata-sections
			-fverbose-asm
			-Wunused-parameter
		)
		if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
			target_compile_options(${tgt} PRIVATE -ffat-lto-objects)
		endif()
	endif()
endforeach()

if (NOT ALIAS_STANDARD_FUNCTION_NAMES)
	# These following is necessary, since thee targets applying printf_config.h
	# do not actually use the compiled library - they includes the library's source .c file;
	# so we need to make sure it's accessible to them
	foreach(tgt ${targets_needing_config_h})
		target_compile_definitions(test_suite PRIVATE PRINTF_INCLUDE_CONFIG_H)
		target_include_directories(
			${tgt}
			PRIVATE
			"${GENERATED_INCLUDE_DIR}"
			"$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/../src/>"
		)
	endforeach()
	add_test(
		NAME "${PROJECT_NAME}.test_suite"
		COMMAND "test_suite" # ${TEST_RUNNER_PARAMS}
	)
endif()

target_link_libraries(autotest PRIVATE printf)
target_include_directories(autotest PRIVATE "$<BUILD_INTERFACE:${GENERATED_INCLUDE_DIR}>")

add_test(
	NAME "${PROJECT_NAME}.aliasing"
	COMMAND "aliasing"
)
target_link_libraries(aliasing PRIVATE printf)
target_include_directories(aliasing PRIVATE "$<BUILD_INTERFACE:${GENERATED_INCLUDE_DIR}>")

# Not running autotest by default - it's randomized after all.

