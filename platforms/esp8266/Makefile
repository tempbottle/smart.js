# Makefile for ESP8266 projects
#
# Thanks to:
# - zarya
# - Jeroen Domburg (Sprite_tm)
# - Christian Klippel (mamalala)
# - Tommie Gannert (tommie)
#
# Changelog:
# - 2014-10-06: Changed the variables to include the header file directory
# - 2014-10-06: Added global var for the Xtensa tool root
# - 2014-11-23: Updated for SDK 0.9.3
# - 2014-12-25: Replaced esptool by esptool.py
# Output directors to store intermediate compiled files
# relative to the project directory

ESPPORT		?= /dev/ttyACM0

BUILD_BASE	= build
FW_BASE		= firmware

# base directory for the V7 sources
V7_DIR = ../../src

# base directory for the compiler
XTENSA_TOOLS_ROOT ?= /opt/Espressif/crosstool-NG/builds/xtensa-lx106-elf/bin

# base directory of the ESP8266 SDK package, absolute
SDK_BASE        ?= /opt/Espressif/ESP8266_SDK

# esptool.py path and port
ESPTOOL		?= esptool.py

# name for the target project
TARGET		= app

# which modules (subdirectories) of the project to include in compiling
MODULES		= driver user spiffs
EXTRA_INCDIR    = include

# libraries used in this project, mainly provided by the SDK
LIBS		= m cirom gcc hal pp phy net80211 lwip wpa main

# spiffs parameters
FW_FS_ADDR = 0x20000
FW_FS_SIZE = 32768

# compiler flags using during compilation of source files
CFLAGS		= -Os -g -Wpointer-arith -Wl,-EL -fno-inline-functions -nostdlib -mlongcalls -mtext-section-literals  -D__ets__ -DICACHE_FLASH -DV7_BUILD_PROFILE=1 -DNO_LIBC -DON_FLASH='__attribute__((section(".irom0.text")))' -DRODATA='__attribute__((section(".irom.text")))' -DV7_DISABLE_AST_TAG_NAMES -DV7_TEMP_OFF -DV7_ENABLE_GC -DV7_ENABLE_COMPACTING_GC -DDISABLE_MD5 -DDISABLE_SHA1X -DENDL='"\n\r"' -Dprintf=c_printf $(CFLAGS_EXTRA) -DFS_ADDR=$(FW_FS_ADDR) -DFS_SIZE=$(FW_FS_SIZE) -DV7_ENABLE_FILE -Dc_fread=spiffs_fread -Dc_fwrite=spiffs_fwrite -Dc_fclose=spiffs_fclose -Dc_fopen=spiffs_fopen -Dc_rename=spiffs_rename -Dc_remove=spiffs_remove -Dc_file_t=int -DINVALID_FILE=-1 -Dc_fseek=spiffs_fseek -Dc_ftell=spiffs_ftell -Dc_rewind=spiffs_rewind -Dc_ferror=spiffs_ferror

#CFLAGS		= -Os -g -O2 -Wpointer-arith -Wundef -Werror -Wl,-EL -fno-inline-functions -nostdlib -mlongcalls -mtext-section-literals  -D__ets__ -DICACHE_FLASH
# linker flags used to generate the main object file
LDFLAGS		= -nostdlib -Wl,--no-check-sections -u call_user_start -Wl,-static

# linker script used for the above linkier step
LD_SCRIPT	= eagle.app.30000.v6.ld

# various paths from the SDK used in this project
SDK_LIBDIR	= lib
SDK_INCDIR	= include include/json
SDK_LDDIR	= ld
# we create two different files for uploading into the flash
# these are the names and options to generate them
FW_FILE_1_ADDR	= 0x00000
FW_FILE_2_ADDR	= 0x30000

# select which tools to use as compiler, librarian and linker
CC:= $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-gcc
AR:= $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-ar
LD:= $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-gcc

####
#### no user configurable options below here
####
SRC_DIR		:= $(MODULES)
BUILD_DIR	:= $(addprefix $(BUILD_BASE)/,$(MODULES))

SDK_LIBDIR	:= $(addprefix $(SDK_BASE)/,$(SDK_LIBDIR))
SDK_INCDIR	:= $(addprefix -I$(SDK_BASE)/,$(SDK_INCDIR))

