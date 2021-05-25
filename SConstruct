import os
import sys
import pprint
import contextlib
import excons
import excons.config
#import excons.cmake
import SCons.Script # pylint: disable=import-error


expat_version = (2, 3, 0)


if sys.platform == "win32":
   mscver = SCons.Script.ARGUMENTS.get("mscver", "14.1")
   if float(mscver) < 14.1:
      print("mscver >=14.1 required")
      sys.exit(1)
   SCons.Script.ARGUMENTS["mscver"] = mscver


env = excons.MakeBaseEnv()


# Configure

dtd = (excons.GetArgument("expat-dtd", 1, int) != 0)
ns = (excons.GetArgument("expat-ns", 1, int) != 0)
cbytes = excons.GetArgument("expat-context-bytes", 1024, int)
attrinfo = (excons.GetArgument("expat-attr-info", 0, int) != 0)
if sys.platform != "win32":
   urandom = (excons.GetArgument("expat-dev-urandom", 1, int) != 0)
else:
   urandom = 0

expatconfig = {"PACKAGE_NAME": "expat",
               "PACKAGE_VERSION": ".".join(map(str, expat_version)),
               "XML_DTD": dtd,
               "XML_NS": ns,
               "XML_DEV_URANDOM": urandom,
               "XML_ATTR_INFO": attrinfo,
               "XML_CONTEXT_BYTES": cbytes}

@contextlib.contextmanager
def preserve_variable(env, name):
   oldvar = env[name]
   env[name] = type(oldvar)(oldvar)
   try:
      yield
   except:
      raise
   finally:
      env[name] = oldvar

if not env.GetOption("clean"):
   conf = SCons.Script.Configure(env)
   with preserve_variable(conf.env, "CFLAGS"):
      conf.env.Append(CFLAGS=["-fvisibility=hidden"])
      expatconfig["FLAG_VISIBILITY"] = conf.CheckCC()
   with preserve_variable(conf.env, "CFLAGS"):
      conf.env.Append(CFLAGS=["-fno-strict-aliasing"])
      expatconfig["FLAG_NO_STRICT_ALIASING"] = conf.CheckCC()
   expatconfig["HAVE_DLFCN_H"] = conf.CheckHeader("dlfcn.h")
   expatconfig["HAVE_FCNTL_H"] = conf.CheckHeader("fcntl.h")
   expatconfig["HAVE_INTTYPES_H"] = conf.CheckHeader("inttypes.h")
   expatconfig["HAVE_MEMORY_H"] = conf.CheckHeader("memory.h")
   expatconfig["HAVE_STDINT_H"] = conf.CheckHeader("stdint.h")
   expatconfig["HAVE_STDLIB_H"] = conf.CheckHeader("stdlib.h")
   expatconfig["HAVE_STRINGS_H"] = conf.CheckHeader("strings.h")
   expatconfig["HAVE_STRING_H"] = conf.CheckHeader("string.h")
   expatconfig["HAVE_SYS_STAT_H"] = conf.CheckHeader("sys/stat.h")
   expatconfig["HAVE_SYS_TYPES_H"] = conf.CheckHeader("sys/types.h")
   expatconfig["HAVE_UNISTD_H"] = conf.CheckHeader("unistd.h")
   expatconfig["HAVE_GETPAGESIZE"] = conf.CheckFunc("getpagesize", "#include <unistd.h>\n")
   expatconfig["HAVE_MMAP"] = conf.CheckDeclaration("mmap", "#include <sys/mman.h>\n")
   expatconfig["HAVE_GETRANDOM"] = conf.CheckDeclaration("getrandom", "#include <sys/random.h>\n")
   expatconfig["HAVE_LIBBSD"] = False
   expatconfig["HAVE_ARC4RANDOM_BUF"] = conf.CheckDeclaration("arc4random_buf", "#include <stdlib.h>\n")
   expatconfig["HAVE_ARC4RANDOM"] = conf.CheckDeclaration("arc4random", "#include <stdlib.h>\n")
   expatconfig["STDC_HEADERS"] = all(map(conf.CheckHeader, ["stdlib.h", "stdarg.h", "string.h", "float.h"]))
   if expatconfig["HAVE_SYS_TYPES_H"]:
      if not conf.CheckType("off_t", "#include <sys/types.h>\n"):
         print("'off_t' not defined")
         SCons.Script.Exit(1)
      if not conf.CheckType("size_t", "#include <sys/types.h>\n"):
         print("'size_t' not defined")
         SCons.Script.Exit(1)
   else:
      expatconfig["OFF_T"] = "long"
      expatconfig["SIZE_T"] = "unsigned"
   src = """
      #include <stdlib.h>  /* for NULL */
      #include <unistd.h>  /* for syscall */
      #include <sys/syscall.h>  /* for SYS_getrandom */
      int main() {
         syscall(SYS_getrandom, NULL, 0, 0);
         return 0;
      }"""
   # TryCompile returns an integer to use as a boolean
   expatconfig["HAVE_SYSCALL_GETRANDOM"] = (True if conf.TryCompile(src, ".c") else False)
   src = """
      #include <stdio.h>
      int main() {
         unsigned int i = 1;
         char *c = (char*)&i;
         if (1 == (int)*c) {
            fprintf(stdout, "Little Endian");
         } else {
            fprintf(stdout, "Big Endian");
         }
         return 0;
      }
      """
   # TryCompile returns an integer to use as boolean and program stdout (stderr not captured)
   success, out = conf.TryRun(src, ".c")
   if success:
      if out.strip() == "Little Endian":
         expatconfig["WORDS_BIGENDIAN"] = False
         expatconfig["BYTEORDER"] = "1234"
      else:
         expatconfig["WORDS_BIGENDIAN"] = True
         expatconfig["BYTEORDER"] = "4321"
   else:
      print("Failed to figure endianness")
      SCons.Script.Exit(1)
   env = conf.Finish()

