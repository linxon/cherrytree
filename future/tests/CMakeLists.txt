
include_directories(
    ${CPPUTEST_INCLUDE_DIRS}
    ${CMAKE_SOURCE_DIR}/src/ct
)

set(CT_TEST_FILES
    tests_misc_utils.cpp
    tests_tmp_n_p7zip.cpp
    tests_types.cpp
)

add_executable(run_tests ${CT_TEST_FILES})
target_link_libraries(run_tests 
    ${CPPUTEST_LIBRARIES}
    cherrytree_shared
)

add_test(run_tests run_tests)
