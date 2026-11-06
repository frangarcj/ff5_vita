cmake_minimum_required(VERSION 2.8)

if(NOT DEFINED CMAKE_TOOLCHAIN_FILE)
  if(DEFINED ENV{VITASDK})
    set(CMAKE_TOOLCHAIN_FILE "$ENV{VITASDK}/share/vita.toolchain.cmake" CACHE PATH "toolchain file")
  else()
    message(FATAL_ERROR "Please define VITASDK to point to your SDK path!")
  endif()
endif()

project(FF5.elf C CXX)
include("${VITASDK}/share/vita.cmake" REQUIRED)
set(VITA_APP_NAME "Final Fantasy V")
set(VITA_TITLEID  "FF5000000")
set(VITA_VERSION  "01.01")
set(VITA_MKSFOEX_FLAGS "-d ATTRIBUTE2=12")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -Wl,-q,--wrap,memcpy,--wrap,memmove,--wrap,memset -Wall -O3 -mfloat-abi=softfp")
set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -std=c++11")

# Resolve the target libraries, including libsndfile's optional static codecs.
set(PKG_CONFIG_EXECUTABLE "${VITASDK}/bin/arm-vita-eabi-pkg-config")
find_package(PkgConfig REQUIRED)
pkg_check_modules(FF5_SNDFILE REQUIRED sndfile)
pkg_check_modules(FF5_ICU REQUIRED icu-uc)

# ICU loads versioned runtime data from app0:/. Package the data belonging to
# the linked SDK library; the bundled data remains available for ICU 59 SDKs.
string(REGEX MATCH "^[0-9]+" FF5_ICU_MAJOR_VERSION "${FF5_ICU_VERSION}")
set(FF5_ICU_DATA_NAME "icudt${FF5_ICU_MAJOR_VERSION}l")
set(FF5_ICU_DATA_DIR "${VITASDK}/arm-vita-eabi/share/icu/${FF5_ICU_VERSION}/${FF5_ICU_DATA_NAME}")
if(NOT IS_DIRECTORY "${FF5_ICU_DATA_DIR}" AND
   IS_DIRECTORY "${CMAKE_SOURCE_DIR}/${FF5_ICU_DATA_NAME}")
  set(FF5_ICU_DATA_DIR "${CMAKE_SOURCE_DIR}/${FF5_ICU_DATA_NAME}")
endif()
set(FF5_ICU_DATA_FILES)
foreach(ICU_DATA_FILE cnvalias.icu confusables.cfu unames.icu
                      ibm-943_P15A-2003.cnv ibm-5348_P100-1997.cnv)
  if(NOT EXISTS "${FF5_ICU_DATA_DIR}/${ICU_DATA_FILE}")
    message(FATAL_ERROR "Missing ICU ${FF5_ICU_VERSION} runtime data: ${FF5_ICU_DATA_DIR}/${ICU_DATA_FILE}")
  endif()
  list(APPEND FF5_ICU_DATA_FILES
       "${FF5_ICU_DATA_DIR}/${ICU_DATA_FILE}" "${FF5_ICU_DATA_NAME}/${ICU_DATA_FILE}")
endforeach()

add_executable(FF5.elf
  loader/main.c
  loader/dialog.c
  loader/so_util.c
  loader/bridge.c
  loader/stb_image.c
  loader/stb_truetype.c
)

target_link_libraries(FF5.elf
  vitaGL
  vitashark
  SceShaccCgExt
  mathneon
  OpenSLES
  ${FF5_SNDFILE_STATIC_LDFLAGS}
  ${FF5_ICU_STATIC_LDFLAGS}
  z
  m
  stdc++
  kubridge_stub
  taihen_stub
  SceAppMgr_stub
  SceAppUtil_stub
  SceAudio_stub
  SceCtrl_stub
  SceCommonDialog_stub
  SceDisplay_stub
  SceKernelDmacMgr_stub
  SceGxm_stub
  SceShaccCg_stub
  SceSysmodule_stub
  ScePower_stub
  SceTouch_stub
  SceVshBridge_stub
  SceHid_stub
  -Wl,--whole-archive
  pthread
  -Wl,--no-whole-archive
)