def _bool2str(val):
   return ("1" if val else "0")
GenerateConfig = excons.config.AddGenerator(env, "expat", expatconfig, converters={bool: _bool2str})

GenerateConfig(excons.OutputBaseDirectory() + "/include/expat_config.h", "expat_config.h.in")


# Build options

defs = ["XML_BUILDING_EXPAT", "HAVE_EXPAT_CONFIG_H"]
cflags = []

_static = (excons.GetArgument("expat-static", 1, int) != 0)
if _static:
   defs.append("XML_STATIC")

chart = excons.GetArgument("expat-char-type", "char")
if not chart in ("char", "ushort", "wchar_t"):
   SCons.Script.Exit(1)
elif chart != "char":
   defs.append("EXPAT_CHAR_TYPE=%s" % chart)
   defs.append("XML_UNICODE")
   if chart == "wchar_t":
      defs.append("XML_UNICODE_WCHAR_T")
      if sys.platform != "win32":
         cflags.append("-fshort-wchar")

if expatconfig.get("FLAG_NO_STRICT_ALIASING"):
   cflags.append("-fno-strict-aliasing")

if expatconfig.get("FLAG_VISIBILITY"):
   defs.append("XML_ENABLE_VISIBILITY=1")

if excons.GetArgument("expat-large-size", 0, int) != 0:
   defs.append("XML_LARGE_SIZE")
if excons.GetArgument("expat-min-size", 0, int) != 0:
   defs.append("XML_MIN_SIZE")


# Exports

def ExpatName(static=True):
   if sys.platform == "win32":
      # EXPAT_MSVC_STATIC_CRT is hardcoded to 0, static suffix can be 'MT'
      # EXPAT_CHAR_TYPE is hardcoded to 'char', suffix won't include 'w'
      libname = "libexpat"
      if excons.GetArgument("debug", 0, int) != 0:
         libname += "d"
      if static:
         libname += "MD"
      return libname
   else:
      return "expat"

def ExpatPath(static=True):
   name = ExpatName(static)
   if sys.platform == "win32":
      libname = name + ".lib"
   else:
      libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
   return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireExpat(env, static=True):
   if static:
      env.Append(CPPDEFINES=["XML_STATIC"])
   env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
   env.Append(LIBPATH=[excons.OutputBaseDirectory() + "/lib"])
   excons.Link(env, ExpatPath(static), static=static, force=True, silent=True)


# Targets

prjs = [
   {
      "name": ExpatName(_static),
      "alias": "expat",
      "type": "staticlib" if _static else "sharedlib",
      "defs": defs,
      "symvis": ("hidden" if (expatconfig.get("FLAG_VISIBILITY") or _static) else "default"),
      "ccflags": cflags,
      "version": expatconfig["PACKAGE_VERSION"],
      "soname": "lib%s.so.%s" % (ExpatName(False), expat_version[0]),
      "install_name": "lib%s.%s.so" % (ExpatName(False), expat_version[0]),
      "install": {"include": ["expat/lib/expat.h",
                              "expat/lib/expat_external.h"]},
      "srcs": ["expat/lib/xmltok.c",
               "expat/lib/xmlparse.c",
               "expat/lib/xmlrole.c"]
   }
]

excons.AddHelpOptions(expat="""EXPAT OPTIONS
  expat-static=0|1                    : Toggle between static and shared library build               [1]
  expat-dtd=0|1                       : Enable DTD support                                           [1]
  expat-ns=0|1                        : Enable namespace support                                     [1]
  expat-char-type=char|ushort|wchar_t : Character type to use                                        [char]
  expat-large-size=0|1                : Use 64 bits offsets                                          [0]
  expat-min-size=0|1                  : Get a smaller (but slower) parser                            [0]
  expat-context-bytes=<int>           : How much context to retain around current parse point        [1024]
  expat-attr-info=0|1                 : Allow retrieving byte offsets for attribute names and values [0]
  expat-dev-urandom=0|1               : Include code reading entropy from /dev/urandom               [1] (no effect on windows)
""")

excons.DeclareTargets(env, prjs)

SCons.Script.Export("ExpatName ExpatPath RequireExpat")
