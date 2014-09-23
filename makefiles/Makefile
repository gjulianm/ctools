# Generic Makefile
# Supports
#  - Automatic dependency detection for source and libraries.
#  - Multiple configurations.
#  - Library compilation (static and dynamic linking).

CC = gcc
CFLAGS = -Wall -D_GNU_SOURCE
DEBUG_CFLAGS = -ggdb -DDEBUG -ftrapv 
RELEASE_CFLAGS = -O3 
PROFILE_CFLAGS = $(RELEASE_CFLAGS) -pg -static-libgcc

# External libraries used and (optional) their paths if the compiler
# 	doesn't recognize them.
EXTERNAL_LIBS = 
LIBRARY_PATHS = /opt/local/lib

# Project name, just for packing and some automated target guesses.
PROJECT_NAME = __project_name__

# Each target will be compiled as a binary in bin/targetname, using
#	src/targetname.c as main file. src/targetname.c will not be included
#	in other binaries.
TARGETS = $(PROJECT_NAME)

CONFS = debug release
DEFAULT_CONF = release

TAR_EXCLUDES = bin obj doc .tar.gz .git tasks \
		cscope.out $(PROJECT_NAME).sublime-project $(PROJECT_NAME).sublime-workspace \
		*.dat callgrind.* gmon.out

## Folder structure
OBJDIR = obj
SRCDIR = src
INCDIR = includes
BINDIR = bin
DOCDIR = doc
LIBDIR = lib
LIBSRCDIR = srclib
TESTDIR = tests

# Header files include path
INCLUDES = -I./includes/

#### Internal variables ####

INCS := $(wildcard $(INCDIR)/*.h)
SRCS := $(wildcard $(SRCDIR)/*.c)
OBJS := $(patsubst %.c,%.o, $(SRCS))

LIB_SRCS := $(wildcard $(LIBSRCDIR)/**/*.c)
LIB_OBJS := $(addprefix $(OBJDIR)/, $(patsubst %.c,%.o, $(LIB_SRCS)))
LIB_NAMES := $(patsubst $(LIBSRCDIR)/%, %, $(wildcard $(LIBSRCDIR)/*))
LIBS :=	$(addsuffix .a, $(LIB_NAMES))
LIBS +=	$(addsuffix .so, $(LIB_NAMES))
LIB_DEPS := $(addsuffix .deps, $(addprefix $(OBJDIR)/., $(LIB_NAMES)))
LIB_OBJDIRS := $(foreach conf, $(CONFS), $(addprefix $(OBJDIR)/$(conf)/$(LIBSRCDIR)/, $(LIB_NAMES)))
LIB_OUTDIRS := $(addprefix $(LIBDIR)/, $(CONFS))
E_LIBS := $(EXTERNAL_LIBS)
E_LIBS += $(addprefix -L, $(LIBRARY_PATHS))

TEST_SRCS := $(wildcard $(TESTDIR)/*.c)
TEST_OBJS := $(patsubst %.c,%.o, $(TEST_SRCS))
OBJS_NOMAIN := $(filter-out $(addprefix %/, $(addsuffix .o, $(TARGETS))), $(OBJS))

# Documentation targets
DOC_TEXS := $(wildcard $(DOCDIR)/*.tex)
DOC_TEXS += $(DOCDIR)/latex/refman.tex
DOC_PDFS := $(patsubst %.tex, %.pdf, $(DOC_TEXS))

DOXY_OUTPUTS := $(filter-out $(DOCDIR)/, $(dir $(wildcard $(DOCDIR)/*/)))

FMT_BOLD := $(shell tput bold)
FMT_NORM := $(shell tput sgr0)

TAR_EXCLUDES_ARG = $(addprefix --exclude=, $(TAR_EXCLUDES))

### Makefile plugins

### End Makefile plugins

.PRECIOUS: %.o %.d %.g 
.PHONY: benchmark clean pack doxydoc docclean benchmark-run configs $(TARGETS)
 
all: release test
libs: $(LIBS)
final: all docs pack

#### Makefile rules ####

### Misc. rules ###

# Generate configuration data file
$(CONFS): configs 

# Output current CFLAGS to file flags.dat
flags.dat: Makefile
	@echo $(CFLAGS) > $@

# Clang static analyzer
scan: clean
	@scan-build make debug

# Test target
test: $(BINDIR)/debug/test $(TEST_DEPS)
	
#### Automated dependency detection #### 

depend: $(OBJDIR)/.deps $(OBJDIR)/.test.deps $(LIB_DEPS) 

