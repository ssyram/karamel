#############################
# This is the main Makefile #
#############################

# This tutorial assumes you have a degree of familiarity with GNU make,
# including automatic variables such as $@, $< and $^. Some mandatory reading if
# you are not fluent in GNU make:
# - https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html#Automatic-Variables
# - https://www.gnu.org/software/make/manual/html_node/Pattern-Intro.html#Pattern-Intro
# - https://www.gnu.org/software/make/manual/html_node/Pattern_002dspecific.html#Pattern_002dspecific

# I usually prefer to rule out OSX make on the basis that it doesn't have the
# shortest stem rule, which is incredibly useful.
ifeq (3.81,$(MAKE_VERSION))
  $(error You seem to be using the OSX antiquated Make version. Hint: brew \
    install make, then invoke gmake instead of make)
endif


# Main entry points (first one is default)
# ----------------------------------------

all: dist/libbignum.a

test: tests/c-tests.test obj/specs-test.test

%.test: %.exe
	$<

include Makefile.include

# Definition of F* flags
# ----------------------

# This flag controls what gets extracted to OCaml. Generally, we don't extract
# the FStar namespace since it's already extracted and packaged as the ocamlfind
# package fstar.lib. Here, unlike in -bundle, +Spec matches both Spec and
# Spec.*
FSTAR_EXTRACT = --extract 'OCaml:-* +Spec'

# Some reasonable flags to turn on:
# - 247: checked file not written because some of its dependencies...
# - 285: missing or file not found, almost always something to act on
# - 241: stale dependencies, almost always a sign that the build is incorrect
#
# But also:
# - --cmi, for cross-module inlining, a must-have for anyone who relies on
#   inline_for_extraction in the presence of interfaces
# - --cache_dir, to place checked files there

FSTAR_NO_FLAGS = $(FSTAR_EXE) --ext optimize_let_vc \
  --odir obj $(FSTAR_INCLUDES) --cmi \
  --already_cached 'Prims FStar LowStar C Spec.Loops TestLib WasmSupport' --warn_error '+241@247+285' \
  --cache_dir obj

# Initial dependency analysis
# ---------------------------

