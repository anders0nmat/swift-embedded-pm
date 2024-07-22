idf_component_register(
	SRCS /dev/null
	INCLUDE_DIRS ".")

idf_build_get_property(TARGET IDF_TARGET)
idf_build_get_property(ARCH IDF_TARGET_ARCH)

if(NOT "${ARCH}" STREQUAL "riscv")
	message(FATAL_ERROR "Architecture '${ARCH}' of target '${TARGET}' not supported")
endif()



# The target swift will be building for
set(SWIFT_TARGET "riscv32-none-none-eabi")

# The target of the host OS, required for sourcekitLSP to work
set(SWIFT_LSP_TARGET "x86_64-unknown-linux-gnu")

# Name of the library product in Package.swift
set(SWIFT_PRODUCT_NAME "swiftcode")



if(${TARGET} STREQUAL "esp32c2" OR ${TARGET} STREQUAL "esp32c3")
    set(MARCH_FLAG "rv32imc_zicsr_zifencei")
    set(MABI_FLAG "ilp32")
elseif(${TARGET} STREQUAL "esp32p4")
    set(MARCH_FLAG "rv32imafc_zicsr_zifencei")
    set(MABI_FLAG "ilp32f")
else()
    set(MARCH_FLAG "rv32imac_zicsr_zifencei")
    set(MABI_FLAG "ilp32")
endif()

# Get includes for c-std libraries
set(SWIFT_INCLUDES)
foreach(dir ${CMAKE_C_IMPLICIT_INCLUDE_DIRECTORIES})
	list(APPEND SWIFT_INCLUDES "-Xcc" "-I${dir}")
endforeach()
foreach(dir ${CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES})
	list(APPEND SWIFT_INCLUDES "-Xcc" "-I${dir}")
endforeach()
# Get includes for extra components
foreach(dir ${INCLUDE_DIRECTORIES})
	list(APPEND SWIFT_INCLUDES "-Xcc" "-I${dir}")	
endforeach()

# Define output file of swift package
set(SWIFT_PRODUCT_BUILD   "${COMPONENT_DIR}/.build")
set(SWIFT_PRODUCT_RELEASE "${SWIFT_PRODUCT_BUILD}/release")
set(SWIFT_PRODUCT_ARCHIVE "${SWIFT_PRODUCT_RELEASE}/lib${SWIFT_PRODUCT_NAME}.a")

# Generate Archive
add_custom_target(swift-archive
	COMMAND
		# Remove the archive. Swift does not overwrite this on building but merges them.
		# If the file with app_main gets renamed, both objectfiles (old and new) will be in the archive, potentially causing problems
		rm -f ${SWIFT_PRODUCT_ARCHIVE}
		&&
		swift package update
			--package-path ${COMPONENT_DIR}	
		&&
		swift build
			--configuration=release
			--package-path ${COMPONENT_DIR}
			
			--triple ${SWIFT_TARGET}
			-Xswiftc -enable-experimental-feature -Xswiftc Embedded
			-Xswiftc -wmo
			-Xswiftc -parse-as-library
			-Xswiftc -Osize
			
			-Xswiftc -Xfrontend -Xswiftc -function-sections

			-Xcc -march=${MARCH_FLAG}
			-Xcc -mabi=${MABI_FLAG}
			
			# Get includes for C-std libraries and extra components
			${SWIFT_INCLUDES}

			# Get includes for ESP-IDF Components
			$$\( echo '$<TARGET_PROPERTY:__idf_main,INCLUDE_DIRECTORIES>' | tr '\;' '\\n' | sed -e 's/\\\(.*\\\)/-Xcc -I\\1/g' \)
	BYPRODUCTS
		${SWIFT_PRODUCT_ARCHIVE}
)

# Generate all necessary include paths for LSP
add_custom_target(swift-compile-flags
	COMMAND
		echo '' &&
		\( echo '${SWIFT_INCLUDES}' && \( echo '$<TARGET_PROPERTY:__idf_main,INCLUDE_DIRECTORIES>' | tr '\;' '\\n' | sed -e 's/\\\(.*\\\)/-Xcc -I\\1/g' \) \)
		| tr ' ' '\\n' | grep '^-I' > ${SWIFT_PRODUCT_RELEASE}/compile_flags.txt
	BYPRODUCTS
		${SWIFT_PRODUCT_RELEASE}/compile_flags.txt
)

# Create needed symlinks so LSP finds the module information
add_custom_target(swift-lsp-symlink
	COMMAND
		ln -sfT ${SWIFT_PRODUCT_BUILD}/${SWIFT_TARGET} ${SWIFT_PRODUCT_BUILD}/${SWIFT_LSP_TARGET}
		&& ln -sfT ${SWIFT_PRODUCT_BUILD}/${SWIFT_TARGET}/release ${SWIFT_PRODUCT_BUILD}/${SWIFT_TARGET}/debug
	DEPENDS
		swift-archive
)

# Extract Objectfile with app_main() symbol
# So we can append it to __idf_main and not (just) as archive
# This is so the linker will correctly resolve dependencies:
# * of FreeRTOS to app_main()
# * from app_main() to other components
add_custom_command(
	OUTPUT ${SWIFT_PRODUCT_RELEASE}/_main_swiftcode.o
	COMMAND
		# Extract first .o which defines the 'app_main' symbol
		ar x ${SWIFT_PRODUCT_ARCHIVE} $$\( nm --defined-only -A ${SWIFT_PRODUCT_ARCHIVE} | grep -m 1 ' T app_main' | cut -d: -f2 \) --output ${SWIFT_PRODUCT_RELEASE}
		# Rename it to a predicatble file
		&& mv -f -T ${SWIFT_PRODUCT_RELEASE}/$$\( nm --defined-only -A ${SWIFT_PRODUCT_ARCHIVE} | grep -m 1 ' T app_main' | cut -d: -f2 \) ${SWIFT_PRODUCT_RELEASE}/_main_swiftcode.o
	DEPENDS 
		swift-archive
		${SWIFT_PRODUCT_ARCHIVE}
		swift-compile-flags
		${SWIFT_PRODUCT_RELEASE}/compile_flags.txt
		swift-lsp-symlink
)
add_custom_target(main-swiftcode DEPENDS ${SWIFT_PRODUCT_RELEASE}/_main_swiftcode.o)

target_link_libraries(__idf_main ${SWIFT_PRODUCT_ARCHIVE})
target_link_libraries(__idf_main ${SWIFT_PRODUCT_RELEASE}/_main_swiftcode.o)

add_dependencies(__idf_main main-swiftcode)

