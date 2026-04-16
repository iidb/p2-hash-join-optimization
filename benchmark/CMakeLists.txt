add_executable(p2-leaderboard run_job.cpp)
target_include_directories(
        p2-leaderboard PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../src/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
target_link_libraries(p2-leaderboard babydb)