vita_create_self(eboot.bin FF5.elf UNSAFE)

add_executable(companion
  companion/main.cpp
)

target_link_libraries(companion
  imgui
  vitaGL
  vitashark
  SceShaccCgExt
  mathneon
  taihen_stub
  SceAppMgr_stub
  SceAudio_stub
  SceCtrl_stub
  SceCommonDialog_stub
  SceDisplay_stub
  SceKernelDmacMgr_stub
  SceGxm_stub
  SceShaccCg_stub
  SceSysmodule_stub
  SceTouch_stub
  SceVshBridge_stub
)

vita_create_self(companion.bin companion UNSAFE)

vita_create_vpk(FF5.vpk ${VITA_TITLEID} eboot.bin
  VERSION ${VITA_VERSION}
  NAME ${VITA_APP_NAME}
  FILE ${CMAKE_SOURCE_DIR}/sce_sys/icon0.png sce_sys/icon0.png
       ${CMAKE_SOURCE_DIR}/sce_sys/livearea/contents/bg0.png sce_sys/livearea/contents/bg0.png
       ${CMAKE_SOURCE_DIR}/sce_sys/livearea/contents/config.png sce_sys/livearea/contents/config.png
       ${CMAKE_SOURCE_DIR}/sce_sys/livearea/contents/startup.png sce_sys/livearea/contents/startup.png
       ${CMAKE_SOURCE_DIR}/sce_sys/livearea/contents/template.xml sce_sys/livearea/contents/template.xml
       ${CMAKE_SOURCE_DIR}/NotoSansJP-Regular.ttf NotoSansJP-Regular.ttf
       ${CMAKE_SOURCE_DIR}/NotoSansSC-Regular.ttf NotoSansSC-Regular.ttf
       ${CMAKE_SOURCE_DIR}/NotoSansKR-Regular.ttf NotoSansKR-Regular.ttf
       ${CMAKE_BINARY_DIR}/companion.bin companion.bin
       ${FF5_ICU_DATA_FILES}
       ${CMAKE_SOURCE_DIR}/shaders/1_Negative_f.cg shaders/1_Negative_f.cg
       ${CMAKE_SOURCE_DIR}/shaders/1_Negative_v.cg shaders/1_Negative_v.cg
       ${CMAKE_SOURCE_DIR}/shaders/2_FXAA_f.cg shaders/2_FXAA_f.cg
       ${CMAKE_SOURCE_DIR}/shaders/2_FXAA_v.cg shaders/2_FXAA_v.cg
       ${CMAKE_SOURCE_DIR}/shaders/3_Sepia_f.cg shaders/3_Sepia_f.cg
       ${CMAKE_SOURCE_DIR}/shaders/3_Sepia_v.cg shaders/3_Sepia_v.cg
       ${CMAKE_SOURCE_DIR}/shaders/4_Greyscale_f.cg shaders/4_Greyscale_f.cg
       ${CMAKE_SOURCE_DIR}/shaders/4_Greyscale_v.cg shaders/4_Greyscale_v.cg
)

add_custom_target(copy
  COMMAND echo "destroy" | nc $(PSVITAIP) 1338
  COMMAND curl -T eboot.bin ftp://$(PSVITAIP):1337/ux0:/app/${VITA_TITLEID}/eboot.bin  DEPENDS eboot.bin
  COMMAND curl -T companion.bin ftp://$(PSVITAIP):1337/ux0:/app/${VITA_TITLEID}/companion.bin  DEPENDS companion.bin
)

add_custom_target(launch
  COMMAND echo "screen on" | nc $(PSVITAIP) 1338
  COMMAND echo "destroy" | nc $(PSVITAIP) 1338
  COMMAND curl -T eboot.bin ftp://$(PSVITAIP):1337/ux0:/app/${VITA_TITLEID}/eboot.bin   DEPENDS eboot.bin	
  COMMAND curl -T companion.bin ftp://$(PSVITAIP):1337/ux0:/app/${VITA_TITLEID}/companion.bin  DEPENDS companion.bin
  COMMAND echo "launch ${VITA_TITLEID}" | nc $(PSVITAIP) 1338
)
