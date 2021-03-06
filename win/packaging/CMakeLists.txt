SET(CLIENT_LIB_DIR ${CMAKE_BINARY_DIR}/libmariadb/${CMAKE_BUILD_TYPE})
SET(CLIENT_DBG_DIR ${CMAKE_BINARY_DIR}/libmariadb/Debug)
SET(CLIENT_INC_DIR ${CMAKE_SOURCE_DIR}/include)

SET(PRODUCT_NAME "MariaDB Connector C")
SET(PRODUCT_INSTALL_DIR "MariaDB")
SET(PRODUCT_MANUFACTURER "MariaDB Corporation")
SET(PRODUCT_VERSION "${CPACK_PACKAGE_VERSION}")

MESSAGE(STATUS "sizeof ${CMAKE_SIZEOF_VOID_P}")

IF (${CMAKE_SIZEOF_VOID_P} EQUAL 8)
  SET(PRODUCT_NAME "${PRODUCT_NAME} 64-bit")
  SET(PLATFORM "win64")
  SET(IS_WIN64 "yes")
  SET(WIXPLATFORM "x64")
  SET(PRODUCT_UPGRADE_CODE "4E630B8C-4645-416D-A561-45D88E7BDCF1")
  SET(FOLDER "ProgramFiles64Folder")
ELSE()
  SET(PLATFORM "win32")
  SET(IS_WIN64 "no")
  SET(WIXPLATFORM "x86")
  SET(PRODUCT_UPGRADE_CODE "FCBC4419-07C8-4595-9803-DFD602A84F29")
  SET(FOLDER "ProgramFilesFolder")
ENDIF()

IF (WITH_SIGNCODE)
  SET(SIGN_OPTIONS " /a /t http://timestamp.verisign.com/scripts/timstamp.dll ")
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CLIENT_LIB_DIR}/libmariadb.dll)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CLIENT_LIB_DIR}/libmariadb.lib)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CLIENT_LIB_DIR}/mariadbclient.lib)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CLIENT_DBG_DIR}/libmariadb.dll)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CLIENT_DBG_DIR}/libmariadb.lib)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CLIENT_DBG_DIR}/mariadbclient.lib)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CMAKE_BINARY_DIR}/plugins/auth/${CMAKE_BUILD_TYPE}/dialog.dll)
  EXECUTE_PROCESS(
    COMMAND signtool sign ${SIGN_OPTIONS} ${CMAKE_BINARY_DIR}/plugins/auth/${CMAKE_BUILD_TYPE}/mysql_clear_password.dll)
ENDIF()

SET(MARIADB_PLUGINS "${MARIADB_PLUGINS} <File Id=\"dialog.dll\" Name=\"dialog.dll\" DiskId=\"1\" Source=\"${CMAKE_BINARY_DIR}/plugins/auth/${CMAKE_BUILD_TYPE}/dialog.dll\"/>\n")
SET(MARIADB_PLUGINS "${MARIADB_PLUGINS} <File Id=\"mysql_clear_password.dll\" Name=\"mysqlclearpassword.dll\" DiskId=\"1\" Source=\"${CMAKE_BINARY_DIR}/plugins/auth/${CMAKE_BUILD_TYPE}/mysql_clear_password.dll\"/>\n")


FOREACH(src ${MARIADB_CLIENT_INCLUDES})
  STRING(REPLACE "-" "_" src_id ${src})
  STRING(REPLACE "mysql/" "" src_id ${src_id})
  STRING(REPLACE "mysql/" "" src_name ${src})
  IF(${src} MATCHES "mysql/")
    SET(MARIADB_INCLUDEMYSQL_FILES "${MARIADB_INCLUDEMYSQL_FILES} <File Id=\"${src_id}\" Name=\"${src_name}\"  DiskId=\"1\"  Source=\"${CMAKE_SOURCE_DIR}/include/${src}\"/>\n")
  ELSE()
    SET(MARIADB_INCLUDE_FILES "${MARIADB_INCLUDE_FILES} <File Id=\"${src_id}\" Name=\"${src_name}\"  DiskId=\"1\"  Source=\"${CMAKE_SOURCE_DIR}/include/${src}\"/>\n")
  ENDIF()
ENDFOREACH()


IF(NOT WIX_DIR)
  SET(WIX_DIR $ENV{WIX}/bin)
ENDIF()

CONFIGURE_FILE(${CMAKE_SOURCE_DIR}/win/packaging/mariadb-connector-c.xml.in
  ${CMAKE_BINARY_DIR}/win/packaging/mariadb-connector-c.xml)
SET(MSI_PACKAGE "mariadb-connector-c-${PRODUCT_VERSION}-${PLATFORM}.msi")

IF(WITH_SIGNCODE)
  ADD_CUSTOM_TARGET(SIGNMSI
    DEPENDS ${MSI_PACKAGE}
    COMMAND signtool sign ${SIGN_OPTIONS} ${MSI_PACKAGE})
  ADD_DEPENDENCIES(SIGNMSI ${MSI_PACKAGE})
  SET_TARGET_PROPERTIES(SIGNMSI PROPERTIES EXCLUDE_FROM_ALL OFF)
ENDIF()

ADD_CUSTOM_TARGET(
        ${MSI_PACKAGE}
        DEPENDS WIXOBJ
#mariadb-connector-c.wixobj
        COMMAND ${WIX_DIR}/light.exe -ext WixUIExtension mariadb-connector-c.wixobj -o ${MSI_PACKAGE})

ADD_CUSTOM_TARGET(WIXOBJ
        DEPENDS mariadb-connector-c.xml
        COMMAND ${WIX_DIR}/candle.exe mariadb-connector-c.xml -o mariadb-connector-c.wixobj)

SET_TARGET_PROPERTIES(${MSI_PACKAGE} PROPERTIES EXCLUDE_FROM_ALL OFF)
SET_TARGET_PROPERTIES(${WIXOBJ} PROPERTIES EXCLUDE_FROM_ALL OFF)
ADD_DEPENDENCIES(${MSI_PACKAGE} WIXOBJ)
ADD_DEPENDENCIES(WIXOBJ libmariadb mariadbclient dialog mysql_clear_password) 


