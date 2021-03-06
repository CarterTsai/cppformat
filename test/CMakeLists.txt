set(TEST_MAIN_SRC test-main.cc gtest-extra.cc gtest-extra.h util.cc)
add_library(test-main STATIC ${TEST_MAIN_SRC})
target_link_libraries(test-main format gmock)

# Adds a test.
# Usage: add_fmt_test(name [CUSTOM_LINK] srcs...)
function(add_fmt_test name)
  cmake_parse_arguments(add_fmt_test CUSTOM_LINK "" "" ${ARGN})
  add_executable(${name} ${name}.cc ${add_fmt_test_UNPARSED_ARGUMENTS})
  target_link_libraries(${name} test-main)
  if (NOT add_fmt_test_CUSTOM_LINK)
    target_link_libraries(${name} format)
  endif ()
  add_test(${name} ${name})
endfunction()

add_fmt_test(gtest-extra-test)
add_fmt_test(format-test)
add_fmt_test(format-impl-test CUSTOM_LINK)
add_fmt_test(printf-test)
foreach (target format-test printf-test)
  if (CMAKE_COMPILER_IS_GNUCXX)
    set_target_properties(${target} PROPERTIES COMPILE_FLAGS
      "-Wall -Wextra -pedantic -Wno-long-long -Wno-variadic-macros")
  endif ()
  if (CPP11_FLAG)
    set_target_properties(${target} PROPERTIES COMPILE_FLAGS ${CPP11_FLAG})
  endif ()
endforeach ()
add_fmt_test(util-test mock-allocator.h)
if (CPP11_FLAG)
  set_target_properties(util-test PROPERTIES COMPILE_FLAGS ${CPP11_FLAG})
endif ()

foreach (src ${FMT_SOURCES})
  set(FMT_TEST_SOURCES ${FMT_TEST_SOURCES} ../${src})
endforeach ()

check_cxx_source_compiles("
  #include <type_traits>
  class C { void operator=(const C&); };
  int main() { static_assert(!std::is_copy_assignable<C>::value, \"\"); }"
  HAVE_TYPE_TRAITS)
if (HAVE_TYPE_TRAITS)
  add_definitions(-DFMT_USE_TYPE_TRAITS=1)
endif ()

add_executable(macro-test macro-test.cc ${FMT_TEST_SOURCES} ${TEST_MAIN_SRC})
set_target_properties(macro-test
  PROPERTIES COMPILE_DEFINITIONS "FMT_USE_VARIADIC_TEMPLATES=0")
target_link_libraries(macro-test gmock)

if (HAVE_OPEN)
  add_executable(posix-test posix-test.cc ${FMT_TEST_SOURCES} ${TEST_MAIN_SRC})
  set_target_properties(posix-test
    PROPERTIES COMPILE_DEFINITIONS "FMT_INCLUDE_POSIX_TEST=1")
  target_link_libraries(posix-test gmock)
  add_test(posix-test posix-test)
endif ()

add_executable(header-only-test
  header-only-test.cc header-only-test2.cc test-main.cc)
set_target_properties(header-only-test
  PROPERTIES COMPILE_DEFINITIONS "FMT_HEADER_ONLY=1")
target_link_libraries(header-only-test gmock)

# Test that the library can be compiled with exceptions disabled.
check_cxx_compiler_flag(-fno-exceptions HAVE_FNO_EXCEPTIONS_FLAG)
if (HAVE_FNO_EXCEPTIONS_FLAG)
  add_library(noexception-test STATIC ../format.cc)
  set_target_properties(noexception-test
    PROPERTIES COMPILE_FLAGS -fno-exceptions)
endif ()

add_test(compile-test ${CMAKE_CTEST_COMMAND}
  --build-and-test
  "${CMAKE_CURRENT_SOURCE_DIR}/compile-test"
  "${CMAKE_CURRENT_BINARY_DIR}/compile-test"
  --build-generator ${CMAKE_GENERATOR}
  --build-makeprogram ${CMAKE_MAKE_PROGRAM})
