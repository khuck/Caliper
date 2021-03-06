include_directories("../../src/caliper")

# build the C/C++ CI test apps

set(CALIPER_CI_CXX_TEST_APPS
  ci_test_aggregate
  ci_test_basic
  ci_test_binding
  ci_test_esc
  ci_test_loop
  ci_test_macros
  ci_test_nesting
  ci_test_postprocess_snapshot
  ci_test_report
  ci_test_thread)
set(CALIPER_CI_C_TEST_APPS
  ci_dgemm_memtrack
  ci_test_alloc
  ci_test_alloc_hooks
  ci_test_c_ann
  ci_test_c_snapshot)
set(CALIPER_CI_MPI_TEST_APPS
  ci_test_cali_before_mpi
  ci_test_mpi_before_cali)

foreach(app ${CALIPER_CI_CXX_TEST_APPS})
  add_executable(${app} ${app}.cpp)
  target_link_libraries(${app} caliper)
endforeach()

target_link_libraries(ci_test_thread  Threads::Threads)
target_link_libraries(ci_test_nesting Threads::Threads)

foreach(app ${CALIPER_CI_C_TEST_APPS})
  add_executable(${app} ${app}.c)
  set_target_properties(${app} PROPERTIES LINKER_LANGUAGE CXX)
  target_link_libraries(${app} caliper)
endforeach()

# add symlinks to the python test driver scripts
set(PYTHON_SCRIPTS
  test_allocservice.py
  test_aggregate.py
  test_basictrace.py
  test_c_api.py
  test_json.py
  test_log.py
  test_thread.py
  calipertest.py)

if (CALIPER_HAVE_LIBUNWIND)
  list(APPEND PYTHON_SCRIPTS test_callpath.py)
endif()
if (CALIPER_HAVE_GOTCHA)
  list(APPEND PYTHON_SCRIPTS 
    test_alloc_hooks.py
    test_pthread.py)
endif()
if (CALIPER_HAVE_PAPI)
  list(APPEND PYTHON_SCRIPTS test_papi.py)
endif()
if (CALIPER_HAVE_LIBPFM)
  list(APPEND PYTHON_SCRIPTS test_libpfm.py)
endif()
if (CALIPER_HAVE_SAMPLER AND CALIPER_HAVE_DYNINST)
  list(APPEND PYTHON_SCRIPTS test_sampler.py)
endif()

if (CALIPER_HAVE_TAU)
  add_executable(ci_test_tau_service ci_test_tau_service.cpp)
  target_link_libraries(ci_test_tau_service caliper ${TAU_LIBRARIES})
  list(APPEND PYTHON_SCRIPTS test_tau_service.py)
endif()

if (CALIPER_HAVE_MPI)
  include_directories(${MPI_CXX_INCLUDE_PATH})
  
  foreach(app ${CALIPER_CI_MPI_TEST_APPS})
    add_executable(${app} ${app}.cpp)
    target_link_libraries(${app} caliper caliper-mpi ${MPI_CXX_LIBRARIES})
  endforeach()

  if (RUN_MPI_TESTS)
    list(APPEND PYTHON_SCRIPTS test_mpi.py)
  endif()
endif()

foreach(file ${PYTHON_SCRIPTS})
  add_custom_target(${file} ALL
    COMMAND ${CMAKE_COMMAND} -E create_symlink 
      ${CMAKE_CURRENT_SOURCE_DIR}/${file} 
      ${CMAKE_CURRENT_BINARY_DIR}/${file})
endforeach()

add_test(NAME CI_app_tests COMMAND ${PYTHON_EXECUTABLE} -B -m unittest discover -p "test_*.py")
