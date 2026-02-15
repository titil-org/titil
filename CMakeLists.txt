cmake_minimum_required(VERSION 3.10)

# =========================================

# Set this source directory as the tree root for project.
set(ae2f_ProjRoot ${CMAKE_CURRENT_SOURCE_DIR} CACHE STRING "Tree root for project")
set(ae2f_BinRoot ${CMAKE_CURRENT_BINARY_DIR} CACHE STRING "Tree root for project")
set(ae2f_submod  .submod CACHE STRING "Relative path to submodule (subdirectory)")
set(ae2f_CoreScript_Wh ${ae2f_ProjRoot}/${ae2f_submod}/ae2f/Preproc)

# Execute git to fetch CoreScript when repository does not exist
if(NOT EXISTS ${ae2f_CoreScript_Wh})
	execute_process(
		COMMAND git clone https://codeberg.org/ae2f/Preproc
		${ae2f_CoreScript_Wh}
		)
endif()

# add subdirectory
if(NOT TARGET ae2f::Preproc)
	add_subdirectory(${ae2f_CoreScript_Wh} ${ae2f_submod}/ae2f/Preproc)
endif()

# ========================================

include(ExternalProject)

if(NOT EXISTS ${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi)
	execute_process(
		COMMAND git clone https://github.com/ncroxon/gnu-efi.git
		${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi
		)

endif()

message(STATUS "[titil] Processor: ${CMAKE_SYSTEM_PROCESSOR}")
if(NOT DEFINED CMAKE_SYSTEM_PROCESSOR)
	message(FATAL_ERROR "You need to set processor with -DCMAKE_SYSTEM_PROCESSOR=<target>")
endif()

if(NOT DEFINED LXIV_TDCOUNT)
	set(LXIV_TDCOUNT 1)
endif()

message(STATUS "[titil::LXIV_TDCOUNT] threadcount for make: ${LXIV_TDCOUNT}")

add_custom_target(
	gnu-efi ALL ${CMAKE_MAKE_PROGRAM} ARCH=${CMAKE_SYSTEM_PROCESSOR} -j${LXIV_TDCOUNT}
	WORKING_DIRECTORY      ${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi
	VERBATIM
	BYPRODUCTS
	${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi/${CMAKE_SYSTEM_PROCESSOR}/gnuefi/crt0-efi-${CMAKE_SYSTEM_PROCESSOR}.o 
	${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi/${CMAKE_SYSTEM_PROCESSOR}/gnuefi/libgnuefi.a
	${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi/${CMAKE_SYSTEM_PROCESSOR}/lib/libefi.a
	)

add_custom_target(clean-all
	COMMAND		${CMAKE_MAKE_PROGRAM} clean
	WORKING_DIRECTORY ${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi
	VERBATIM
	)

# ========================================

project(titil)
set(titilgnuefi ${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi)

file(GLOB_RECURSE titil-src "${CMAKE_SOURCE_DIR}/app/*")

add_library(titil-obj OBJECT ${titil-src})
target_compile_options(titil-obj PRIVATE 
	-fpic -ffreestanding -fno-stack-protector 
	-fno-stack-check -fshort-wchar 
	-mno-red-zone -maccumulate-outgoing-args
	-DEFI_FUNCTION_WRAPPER
	)

target_include_directories(titil-obj PUBLIC ${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi/inc)
target_include_directories(titil-obj PUBLIC ${ae2f_ProjRoot}/${ae2f_submod}/ncroxon/gnu-efi/inc/${CMAKE_SYSTEM_PROCESSOR})
add_dependencies(titil-obj gnu-efi)

# ========================================

option(titil-qemu "" ON)
if(${titil-qemu})
	message(STATUS "[titil-qemu] qemu test is activated")
	file(MAKE_DIRECTORY	${CMAKE_CURRENT_BINARY_DIR}/efitest/EFI/BOOT)
	set(output_efi ${CMAKE_CURRENT_BINARY_DIR}/efitest/EFI/BOOT/BOOTX64.EFI)
	add_custom_command(
		MAIN_DEPENDENCY	${CMAKE_CURRENT_BINARY_DIR}/titil.efi
		COMMAND	${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_BINARY_DIR}/titil.efi ${output_efi}
		OUTPUT	${output_efi}
		COMMENT	"[titil-qemu] copying efi for test"
		)

	message(STATUS "[titil-qemu] LXIV_OVMF_CODE_FD: ${LXIV_OVMF_CODE_FD}")
	message(STATUS "[titil-qemu] LXIV_OVMF_VARS_FD: ${LXIV_OVMF_VARS_FD}")

	if(NOT DEFINED LXIV_OVMF_CODE_FD)
		message(FATAL_ERROR "[titil-qemu] You need to define LXIV_OVMF_CODE_FD")
	endif()

	if(NOT DEFINED LXIV_OVMF_VARS_FD)
		message(FATAL_ERROR "[titil-qemu] You need to define LXIV_OVMF_VARS_FD")
	endif()

	configure_file(
		${CMAKE_CURRENT_SOURCE_DIR}/test-qemu.sh.in
		${CMAKE_CURRENT_BINARY_DIR}/test-qemu.sh
		)
else()
	set(output_efi ${CMAKE_CURRENT_BINARY_DIR}/titil.efi)
endif()

# ========================================

message(STATUS "[titil] linker: ${CMAKE_LINKER}")

add_custom_command(
	DEPENDS	$<TARGET_OBJECTS:titil-obj>
	OUTPUT	${CMAKE_CURRENT_BINARY_DIR}/titil.so
	COMMAND ${CMAKE_LINKER}
	-shared -Bsymbolic
	-nostdlib
	-znocombreloc
	-L${titilgnuefi}/${CMAKE_SYSTEM_PROCESSOR}/lib
	-L${titilgnuefi}/${CMAKE_SYSTEM_PROCESSOR}/gnuefi
	-L${titilgnuefi}/${CMAKE_SYSTEM_PROCESSOR}/gnuefi
	-T${titilgnuefi}/gnuefi/elf_${CMAKE_SYSTEM_PROCESSOR}_efi.lds
	$<TARGET_OBJECTS:titil-obj>
	-o ${CMAKE_CURRENT_BINARY_DIR}/titil.so
	${titilgnuefi}/${CMAKE_SYSTEM_PROCESSOR}/gnuefi/crt0-efi-${CMAKE_SYSTEM_PROCESSOR}.o
	-l:libgnuefi.a
	-l:libefi.a
	VERBATIM
	)

add_custom_command(
	OUTPUT		${CMAKE_CURRENT_BINARY_DIR}/titil.efi 
	MAIN_DEPENDENCY ${CMAKE_CURRENT_BINARY_DIR}/titil.so
	COMMAND objcopy
	-j .text
	-j .sdata               
	-j .data                
	-j .rodata              
	-j .dynamic             
	-j .dynsym              
	-j .rel                 
	-j .rela                
	-j .reloc             
	--strip-debug
	--output-target=efi-app-${CMAKE_SYSTEM_PROCESSOR}
	${CMAKE_CURRENT_BINARY_DIR}/titil.so
	${CMAKE_CURRENT_BINARY_DIR}/titil.efi 
	VERBATIM
	)

add_custom_target(titil ALL SOURCES ${output_efi})
add_dependencies(titil titil-obj)

# ========================================