SRC		:= $(foreach sdir,$(SRC_DIR),$(wildcard $(sdir)/*.c))
OBJ		:= $(patsubst %.c,$(BUILD_BASE)/%.o,$(SRC))
LIBS		:= $(addprefix -l,$(LIBS))
APP_AR		:= $(addprefix $(BUILD_BASE)/,$(TARGET)_app.a)
TARGET_OUT	:= $(addprefix $(BUILD_BASE)/,$(TARGET).out)

LD_SCRIPT	:= $(addprefix -T./user/,$(LD_SCRIPT))

INCDIR	:= $(addprefix -I,$(SRC_DIR)) 
EXTRA_INCDIR	:= $(addprefix -I,$(EXTRA_INCDIR))
MODULE_INCDIR	:= $(addsuffix /include,$(INCDIR))

FW_FILE_1	:= $(addprefix $(FW_BASE)/,$(FW_FILE_1_ADDR).bin)
FW_FILE_2	:= $(addprefix $(FW_BASE)/,$(FW_FILE_2_ADDR).bin)
FW_FILE_FS	:= $(addprefix $(FW_BASE)/,$(FW_FS_ADDR).bin)

UNAME_S := $(shell uname -s)

# users might need to build on docker and flash on local.
# flash rule might trigger rebuilding of spiffs image
# and thus the binaries have to have a distinct name
MKSPIFFS=mkspiffs_$(UNAME_S)

SPIFFS_SRC=$(wildcard spiffs/*.c)

V ?= $(VERBOSE)
ifeq ("$(V)","1")
Q :=
vecho := @true
else
Q := @
vecho := @echo
endif

vpath %.c $(SRC_DIR)

define compile-objects
$1/%.o: %.c
	$(vecho) "CC $$<"
	$(Q) $(CC) $(INCDIR) $(MODULE_INCDIR) $(EXTRA_INCDIR) $(SDK_INCDIR) $(CFLAGS) -c $$< -o $$@
endef

.PHONY: all checkdirs flash clean

v7: user/v7.c user/v7.h
	@$(MAKE) all

all: checkdirs $(TARGET_OUT) $(FW_FILE_1) $(FW_FILE_2) $(FW_FILE_FS)

$(FW_BASE)/%.bin: $(TARGET_OUT) | $(FW_BASE)
	$(vecho) "FW $(FW_BASE)/"
	$(Q) $(ESPTOOL) elf2image -o $(FW_BASE)/ $(TARGET_OUT)

$(TARGET_OUT): $(APP_AR)
	$(vecho) "LD $@"
	$(Q) $(LD) -L$(SDK_LIBDIR) $(LD_SCRIPT) $(LDFLAGS) -Wl,--start-group $(LIBS) $(APP_AR) -Wl,--end-group -o $@

$(APP_AR): $(OBJ)
	$(vecho) "AR $@"
	$(Q) $(AR) cru $@ $^

checkdirs: $(BUILD_DIR) $(FW_BASE)

$(BUILD_DIR):
	$(Q) mkdir -p $@

$(FW_BASE):
	$(Q) mkdir -p $@

$(FW_FILE_FS): $(MKSPIFFS) $(wildcard fs/*)
	./$(MKSPIFFS) $(FW_FS_SIZE) ./fs >$(FW_FILE_FS)

$(MKSPIFFS): mkspiffs.c $(SPIFFS_SRC)
	gcc -o $@ $^ -DON_FLASH="" -DNO_ESP

flash: $(FW_FILE_1) $(FW_FILE_2)
	$(ESPTOOL) --port $(ESPPORT) write_flash $(FW_FILE_1_ADDR) $(FW_FILE_1) $(FW_FILE_2_ADDR) $(FW_FILE_2) $(FW_FS_ADDR) $(FW_FILE_FS)

flash_no_fs: $(FW_FILE_1) $(FW_FILE_2)
	$(ESPTOOL) --port $(ESPPORT) write_flash $(FW_FILE_1_ADDR) $(FW_FILE_1) $(FW_FILE_2_ADDR) $(FW_FILE_2)

flash_fs: $(FW_FILE_FS)
	$(ESPTOOL) --port $(ESPPORT) write_flash $(FW_FS_ADDR) $(FW_FILE_FS)

clean:
	$(Q) rm -rf $(FW_BASE) $(BUILD_BASE) $(MKSPIFFS) user/v7.c user/v7.h

user/v7.c: $(V7_DIR)/v7.c
	cp $(V7_DIR)/v7.c user/v7.c

user/v7.h: $(V7_DIR)/v7.h
	cp $(V7_DIR)/v7.h user/v7.h

$(foreach bdir,$(BUILD_DIR),$(eval $(call compile-objects,$(bdir))))
