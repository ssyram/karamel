include ../Makefile.common

RECENT_GCC	= $(shell [ "$$(gcc -dumpversion | cut -c -1)" -ge 5 ] && echo yes)

ifeq (3.81,$(MAKE_VERSION))
  $(error You seem to be using the OSX antiquated Make version. Hint: brew \
    install make, then invoke gmake instead of make)
endif

FSTAR_EXE ?= fstar.exe

# Note: these files have been moved in the F* repo in Dec 2023,
# so this has not been tested in a while.
ifdef FSTAR_HOME
CRYPTO = $(shell \
  if test -d $(FSTAR_HOME)/examples/low-level/crypto ; \
  then echo $(FSTAR_HOME)/examples/low-level/crypto ; \
  elif test -d $(FSTAR_HOME)/share/fstar/examples/low-level/crypto ; \
  then echo $(FSTAR_HOME)/share/fstar/examples/low-level/crypto ; \
  fi \
)
endif

ifneq ($(CRYPTO),)
  CRYPTO_OPTS	= -I $(CRYPTO) -I $(CRYPTO)/real
endif
TEST_OPTS	= -warn-error @4 -skip-makefiles
KRML_BIN	= ../_build/default/src/Karamel.exe
KRML		= $(KRML_BIN) -fstar $(FSTAR_EXE) $(KOPTS) $(TEST_OPTS)

# TODO: disambiguate between broken tests that arguably should work (maybe
# HigherOrder6?) and helper files that are required for multiple-module tests
# For Layered.test: see F* bug 2641 and associated PR
BROKEN		= \
  HigherOrder6.fst ForwardDecl.fst BlitNull.fst \
  Ctypes1.fst Ctypes2.fst Ctypes3.fst Ctypes4.fst \
  AbstractStructAbstract.fst MonomorphizationSeparate1.fst \
  Layered.fst GlobalInit2.fst $(wildcard Rust*.fst)

# Lowlevel is not really broken, but its test shouldn't be run since it's a
# special file and doesn't have a main.

ifneq ($(CRYPTO),)
  CRYPTO_FILES=$(CRYPTO)/Crypto.Symmetric.Chacha20.test
endif