# Dependency detection for source files
$(OBJDIR)/.deps: $(SRCS) Makefile | $(OBJDIR)
	@-rm -f $(OBJDIR)/.deps
	@$(CC) $(CFLAGS) $(INCLUDES) -MM $(filter-out Makefile, $^) >> $(OBJDIR)/.deps;
	@awk '{if (sub(/\\$$/,"")) printf "%s", $$0; else print $$0}' $@ > "$@.0"
	@mv "$@.0" $@
	@for c in $(CONFS); do \
		awk '{printf("$(OBJDIR)/%s/$(SRCDIR)/%s\n", conf, $$0)}' conf=$$c $@ >> "$@.0"; \
	done
	@mv "$@.0" $@

# Dependency detection for test files. It's more or less the same code that before,
# 	but we have to account for special test directories.
$(OBJDIR)/.test.deps: $(TEST_SRCS) Makefile | $(OBJDIR)
	@-rm -f $@
	@$(CC) $(CFLAGS) $(INCLUDES) -MM $(filter-out Makefile, $^) >> $@;
	@awk '{if (sub(/\\$$/,"")) printf "%s", $$0; else print $$0}' $@ > "$@.0"
	@mv "$@.0" $@
	@for c in $(CONFS); do \
		awk '{printf("$(OBJDIR)/%s/$(TESTDIR)/%s\n", conf, $$0)}' conf=$$c $@ >> "$@.0"; \
	done
	@mv "$@.0" $@

# Library dependency detection.
#  Generates automatically dependences for the *.a and *.so files.
$(OBJDIR)/.%.deps: $(LIBSRCDIR)/%/*.c Makefile | $(OBJDIR)
	@-rm -f $@
	@$(CC) $(CFLAGS) $(INCLUDES) -MM $(filter-out Makefile, $^) >> $@;
	@awk '{if (sub(/\\$$/,"")) printf "%s", $$0; else print $$0}' $@ > "$@.0"
	@mv "$@.0" $@
	@for c in $(CONFS); do \
		awk '{printf("$(OBJDIR)/%s/$(LIBSRCDIR)/$*/%s | $(LIBDIR)\n", conf, $$0)}' conf=$$c $@ >> "$@.0"; \
		echo "$(LIBDIR)/$$c/$*.a: $(patsubst %.c, $(OBJDIR)/$$c/%.o, $(filter-out Makefile, $^))" >> "$@.0"; \
		echo "$(LIBDIR)/$$c/$*.so: $(patsubst %.c, $(OBJDIR)/$$c/%.o, $(filter-out Makefile, $^))" >> "$@.0"; \
	done
	@mv "$@.0" $@
	@echo "$*.a: $(LIBDIR)/$(DEFAULT_CONF)/$*.a" >> $@
	@echo "$*.so: $(LIBDIR)/$(DEFAULT_CONF)/$*.so" >> $@

-include $(OBJDIR)/.deps
-include $(OBJDIR)/.test.deps
-include $(LIB_DEPS)

#### Configuration flag creation and detection ####
configs: $(OBJDIR)/.conf_flags.mk

$(OBJDIR)/.conf_flags.mk: Makefile | $(OBJDIR)
	@echo "$(FMT_BOLD)Generating configurations...$(FMT_NORM)"
	@-rm -f $@
	@for c in $(CONFS); do \
		cnf=$$(echo $$c | tr '[:lower:]' '[:upper:]'); \
		for t in $(TARGETS); do \
			echo "$(BINDIR)/$$c/$$t: CFLAGS += \$$(""$$cnf""_CFLAGS)" >> $@; \
			echo "$(BINDIR)/$$c/$$t: LDFLAGS += \$$(""$$cnf""_LDFLAGS)" >> $@; \
			echo "$(BINDIR)/$$c/$$t: \$$(addprefix $(OBJDIR)/$$c/, \$$(OBJS_NOMAIN)) $(OBJDIR)/$$c/$(SRCDIR)/$$t.o $(addprefix $(LIBDIR)/$$c/, $(LIBS))" >> $@; \
		done; \
		echo "$(BINDIR)/$$c/test: CFLAGS += \$$(""$$cnf""_CFLAGS)" >> $@; \
		echo "$(BINDIR)/$$c/test: LDFLAGS += \$$(""$$cnf""_LDFLAGS)" >> $@; \
		echo "$(BINDIR)/$$c/test: \$$(addprefix $(OBJDIR)/$$c/, \$$(OBJS_NOMAIN)) $(OBJDIR)/$$c/$(TESTDIR)/test.o \$$(addprefix obj/$$c/, \$$(TEST_OBJS)) $(addprefix $(LIBDIR)/$$c/, $(LIBS)) | $(TEST_DEPS)" >> $@; \
		echo "$$c: $(addprefix $(BINDIR)/$$c/, $(TARGETS) test)" >> $@; \
	done
	@for t in $(TARGETS); do \
		echo "$$t: $(BINDIR)/$(DEFAULT_CONF)/$$t" >> $@; \
	done

