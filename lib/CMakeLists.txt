INCLUDE_DIRECTORIES("." "${CMAKE_BINARY_DIR}")
INCLUDE_DIRECTORIES("${EXPAT_INCLUDE_DIRS}")
IF (CUNIT_INCLUDE_DIR)
    INCLUDE_DIRECTORIES("${CUNIT_INCLUDE_DIR}")
ENDIF()

IF(YY_NO_UNISTD_H)
    ADD_DEFINITIONS(-DYY_NO_UNISTD_H)
ENDIF(YY_NO_UNISTD_H)

##
# Check to see if we have flex and bison.
##
FIND_PROGRAM(UD_FLEX NAMES flex)
FIND_PROGRAM(UD_BISON NAMES bison)

# Ensure that the C source-files for the scanner and parser are up-to-date
# on a Unix system.
#
# MODIFICATION: Instead of checking for Unix, see if flex and bison
# are available.
#
# MODIFICATION 2: Moving back for now, but leaving framework.
#  Other complications arose on Windows with MSVC.
#

IF(UNIX)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/scanner.c
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMAND ${UD_FLEX} -d -Put -o scanner.c scanner.l
        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/scanner.l)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/parser.c
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMAND ${UD_BISON} -t -p ut -o parser.c parser.y
        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/parser.y)
    set_source_files_properties(parser.c
        PROPERTIES OBJECT_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/scanner.c)
endif()

SET(libudunits2_src converter.c
		    error.c
		    formatter.c
		    idToUnitMap.c
		    parser.c
		    prefix.c
		    status.c
		    systemMap.c
		    unitAndId.c
		    unitcore.c
		    unitToIdMap.c
		    ut_free_system.c
		    xml.c
		    udunits2.h)

if(MSVC)
	SET(libudunits2_src ${libudunits2_src}
	tsearch.c tsearch.h udunits_snprintf.c)
endif()

add_library(libudunits2-static STATIC ${libudunits2_src})	
target_link_libraries(libudunits2-static PUBLIC ${EXPAT_LIBRARIES} ${MATH_LIBRARY} ${CMAKE_DL_LIBS})
target_compile_options(libudunits2-static PUBLIC -fpic)
set_target_properties(libudunits2-static PROPERTIES
    LIBRARY_OUTPUT_NAME udunits2
    ARCHIVE_OUTPUT_NAME udunits2
    RUNTIME_OUTPUT_NAME udunits2)

if (NOT BUILD_SHARED_LIBS)
    add_library(libudunits2 ALIAS libudunits2-static)
endif()

if (BUILD_SHARED_LIBS)
    add_library(libudunits2-shared SHARED $<TARGET_OBJECTS:libudunits2-static>)	
    target_compile_options(libudunits2-shared PUBLIC -fpic)
    set_target_properties(libudunits2-shared PROPERTIES
        LIBRARY_OUTPUT_NAME udunits2
        ARCHIVE_OUTPUT_NAME udunits2
        RUNTIME_OUTPUT_NAME udunits2)
    target_link_libraries(libudunits2-shared ${EXPAT_LIBRARIES} ${MATH_LIBRARY} ${CMAKE_DL_LIBS})
    add_library(libudunits2 ALIAS libudunits2-shared)

    IF(MSVC)
        SET_TARGET_PROPERTIES(libudunits2-static libudunits2-shared PROPERTIES	LINK_FLAGS_DEBUG " /NODEFAULTLIB:MSVCRT"
    )
    ENDIF()
endif()

if (CUNIT_LIBRARY AND BUILD_SHARED_LIBS)
    add_executable(testUnits testUnits.c)
    target_link_libraries (testUnits libudunits2-shared)
    target_link_libraries (testUnits ${CUNIT_LIBRARY})
    add_test(
        NAME testUnits
        COMMAND testUnits ${CMAKE_CURRENT_SOURCE_DIR}/udunits2.xml)
endif()

# The documentation is in multiple texinfo(5) format files
#texi_doc(udunits2lib.texi ${CMAKE_SOURCE_DIR}/COPYRIGHT)
    
install(TARGETS libudunits2-static EXPORT udunits2-targets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin)

if (BUILD_SHARED_LIBS)
    install(TARGETS libudunits2-shared EXPORT udunits2-targets
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin)
endif()

install(FILES udunits2.h converter.h DESTINATION include)
install(FILES
        udunits2.xml
        udunits2-accepted.xml
        udunits2-base.xml
        udunits2-common.xml
        udunits2-derived.xml
        udunits2-prefixes.xml
    DESTINATION share/udunits)
