
# find the OS
uname_S := $(shell sh -c 'uname -s 2>/dev/null || echo not')

# Compiler and YACC compiler
CC:=$(shell sh -c 'type $(CC) >/dev/null 2>/dev/null && echo $(CC) || echo gcc')
YACC=$(shell which lemon)
export CC
export YACC

# if DEBUG env var is set, we compile with "debug" cflags
DEBUGFLAGS = -g -ggdb -O3 
ifeq ($(DEBUG), 1)
	DEBUGFLAGS = -g -ggdb -O0
endif

# Default CFLAGS
CFLAGS= -Wall -Wno-unused-function -Wno-unused-variable -Wno-unused-result -fPIC \
 	-D_GNU_SOURCE -std=gnu99 -I"$(shell pwd)" -DREDIS_MODULE_TARGET \
	-DREDISMODULE_EXPERIMENTAL_API 
CFLAGS += $(DEBUGFLAGS)

# Compile flags for linux / osx
ifeq ($(uname_S),Linux)
	SHOBJ_LDFLAGS ?= -shared -Bsymbolic -Bsymbolic-functions -ldl -lpthread
else
	CFLAGS += -mmacosx-version-min=10.6
	SHOBJ_LDFLAGS ?= -macosx_version_min 10.6 -exported_symbol _RedisModule_OnLoad -bundle -undefined dynamic_lookup -ldl -lpthread
endif
export CFLAGS

# Sources
SOURCEDIR=$(shell pwd -P)
CC_SOURCES = $(wildcard $(SOURCEDIR)/*.c) 
CC_SOURCES += $(wildcard $(SOURCEDIR)/query_parser/*.c) 
CC_SOURCES += $(wildcard $(SOURCEDIR)/ext/*.c) 
CC_SOURCES += $(wildcard $(SOURCEDIR)/util/*.c) 
CC_SOURCES += $(wildcard $(SOURCEDIR)/trie/*.c) 
CC_SOURCES += $(wildcard $(SOURCEDIR)/dep/thpool/*.c) 

# Convert all sources to .o files
CC_OBJECTS = $(patsubst %.c, %.o, $(CC_SOURCES) )

# Library dependencies
LIBRMUTIL=rmutil/librmutil.a
LIBTRIE=trie/libtrie.a
LIBTRIEMAP=dep/triemap/libtriemap.a
LIBNU=dep/libnu/libnu.a
LIBSTEMMER=dep/snowball/libstemmer.o

# Compilation deps for the module
MODULE=$(CC_OBJECTS) $(LIBTRIE) $(LIBTRIEMAP) $(LIBRMUTIL) $(LIBNU) $(LIBSTEMMER)

%.c: %.y

all: redisearch.so


# Library compile rules
$(LIBRMUTIL):
	$(MAKE) -C rmutil
.PHONY: $(LIBRMUTIL)
$(LIBTRIE):
	$(MAKE) -C trie
.PHONY: $(LIBTRIE)
$(LIBTRIEMAP):
	$(MAKE) -C dep/triemap
.PHONY: $(LIBTRIEMAP)
$(LIBSTEMMER):
	$(MAKE) -C dep/snowball libstemmer.o
.PHONY: $(LIBSTEMMER)
$(LIBNU):
	$(MAKE) -C dep/libnu
.PHONY: $(LIBNU)

# Compile query parse .y and .rl files. 
# This is not included in the usual make target!
query_parser:
	$(MAKE) -C $@
.PHONY: query_parser

build_tests: redisearch.so libredisearch.a
	$(MAKE) -C tests build

test: $(MODULE) build_tests
	# low level redis-independant tests
	$(MAKE) -C ./tests test
	# high level python integration tests
	$(MAKE) -C pytest test
	# Triemap tests
	$(MAKE) -C dep/triemap/test test

buildall: redisearch.so libredisearch.a build_tests

# Build the module...
redisearch.so: $(MODULE) version.h
	# Just to make sure old versions of the modules are deleted
	rm -f module.so
	$(LD) -o $@ $(MODULE) $(SHOBJ_LDFLAGS) -lc -lm

# Build a stand-alone static library without the module entry point.
# This is used to include the module's functionality in other modules
libredisearch.a: CFLAGS+=-DRS_CMD_PREFIX='"_FT"' -DRS_STATIC
libredisearch.a: $(CC_OBJECTS) version.h
	ar rcs $@ $(CC_OBJECTS)

staticlib: libredisearch.a

clean:
	rm -fv *.o trie/*.o query_parser/*.o ext/*.o
	$(MAKE) -C tests clean

distclean:
	find . -type f \( -name '*.o' -o -name '*.a' -o -name '*.so' \) -delete -print

package: redisearch.so
	mkdir -p ../build
	module_packer -v -o "../build/redisearch.{os}-{architecture}.latest.zip" "`pwd`/redisearch.so"

rebuild: clean all
