# GDCM port to WebAssembly [UNOFFICIAL]

This is a humble guide to port standard [GDCM](https://github.com/malaterre/GDCM) applications to WebAssembly (WASM) using [Emscripten](https://emscripten.org/index.html). It covers compilation of these apps:
- gdcmanon
- gdcmclean
- gdcmconv
- gdcmdiff
- gdcmdump
- gdcmgendir
- gdcmimg
- gdcminfo
- gdcmpap3
- gdcmraw
- gdcmscanner
- gdcmscu
- gdcmtar
- gdcmxml

This guide compiles each application to a pair of files (e.g. gdcmconv.js & gdcmconv.wasm). They are modularized in order to use them in *Flutter* via [universal_ffi](https://pub.dev/packages/universal_ffi) package or more specifically it's [wasm_ffi](https://pub.dev/packages/wasm_ffi) subpackage. You should get familiar with whether your usecase needs these apps to be modularized. For an example, I observed the best way to use them with *node* commands is to not set modularize altogether.

This guide is inspired by [GDCM-JS](https://github.com/InfiniBrains/gdcm-js) and I used the [Leo AI](https://brave.com/leo/) to help me write modifications to *CMakeLists.txt* faster. The [GDCM-JS](https://github.com/InfiniBrains/gdcm-js) project is very outdated in terms of using deprecated and no longer existing [Emscripten](https://emscripten.org/index.html) commands and complex logic to link libraries.

Thus far I succeeded compiling [GDCM](https://github.com/malaterre/GDCM) apps to WASM, importing them into a *Flutter* web app and calling apps with *--version* parameter. I haven't confirmed how *DICOM* files work yet, but will do so in the near future.

---

## Installation

- Install Emscripten from the [official website](https://emscripten.org/docs/getting_started/downloads.html)
- Download the latest GDCM *Source code (zip/tar.gz)* from their [release page](https://github.com/malaterre/GDCM/releases/) and uncompress it in a location (e.g. */home/user/Downloads/GDCM-3.2.6*)
- Open to edit the */home/user/DOwnloads/GDCM-3.2.6/Applications/Cxx/CMakeLists.txt* file:
  - Find the *foreach(exename ${GDCM_EXECUTABLE_NAME})* block
  - At the end of this block add the next block:
```
# --- ADD THIS BLOCK ---
  # Check if building with Emscripten to apply WASM-specific flags
  if(EMSCRIPTEN)
    # Correct format: A single string representing a JSON array
    # No extra quotes around the whole thing inside the flag
    set(GDCM_EXPORTED_FUNCTIONS_FLAG "-sEXPORTED_FUNCTIONS=[\"_main\",\"_malloc\",\"_free\"]")

    # 1. Construct the full flag string with the variable substituted
    # Result example: "-sEXPORT_NAME='createGDCMgdcminfo'"
    # You can also use just "${exename}" if you want the JS function to be named exactly "gdcminfo"
    set(GDCM_EXPORT_NAME_FLAG "-sEXPORT_NAME='${exename}'")

    # Use target_link_options for modern CMake (3.13+)
    # Note: No space between -s and the option name to avoid CMake parsing issues
    target_link_options(${exename} PRIVATE
      -sMODULARIZE=1
      -sALLOW_MEMORY_GROWTH=1
      -sEXPORTED_RUNTIME_METHODS=HEAPU8
      ${GDCM_EXPORTED_FUNCTIONS_FLAG}
      ${GDCM_EXPORT_NAME_FLAG}
    )
  endif()
```
  - The complete block at the time of this writing would look like this:
```
foreach(exename ${GDCM_EXECUTABLE_NAME})
  if(${exename} STREQUAL "gdcminfo")
    add_executable(${exename} ${exename}.cxx puff.c)
  else()
    add_executable(${exename} ${exename}.cxx)
  endif()
  target_link_libraries(${exename} gdcmMSFF)
  if(${exename} STREQUAL "gdcmpdf")
    target_link_libraries(${exename} ${POPPLER_LIBRARIES})
  elseif(${exename} STREQUAL "gdcmxml")
    target_link_libraries(${exename} ${LIBXML2_LIBRARIES})
  elseif(${exename} STREQUAL "gdcmscu")
    if(NOT GDCM_USE_SYSTEM_SOCKETXX)
      target_link_libraries(${exename} gdcmMEXD socketxx)
    else()
      target_link_libraries(${exename} gdcmMEXD socket++)
    endif()
  elseif(${exename} STREQUAL "gdcmstream")
    target_link_libraries(${exename} ${GDCM_OPENJPEG_LIBRARIES})
  elseif(${exename} STREQUAL "gdcmdump")
    target_link_libraries(${exename} ${GDCM_ZLIB_LIBRARIES})
  elseif(${exename} STREQUAL "gdcminfo")
    if(GDCM_USE_SYSTEM_POPPLER)
      target_link_libraries(${exename} ${POPPLER_LIBRARIES})
    endif()
  endif()
  if(GDCM_EXECUTABLE_PROPERTIES)
    set_target_properties(${exename} PROPERTIES ${GDCM_EXECUTABLE_PROPERTIES})
  endif()
  if(WIN32 AND NOT CYGWIN AND NOT MINGW)
    target_link_libraries(${exename} gdcmgetopt)
  endif()
  if(NOT GDCM_INSTALL_NO_RUNTIME)
    install(TARGETS ${exename}
      EXPORT ${GDCM_TARGETS_NAME}
      RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT Applications
    )
  endif()
  # --- ADD THIS BLOCK ---
  # Check if building with Emscripten to apply WASM-specific flags
  if(EMSCRIPTEN)
    # Correct format: A single string representing a JSON array
    # No extra quotes around the whole thing inside the flag
    set(GDCM_EXPORTED_FUNCTIONS_FLAG "-sEXPORTED_FUNCTIONS=[\"_main\",\"_malloc\",\"_free\"]")

    # 1. Construct the full flag string with the variable substituted
    # Result example: "-sEXPORT_NAME='createGDCMgdcminfo'"
    # You can also use just "${exename}" if you want the JS function to be named exactly "gdcminfo"
    set(GDCM_EXPORT_NAME_FLAG "-sEXPORT_NAME='${exename}'")

    # Use target_link_options for modern CMake (3.13+)
    # Note: No space between -s and the option name to avoid CMake parsing issues
    target_link_options(${exename} PRIVATE
      -Oz
      -sMODULARIZE=1
      -sALLOW_MEMORY_GROWTH=1
      -sEXPORTED_RUNTIME_METHODS=HEAPU8
      ${GDCM_EXPORTED_FUNCTIONS_FLAG}
      ${GDCM_EXPORT_NAME_FLAG}
    )
  endif()
endforeach()
```
  - The block exports functions *_main*, *_malloc* and *_free* functions
  - It names the module to the app itself (e.g. *gdcmconv.js* -> *gdcmconv*)
  - It sets the compression to -Oz (for size or set -O3 for speed)
  - It modularizes the app
  - It allows memory growth in case of large *DICOM* inputs and outputs and 
  - It changes memory object to the *HEAPU8* per [wasm_ffi](https://pub.dev/packages/wasm_ffi) instructions.
    - **Note**: You may change any of these values, none are required.
- mkdir /home/user/Downloads/GDCM-3.2.6/build
- cd /home/user/Downloads/GDCM-3.2.6/build
- emcmake cmake ../ -DGDCM_BUILD_APPLICATIONS=1 -DEMSCRIPTEN=1 -DCMAKE_BUILD_TYPE=Release
  -DGDCM_* flags are available in the [GDCM documentation](https://github.com/malaterre/GDCM/blob/master/INSTALL.txt)
- emmake make

There may be some warnings along the way, but for now, I ignore them (some non-safe pointer warnings). If everything compiled successfully, resulting apps reside in */home/user/Downloads/GDCM-3.2.6/build/bin/* directory.