# Important to wildcard both fst and fsti since there are fstis without fsts,
# etc. Note that I'm using wildcard rather than assume $(SHELL) is bash and has
# fancy globbing rules -- particularly true on Windows.
FSTAR_ROOTS = $(wildcard $(addsuffix /*.fsti,$(SOURCE_DIRS))) \
  $(wildcard $(addsuffix /*.fst,$(SOURCE_DIRS))) \


# This is the only bulletproof way that I know of forcing a regeneration of the
# .depend file every single time. Why, you may ask? Well, it's frequent enough
# to add a new file that you don't want to decipher a cryptic error only to
# remember you should run `make depend`. Also, if you move files around, it's
# good to force regeneration even though .depend may be more recent than the
# mtime of the moved files.
ifndef MAKE_RESTARTS
.depend: .FORCE
	$(FSTAR_NO_FLAGS) --dep full $(notdir $(FSTAR_ROOTS)) $(FSTAR_EXTRACT) -o $@

.PHONY: .FORCE
.FORCE:
endif

include .depend

# Verification
# ------------

# Everest-specific idiom: all makefiles accept OTHERFLAGS, for instance, if one
# wants to rebuild with OTHERFLAGS="--admit_smt_queries true". We just don't
# pass such flags to the dependency analysis.
FSTAR = $(FSTAR_NO_FLAGS) $(OTHERFLAGS)

# Creating these directories via a make rule, rather than rely on F* creating
# them, as two calls to F* might race.
obj:
	mkdir $@

# We allow some specific pattern rules to be added here, relying on the shortest
# stem rule for them to take precedence. For instance, you may want to do:
#
# obj/Bignum.Impl.fst.checked: FSTAR_FLAGS = "--query_stats"
#
# (Note: for options that control the SMT encoding, such as
# --smtencoding.nl_arith_repr native, please make sure you also define them in
# Makefile.common for %.fst-in otherwise you'll observe different behaviors
# between interactive and batch modes.)
#
# By default, however, variables are inherited through the dependencies, meaning
# that the example above would normally set these FSTAR_FLAGS for any .checked
# that is rebuilt because it's a dependency of Bignum.Impl.fst.checked.
#
# To avoid this unpleasant behavior, the most general pattern rule (longest
# stem) also defines a suitable default value for FSTAR_FLAGS.
%.checked: FSTAR_FLAGS=

# Note: F* will not change the mtime of a checked file if it is
# up-to-date (checksum matches, file unchanged), but this will confuse
# make and result in endless rebuilds. So, we touch that file.
%.checked: | obj
	$(FSTAR) $(FSTAR_FLAGS) -c $< -o $@ && touch -c $@

# Extraction
# ----------

.PRECIOUS: obj/%.ml
obj/%.ml:
	$(FSTAR) --codegen OCaml $< -o $@

# By default, krml extraction in F* will extract every module into a
# single out.krml file. Make sure to extract the single module we want.
.PRECIOUS: obj/%.krml
obj/%.krml:
	$(FSTAR) $< --codegen krml \
	--extract_module $(basename $(notdir $(subst .checked,,$<))) \
	-o $@

obj/Specs_Driver.ml: specs/ml/Specs_Driver.ml
	# This ensures that all the source directories are not polluted with
	# build artifacts
	cp $< $@

# F* --> C
# --------

KRML=$(KRML_HOME)/krml

# Note: the implementation of the intrinsic uses external linkage, but you could
# easily turn this file into a .h, use -add-include '"Impl_Bignum_Intrinsics.h"'
# and pass -static-header Impl.Bignum.Intrinsics as described in the
# documentation.
HAND_WRITTEN_C_FILES = code/c/Impl_Bignum_Intrinsics.c

# This is now the preferred and recommended way to compile C code with KaRaMeL.
#
# KaRaMeL (via -skip-compilation) only generates a stub Makefile in dist/,
# instead of acting as a C compilation driver like it did before. The Makefile
# that is generated by KaRaMeL is basic, but takes into account:
# - the -o option to determine what is being built
# - the C files passed on the command line, which will be linked together
# - C compiler flags passed to KaRaMeL via -ccopts
#
# This Makefile is called Makefile.basic and should be enough for all your basic
# needs. If you need something more fancy, you can easily author your own custom
# dist/Makefile, which includes Makefile.basic, then proceeds to redefine /
# tweak some variables.
#
# Note that you are of course more than welcome to define your own
# CMakeLists.txt or whatever your favorite build system is: this tutorial only
# describes the supported canonical way of compiling code.
#
# See the advanced topics section for an in-depth explanation of how the -bundle
# option works. We also use -minimal.
dist/Makefile.basic: $(filter-out %prims.krml,$(ALL_KRML_FILES)) $(HAND_WRITTEN_C_FILES)
	mkdir -p $(dir $@)
	cp $(HAND_WRITTEN_C_FILES) $(dir $@)
	$(KRML) -tmpdir $(dir $@) -skip-compilation \
	  $(filter %.krml,$^) \
	  -warn-error @4@5@18 \
	  -fparentheses \
	  -bundle Impl.Bignum.Intrinsics= \
	  -bundle 'LowStar.*,Prims' \
	  -bundle Impl.Bignum=Impl.Bignum.*,Spec.*[rename=Bignum] \
	  -minimal \
	  -bundle 'FStar.*' \
	  -add-include '<stdint.h>' \
	  -add-include '"krml/internal/target.h"' \
	  $(notdir $(HAND_WRITTEN_C_FILES)) \
	  -o libbignum.a

# Compiling the generated C code
# ------------------------------

# Here we do a recursive make invocation because a new dependency analysis needs
# to be performed on the generated C files. This recursive make deals with all
# the files copied in dist, which includes karamel-generated and hand-written,
# copied files from HAND_WRITTEN_C_FILES.

dist/libbignum.a: dist/Makefile.basic
	$(MAKE) -C $(dir $@) -f $(notdir $<)

# Compiling the generated OCaml code
# ----------------------------------

# This is much more difficult, and you're probably better off calling OCamlbuild
# or Dune at this stage. Nonetheless, I still show how to compile the generate
# OCaml code with only GNU make and without relying on an external tool or an
# extra dependency analysis.

# First complication... no comment.
# NOTE: if F* was installed via opam, then this is redundant
ifeq ($(OS),Windows_NT)
  export OCAMLPATH := $(shell $(FSTAR_EXE) --locate_ocaml);$(OCAMLPATH)
else
  export OCAMLPATH := $(shell $(FSTAR_EXE) --locate_ocaml):$(OCAMLPATH)
endif

# Second complication: F* generates a list of ML files in the reverse linking
# order. No POSIX-portable way of reversing the order of a list.
TAC = $(shell which tac >/dev/null 2>&1 && echo "tac" || echo "tail -r")

ALL_CMX_FILES = $(patsubst %.ml,%.cmx,$(shell echo $(ALL_ML_FILES) | $(TAC)))

# Third complication: F* does not know about our hand-written files. So, we need
# to manually add dependency edges in the graph. In our case, the test driver
# needs all files to be built before. This is convenient because the
# hand-written file only needs to be inserted at the end of the list of CMX
# files. If it had to be inserted somewhere in the middle of the topological
# order, it would be trickier...

obj/Specs_Driver.cmx: $(ALL_CMX_FILES)

# Finally, how to compile things...

OCAMLOPT = ocamlfind opt -package fstar.lib -linkpkg -g -I $(BIGNUM_HOME)/obj -w -8-20-26

.PRECIOUS: obj/%.cmx
obj/%.cmx: obj/%.ml
	$(OCAMLOPT) -c $< -o $@

obj/specs-test.exe: $(ALL_CMX_FILES) obj/Specs_Driver.cmx
	$(OCAMLOPT) $^ -o $@

# Compiling the hand-written test
# -------------------------------

CFLAGS += -I dist -I $(KRML_HOME)/include

tests/c-tests.exe: tests/c-tests.o dist/libbignum.a
	$(CC) $^ -o $@