FILES		= \
  $(patsubst %.fst,%.test,$(filter-out Unused_A.fst StaticHeaderAPI.fst \
    StaticHeaderLib.fst NameCollisionHelper.fst ML16Externals.fst MemCpyModel.fst \
    Lowlevel.fst PrivateInclude1.fst $(BROKEN),$(wildcard *.fst))) \
  $(CRYPTO_FILES) \
  $(patsubst %.fst,%.test,$(wildcard ../book/*.fst ../book/notfslit/*.fst))

ifneq ($(RECENT_GCC),"yes")
  FILES 	:= $(filter-out Debug.test,$(FILES))
endif

CUSTOM		= count-eq count-uu check-unused check-no-constructor check-no-erased \
  check-right-file count-deref check-global-init check-useless-alias check-inline-let \
  check-comment check-ignore check-aggressive-inline check-malloca

ifneq ($(CRYPTO),)
  CRYPTO_WASM_FILES=Crypto.Symmetric.Chacha20.wasm-test
endif
WASM_FILES	= \
  WasmTrap.wasm-test Wasm1.wasm-test Wasm2.wasm-test Wasm3.wasm-test \
  Wasm4.wasm-test Wasm5.wasm-test Wasm6.wasm-test Wasm7.wasm-test \
  Wasm8.wasm-test Wasm9.wasm-test Wasm10.wasm-test \
  $(CRYPTO_WASM_FILES)
NEGATIVE	= false

WEB_DIR		?= web
CACHE_DIR	= .cache
OUTPUT_DIR	= .output

ifneq ($(CRYPTO),)
  INCLUDE_CRYPTO=--include $(CRYPTO)
endif
FSTAR		= $(FSTAR_EXE) \
  --cache_dir $(CACHE_DIR) --odir $(OUTPUT_DIR) \
  --include hello-system --include ../krmllib/compat --include ../krmllib/obj \
  --include ../krmllib --include ../runtime \
  $(INCLUDE_CRYPTO) --include ../book --include ../book/notfslit --ext optimize_let_vc \
  --already_cached 'Prims FStar C TestLib Spec.Loops -C.Compat -C.Nullity LowStar WasmSupport' \
  --trivial_pre_for_unannotated_effectful_fns false \
  --cmi --warn_error -274

all: $(FILES) rust $(WASM_FILES) $(CUSTOM) ctypes-test sepcomp-test rust-val-test

# Needs node
wasm: $(WASM_FILES)

# All of the above
everything: all

.PRECIOUS: %.krml

# Use F*'s dependency mechanism and fill out the missing rules.

comma := ,

ifndef MAKE_RESTARTS
ifndef NODEPEND
.depend: .FORCE
	$(call run-with-log, \
	  $(FSTAR) --dep full $(subst .wasm-test,.fst,$(WASM_FILES)) $(subst .test,.fst,$(FILES)) \
	    $(BROKEN) ../runtime/WasmSupport.fst --extract 'krml:*$(comma)-Prims' -o $@ \
	  ,[DEPEND],\
	  .depend)

.PHONY: .FORCE
.FORCE:
endif
endif

include .depend

$(CACHE_DIR)/%.checked: | .depend
	$(call run-with-log,$(FSTAR) $(OTHERFLAGS) -c $< -o $@ && touch $@,[VERIFY] $*,$(CACHE_DIR)/$*)

$(OUTPUT_DIR)/%.krml: | .depend
	$(call run-with-log, \
	  $(FSTAR) $(OTHERFLAGS) --codegen krml \
	    --extract_module $(basename $(notdir $(subst .checked,,$<))) \
	    -o $@ \
	    $< \
	 ,[EXTRACT] $*,\
	  $(OUTPUT_DIR)/$*) 

###############
# Regular (C) #
###############

$(OUTPUT_DIR)/Ctypes2.exe: $(ALL_KRML_FILES) $(KRML_BIN)
	$(KRML) $(EXTRA) -tmpdir $(subst .exe,.out,$@) \
	  -o $@ $(filter %.krml,$^) \
        -skip-compilation $(filter %.krml,$^) \

$(OUTPUT_DIR)/AggressiveInline.exe: EXTRA := $(EXTRA) -faggressive-inlining
$(OUTPUT_DIR)/Malloca.exe: EXTRA := $(EXTRA) -warn-error -6 # Uses VLAs

.PRECIOUS: $(OUTPUT_DIR)/%.exe
$(OUTPUT_DIR)/%.exe: $(ALL_KRML_FILES) $(KRML_BIN)
	$(call run-with-log, \
	  $(KRML) $(EXTRA) -tmpdir $(subst .exe,.out,$@) -no-prefix $(notdir $*) \
	    -o $@ $(filter %.krml,$^) -bundle $(subst _,.,$*)=WindowsHack$(comma)\* \
	,[KRML] $*,$(OUTPUT_DIR)/$*)

.SECONDEXPANSION:
%.test: $(OUTPUT_DIR)/$$(notdir $$(subst .,_,$$*)).exe
	@(if $(NEGATIVE); then ! $^; else $^; fi) && echo -e "\033[01;32m✔\033[00m [TEST,$*]" || (echo -e "\033[01;31m✘\033[00m [TEST,$*]" && false)

ifeq ($(OS),Windows_NT)
  HELLOSYSTEM_LDOPTS = -ldopts -lws2_32
endif

# Various flags to be passed to some targets...
$(OUTPUT_DIR)/NoExtract.exe: EXTRA = -warn-error +2
$(OUTPUT_DIR)/Structs2.exe: EXTRA = -wasm -d force-c wasm-stubs.c
$(OUTPUT_DIR)/ML16.exe: EXTRA = ml16-native.c
$(OUTPUT_DIR)/Scope.exe: EXTRA = -ccopt -O3
$(OUTPUT_DIR)/HigherOrder.exe: EXTRA = -warn-error +9
$(OUTPUT_DIR)/C89.exe: EXTRA = -ccopts -Wno-long-long,-Wno-format,-pedantic,-Wno-c99-extensions -fc89
$(OUTPUT_DIR)/Debug.exe: EXTRA = -d c-calls
$(OUTPUT_DIR)/Server.exe: EXTRA = main-Server.c helpers-Server.c
$(OUTPUT_DIR)/StringLit.exe: EXTRA = -add-include '"krml/internal/compat.h"'
$(OUTPUT_DIR)/TailCalls.exe: EXTRA = -add-include '"krml/internal/compat.h"' -ftail-calls
$(OUTPUT_DIR)/TailCalls2.exe: EXTRA = -ftail-calls
$(OUTPUT_DIR)/FunctionalEncoding.exe: EXTRA = -add-include '"krml/internal/compat.h"'
$(OUTPUT_DIR)/Crypto_Symmetric_Chacha20.exe: EXTRA+=$(CRYPTO_OPTS) main-Chacha.c
$(OUTPUT_DIR)/HelloSystem.exe: EXTRA = -add-include '"system.h"' \
  hello-system/system.c -I hello-system -no-prefix SystemNative \
  -drop SystemNative $(HELLOSYSTEM_LDOPTS)
$(OUTPUT_DIR)/TestKrmlBytes.exe: EXTRA = -add-include '"krml/internal/compat.h"'
$(OUTPUT_DIR)/TestAlloca.exe: EXTRA = -falloca
$(OUTPUT_DIR)/EtaStruct.exe: EXTRA = -fnostruct-passing
$(OUTPUT_DIR)/TotalLoops.exe: EXTRA = -add-include '"krml/internal/compat.h"'
$(OUTPUT_DIR)/CheckedInt.exe: EXTRA = -add-include '"krml/internal/compat.h"'
$(OUTPUT_DIR)/CustomEq.exe: EXTRA = -add-include '"krml/internal/compat.h"'
$(OUTPUT_DIR)/DataTypes.exe: EXTRA = -fnoshort-enums
$(OUTPUT_DIR)/NoShadow.exe: EXTRA = -ccopt -Wshadow -fno-shadow
$(OUTPUT_DIR)/Library.exe: EXTRA = -bundle MemCpyModel= -library MemCpyModel memcpymodel_impl.c
$(OUTPUT_DIR)/IfDef.exe: EXTRA = -ccopt -DX
$(OUTPUT_DIR)/IfDefKrml.exe: EXTRA = -fmicrosoft -ccopts -DBOOLEAN=bool,-DTRUE=true,-DFALSE=false
$(OUTPUT_DIR)/Ctypes2.exe: EXTRA = -ctypes 'Ctypes2,Ctypes4' \
  -bundle 'Ctypes3+Ctypes4=[rename=Lowlevel]' \
  -bundle 'Ctypes2=' \
  -bundle 'Ctypes1=' \
  -bundle '*$(comma)WindowsHack[rename=Leftovers]' \
  -no-prefix 'Ctypes4' -skip-compilation
$(OUTPUT_DIR)/Failwith.exe: EXTRA = -ccopts -Wno-deprecated-declarations,-Wno-infinite-recursion
$(OUTPUT_DIR)/VariableMerge.exe: EXTRA = -fmerge aggressive
$(OUTPUT_DIR)/NameCollision.exe: EXTRA = -no-prefix NameCollisionHelper
$(OUTPUT_DIR)/Intro.exe $(OUTPUT_DIR)/MemCpy.exe: EXTRA = -rst-snippets
$(OUTPUT_DIR)/StaticHeader.exe: EXTRA = -static-header StaticHeaderLib -bundle StaticHeaderAPI=StaticHeaderLib -warn-error @7@8
$(OUTPUT_DIR)/Unused_B.exe: EXTRA=-bundle Unused_A
$(OUTPUT_DIR)/PrivateInclude2.exe: EXTRA=-no-prefix PrivateInclude1 -bundle PrivateInclude1 -add-include 'PrivateInclude1.c:"../../foobar.h"'
$(OUTPUT_DIR)/ExternalEq.exe: EXTRA=-add-include '"external_eq.h"' -ccopt -I. external_eq.c
$(OUTPUT_DIR)/MonomorphizationCrash.exe: EXTRA=monomorphization_crash.c -add-include 'MonomorphizationCrash.c:"monomorphization_crash.h"' -ccopt -I.
$(OUTPUT_DIR)/MonomorphizationSeparate2.exe: EXTRA=-bundle MonomorphizationSeparate1=
$(OUTPUT_DIR)/Loops.exe: EXTRA=-funroll-loops 16
$(OUTPUT_DIR)/GlobalInit.exe: EXTRA=global_init.c

Failure.test: NEGATIVE=true

# Some custom targets

SED=$(shell which gsed >/dev/null 2>&1 && echo gsed || echo sed)
check-useless-alias: $(OUTPUT_DIR)/DepPair.exe
	! grep dtuple2___int32_t____ $(OUTPUT_DIR)/DepPair.out/DepPair.c

count-uu: $(OUTPUT_DIR)/Uu.exe
	[ `grep uu___ $(OUTPUT_DIR)/Uu.out/Uu.c | \
	  $(SED) 's/.*\(uu____\([0-9]\+\)\).*/\1/g' \
	  | uniq | wc -l` = 1 ]

count-eq: $(OUTPUT_DIR)/ExternalEq.exe
	[ `grep eq__ $(OUTPUT_DIR)/ExternalEq.out/ExternalEq.h | wc -l` = 1 ]

count-point: $(OUTPUT_DIR)/FunctionalUpdate.exe
	[ `grep point $(OUTPUT_DIR)/FunctionalUpdate.out/FunctionalUpdate.c | wc -l` = 9 ]
	[ `grep uu___ $(OUTPUT_DIR)/FunctionalUpdate.out/FunctionalUpdate.c | wc -l` = 0 ]

# FStarLang/Karamel#278: in the Deref test, there is expected to be
# only one remaining explicit array access. All other accesses are
# expected to be turned into *b or even local non-array variables.
count-deref: $(OUTPUT_DIR)/Deref.exe
	[ `grep '\[0U\]' $(OUTPUT_DIR)/Deref.out/Deref.c | wc -l` = 1 ]
	! grep _zero_for_deref $(OUTPUT_DIR)/Deref.out/Deref.c $(OUTPUT_DIR)/Deref.out/Deref.h

check-global-init: $(OUTPUT_DIR)/GlobalInit.exe
	! egrep -q GlobalInit2_x $(OUTPUT_DIR)/GlobalInit.out/GlobalInit.h

check-unused: $(OUTPUT_DIR)/Unused_B.exe
	egrep -q 'gets_eliminated.*foobar\)' $(OUTPUT_DIR)/Unused_B.out/Unused_A.c

check-no-constructor: $(OUTPUT_DIR)/NoConstructor.exe
	! grep -q define $(OUTPUT_DIR)/NoConstructor.out/NoConstructor.c

check-no-erased: $(OUTPUT_DIR)/Shifting.exe
	! grep -q erased $(OUTPUT_DIR)/Shifting.out/Shifting.c

check-right-file: $(OUTPUT_DIR)/MonomorphizationSeparate2.exe
	egrep -q 'typedef.*pair' $(OUTPUT_DIR)/MonomorphizationSeparate2.out/MonomorphizationSeparate1.h

check-inline-let: $(OUTPUT_DIR)/InlineLet.exe
	! grep foobar $(OUTPUT_DIR)/InlineLet.out/InlineLet.c
	! grep temp_x $(OUTPUT_DIR)/InlineLet.out/InlineLet.c
	! grep temp_y $(OUTPUT_DIR)/InlineLet.out/InlineLet.c

check-aggressive-inline: $(OUTPUT_DIR)/AggressiveInline.exe
	! grep foobar $(OUTPUT_DIR)/AggressiveInline.out/AggressiveInline.c
	! grep temp_x $(OUTPUT_DIR)/AggressiveInline.out/AggressiveInline.c
	! grep temp_y $(OUTPUT_DIR)/AggressiveInline.out/AggressiveInline.c

check-malloca: $(OUTPUT_DIR)/Malloca.exe
	# Check that a single call to f remains.
	N=$$(grep 'f()' $(OUTPUT_DIR)/Malloca.out/Malloca.c | wc -l) && \
	  if [[ $$N -ne 1 ]]; then echo "Malloca.c seems to have several calls to f" >&2; false; fi

check-comment: $(OUTPUT_DIR)/Comment.exe
	grep -q XXX1 $(OUTPUT_DIR)/Comment.out/Comment.c && \
	grep -q XXX2 $(OUTPUT_DIR)/Comment.out/Comment.c && \
	grep -q XXX3 $(OUTPUT_DIR)/Comment.out/Comment.c && \
	true

check-ignore: $(OUTPUT_DIR)/KrmlTrue.exe
	! grep -q IGNORE $(OUTPUT_DIR)/KrmlTrue.out/KrmlTrue.c

clean:
	rm -rf $(WEB_DIR) .output

distclean: clean
	rm -rf .cache

##########
# CTypes #
##########

LOWLEVEL_DIR=$(OUTPUT_DIR)/Ctypes2.out

$(LOWLEVEL_DIR)/%: ctypes/%
	mkdir -p $(dir $@)
	cp $< $@

ctypes-test: $(LOWLEVEL_DIR)/Client.native
	cd $(LOWLEVEL_DIR) && export LD_LIBRARY_PATH=. && \
	  ./Client.exe && ./Client.bc

sepcomp-test:
	+$(MAKE) -C sepcomp
.PHONY: sepcomp-test

rust-val-test:
	+$(MAKE) -C rust-val
.PHONY: rust-val-test

CTYPES_HAND_WRITTEN_FILES=Client.ml Makefile

.PHONY: $(LOWLEVEL_DIR)/Client.native
$(LOWLEVEL_DIR)/Client.native: $(OUTPUT_DIR)/Ctypes2.exe $(addprefix $(LOWLEVEL_DIR)/,$(CTYPES_HAND_WRITTEN_FILES))
	cd $(dir $@) && env CTYPES_LIB_DIR=$(shell ocamlfind query ctypes) LOWLEVEL_DIR=$(realpath $(LOWLEVEL_DIR)) make

########
# WASM #
########

# A pseudo-target for WASM compilation that does not match any specific file.
# All WASM targets get the -wasm flag; some specific targets may override
# NEGATIVE for negative tests.
.PRECIOUS: $(OUTPUT_DIR)/%.wasm
$(OUTPUT_DIR)/%.wasm: $(ALL_KRML_FILES) $(KRML_BIN)
	$(call run-with-log,$(KRML) -minimal -bundle WasmSupport= -bundle 'FStar.*' -bundle Prims \
	  -bundle C -bundle C.Endianness -bundle C.Nullity -bundle C.String \
	  -bundle TestLib \
	  -bundle $(subst _,.,$*)=WindowsHack$(comma)\* \
	  -backend wasm $(EXTRA) -tmpdir $@ $(JSFILES) -no-prefix $* $(filter %.krml,$^)\
	  ,[KRML-WASM] $*,$(OUTPUT_DIR)/$*)
	[ x$(JSFILES) != x ] && cp $(JSFILES) $@ || true

%.wasm-test: $(OUTPUT_DIR)/%.wasm
	cd $^ && \
	  if ! $(NEGATIVE); then node main.js && echo "\033[01;32m✔\033[00m [WASM-TEST,$*]" || (echo "\033[01;31m✘\033[00m [WASM-TEST,$*]" && false); \
	  else ! node main.js && echo "\033[01;32m✔\033[00m [WASM-TEST,$*]" || (echo "\033[01;31m✘\033[00m [WASM-TEST,$*]" && false); fi

# Customizing some WASM targets.
%/Crypto.Symmetric.Chacha20.wasm: JSFILES=./main-Chacha.js
%/Crypto.Symmetric.Chacha20.wasm: EXTRA+=$(CRYPTO_OPTS) -drop FStar
WasmTrap.wasm-test: NEGATIVE = true

########
# Rust #
########

.PRECIOUS: %.rs
%.rs: $(ALL_KRML_FILES) $(KRML_BIN)
	$(call run-with-log,$(KRML) -minimal -bundle $(notdir $(subst rust,Rust,$*))=\* \
	  -backend rust $(EXTRA) -tmpdir $(dir $@) $(filter %.krml,$^)\
	  ,[KRML-RS] $(basename $*),$*)
	$(SED) -i 's/\(patterns..\)/\1\nmod lowstar { pub mod ignore { pub fn ignore<T>(_x: T) {}}}\n/' $@
	echo 'fn main () { let r = main_ (); if r != 0 { println!("main_ returned: {}\\n", r); panic!() } }' >> $@

$(OUTPUT_DIR)/rustpair.rs: KOPTS+=-fkeep-tuples
$(OUTPUT_DIR)/rustptrfield.rs: KOPTS+=-fkeep-tuples
$(OUTPUT_DIR)/rustamblifetime.rs: KOPTS+=-fkeep-tuples -fno-box

%.rust-test: $(OUTPUT_DIR)/%.rs
	rustc $< -o $*.exe && ./$*.exe

rust: $(RUST_FILES) $(patsubst Rust%.fst,rust%.rust-test,$(filter-out Rust1.fst Rust2.fst Rust3.fst,$(wildcard Rust*.fst)))

RUST_FILES = $(patsubst %.rs,%.rust-test,$(wildcard Rust*.fst))