-include $(OBJDIR)/.conf_flags.mk

#### Directory creation ####

$(OBJS): | $(OBJDIR)

$(OBJDIR): Makefile
	@for c in $(CONFS); do \
		echo "$(FMT_BOLD)Creating object directories for configuration $$c...$(FMT_NORM)"; \
		mkdir -p $(OBJDIR)/$$c/$(SRCDIR); \
		mkdir -p $(OBJDIR)/$$c/$(LIBSRCDIR); \
		mkdir -p $(OBJDIR)/$$c/$(TESTDIR); \
	done 

$(BINDIR): Makefile
	@echo "$(FMT_BOLD)Creating bin directory$(FMT_NORM)"
	@for c in $(CONFS); do \
		mkdir -p $(BINDIR)/$$c; \
	done

$(LIBDIR): Makefile
	@echo "$(FMT_BOLD)Creating lib directory$(FMT_NORM)"
	@mkdir -p $(LIB_OUTDIRS)
	@mkdir -p $(LIB_OBJDIRS)

## Cleaning 

clean: docclean codeclean resultclean
	@echo "$(FMT_BOLD)Directory clean.$(FMT_NORM)"

codeclean:
	@echo "$(FMT_BOLD)Cleaning build folders...$(FMT_NORM)"
	@-rm -rf $(OBJDIR) $(BINDIR) $(LIBDIR)	

resultclean:
	@echo "$(FMT_BOLD)Removing result...$(FMT_NORM)"
	@-rm -rf *.dat *.g *.gdat *.png

#### Compilation rules #### 

# Static library
$(LIBDIR)/%.a: | $(LIBDIR)
	@echo "$(FMT_BOLD)Building library $(notdir $@)... $(FMT_NORM)"
	@$(AR) -rv $@ $^

# Dynamic library
$(LIBDIR)/%.so: | $(LIBDIR)
	@echo "$(FMT_BOLD)Building library $(notdir $@)... $(FMT_NORM)"
	@$(CC) $(CFLAGS) -shared $^ $(E_LIBS) -o $@ 

# Objects
$(OBJDIR)/%.o: | $(OBJDIR) depend configs
	@echo "$< -> $@"
	@$(CC) $(CFLAGS) $(INCLUDES) -c $< -o $@

# Executables
$(BINDIR)/%: | $(BINDIR) depend configs 
	@echo "$(FMT_BOLD)Building final target: $* $(FMT_NORM)"
	@$(CC) $(CFLAGS) $(INCLUDES) $^ $(E_LIBS) -o $@ 


#### Documentation ####

docs: doxydoc $(DOC_PDFS)

doxydoc: 
	@echo "$(FMT_BOLD)Building doxygen docs... $(FMT_NORM)"
	@doxygen
	@echo "Converting files to correct encoding..."
	@for f in doc/man/man3/*; do \
		iconv -t latin1 "$$f" > iconv.tmp; \
		mv iconv.tmp "$$f"; \
	done

%.pdf: %.tex
	@echo "$(FMT_BOLD)Latexmk: generating $@ $(FMT_NORM)"
	@cd $(dir $<); latexmk -pdf -silent $(notdir $<)

docclean:
	@echo "$(FMT_BOLD)Removing LaTeX folders...$(FMT_NORM)"
	@cd $(DOCDIR); (which latexmk > /dev/null && latexmk -C) || true # Clean tex files if latexmk is present.
	@echo "$(FMT_BOLD)Removing doxygen output...$(FMT_NORM)"
	@[ -z "$(DOXY_OUTPUTS)" ] || rm -r $(DOXY_OUTPUTS)


#### Packing ####

pack: $(DOC_PDFS) codeclean
	@cd ..; tar $(TAR_EXCLUDES_ARG) -czf $(PROJECT_NAME).tar.gz $(lastword $(notdir $(CURDIR)))
	@echo "Packed $(PROJECT_NAME).tar.gz in parent directory."

