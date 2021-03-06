# This file is only to verify KreMLib as a library, i.e. to place all
# *.checked files in this directory instead of a cache. This
# is to allow users to directly pick from these .checked files instead
# of rebuilding them. This Makefile assumes that everything else from
# the F* standard library is already built (and fails otherwise)

all: verify-all compile-all

################################################################################
# Verification & extraction                                                    #
################################################################################

ifdef FSTAR_HOME
  # Assume there is a F* source tree
  FSTAR_EXE=$(FSTAR_HOME)/bin/fstar.exe
else
  # Assume F* in the PATH
  FSTAR_EXE=fstar.exe
endif

EXTRACT_DIR = .extract

GENERIC_DIR = dist/generic
UINT128_DIR = dist/uint128
MINI_DIR = dist/minimal

FSTAR_OPTIONS += --record_hints --use_hints --use_two_phase_tc true --odir $(EXTRACT_DIR) \
  # --use_extracted_interfaces

INCLUDE_PATHS = ../runtime

FSTAR = $(FSTAR_EXE) $(FSTAR_OPTIONS) --cache_checked_modules \
  $(addprefix --include , $(INCLUDE_PATHS)) --cmi \
  --already_cached 'Prims FStar -FStar.Kremlin.Endianness LowStar'

# Note: not compatible with the OPAM layout until fstar can be queried for the
# location of ulib.
ROOTS = $(wildcard *.fst) $(wildcard *.fsti) $(wildcard ../runtime/*.fst) \
  FStar.UInt128.fst FStar.Date.fsti \
    FStar.HyperStack.IO.fst FStar.IO.fst FStar.Int.Cast.Full.fst \
    FStar.Bytes.fsti FStar.Dyn.fsti LowStar.Printf.fst

.PHONY: clean clean-c
clean: clean-c
	rm -rf *.checked *.source .depend

SHELL=/bin/bash

clean-c:
	rm -rf dist out extract-all */*.o

.depend: Makefile
	$(FSTAR) $(OTHERFLAGS) --dep full $(ROOTS) > $@

include .depend


%.checked:
	$(FSTAR) $(OTHERFLAGS) $< && \
	touch $@

verify-all: $(ALL_CHECKED_FILES)

$(EXTRACT_DIR)/%.krml: | .depend
	$(FSTAR) $(OTHERFLAGS) --codegen Kremlin \
	  --extract_module $(basename $(notdir $(subst .checked,,$<))) \
	  $(notdir $(subst .checked,,$<)) && \
	touch $@

KREMLIN_ARGS += -fparentheses -fcurly-braces -fno-shadow -header copyright-header.txt

# Default build
$(GENERIC_DIR)/Makefile.include: $(ALL_KRML_FILES) | $(wildcard c/*.c) .depend ../_build/src/Kremlin.native
	../krml $(KREMLIN_ARGS) -minimal -tmpdir $(GENERIC_DIR) \
	  -warn-error +9+11 -skip-compilation -extract-uints \
	  $(addprefix -add-include ,'<inttypes.h>' '"kremlib.h"' '"kremlin/internal/compat.h"' '"kremlin/internal/target.h"') \
	  -bundle FStar.UInt64+FStar.UInt32+FStar.UInt16+FStar.UInt8=[rename=FStar_UInt_8_16_32_64] \
	  -bundle C.Endianness= \
	  -bundle FStar.Reflection,FStar.Reflection.*,FStar.Tactics,FStar.Tactics.*,FStar.Range \
	  -library C,C.Endianness,C.Failure,C.Loops,FStar.BitVector,FStar.Bytes,FStar.Char,FStar.Int,FStar.Kremlin.Endianness,FStar.Math.Lib,FStar.ModifiesGen,FStar.Monotonic.Heap,FStar.Monotonic.HyperStack,FStar.Mul,FStar.Pervasives,FStar.Pervasives.Native,FStar.ST,FStar.UInt,FStar.UInt128,FStar.UInt63,LowStar.Printf \
	  $(filter-out fstar_uint128_msvc.c,$(notdir $(wildcard c/*.c))) \
	  -o libkremlib.a \
	  $^
	cp c/*.c $(GENERIC_DIR)/

# UInt128-only build
$(UINT128_DIR)/Makefile.include: $(ALL_KRML_FILES) | .depend ../_build/src/Kremlin.native
	../krml $(KREMLIN_ARGS) -minimal -tmpdir $(UINT128_DIR) -skip-compilation -extract-uints \
	  $(addprefix -add-include ,'<inttypes.h>' '<stdbool.h>' '"kremlin/internal/types.h"' '"kremlin/internal/target.h"') \
	  -bundle FStar.UInt128=* \
	  -ccopt -DKRML_VERIFIED_UINT128 \
	  -o libkremlib.a \
	  $^

# Minimalistic build
$(MINI_DIR)/Makefile.include: $(ALL_KRML_FILES) | c/fstar_uint128.c .depend ../_build/src/Kremlin.native
	mkdir -p $(dir $@)
	../krml $(KREMLIN_ARGS) -minimal -tmpdir $(MINI_DIR) -skip-compilation -extract-uints \
	  $(addprefix -add-include , \
	    '<inttypes.h>' '<stdbool.h>' \
	    '"kremlin/internal/compat.h"' \
	    '"kremlin/lowstar_endianness.h"' \
	    '"kremlin/internal/types.h"' \
	    '"kremlin/internal/target.h"') \
	  -bundle FStar.UInt64+FStar.UInt32+FStar.UInt16+FStar.UInt8=[rename=FStar_UInt_8_16_32_64] \
	  -bundle C.Endianness= \
	  -library FStar.UInt128 \
	  -bundle FStar.UInt128= \
	  -bundle '*,WindowsWorkaroundSigh' \
	  fstar_uint128.c \
	  -o libkremlib.a \
	  $^
	cp c/fstar_uint128.c $(dir $@)

################################################################################
# Compilation, only works after the stage above has run and C files exist      #
################################################################################

CFLAGS += -O3 -march=native -mtune=native
export CFLAGS

compile-all: compile-dist-generic compile-dist-uint128 compile-dist-minimal

.PHONY: compile-dist-%
compile-dist-%: dist/%/Makefile.include
	$(MAKE) -C dist/$* -f Makefile.basic
