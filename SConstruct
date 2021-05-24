import os
import sys
import pprint
import contextlib
import excons
import excons.config
#import excons.cmake
import SCons.Script # pylint: disable=import-error



if sys.platform == "win32":
   mscver = SCons.Script.ARGUMENTS.get("mscver", "14.1")
   if float(mscver) < 14.1:
      print("mscver >=14.1 required")
      sys.exit(1)
   SCons.Script.ARGUMENTS["mscver"] = mscver


env = excons.MakeBaseEnv()


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
               "PACKAGE_VERSION": "2.3.0",
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
      expatconfig["OFF_T"] = "off_t"
      expatconfig["SIZE_T"] = "size_t"
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

pprint.pprint(expatconfig)

GenerateConfig = excons.config.AddGenerator(env, "expat", expatconfig)

GenerateConfig(excons.OutputBaseDirectory() + "/include/expat_config.h", "expat_config.h.in")

# Build options

defs = ["HAVE_EXPAT_CONFIG_H"]
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

if excons.GetArgument("expat-large-size", 0, int) != 0:
   defs.append("XML_LARGE_SIZE")
if excons.GetArgument("expat-min-size", 0, int) != 0:
   defs.append("XML_MIN_SIZE")

#SCons.Script.Exit(0)

prjs = [
   {
      "name": ExpatName(_static),
      "alias": "expat",
      "type": "staticlib" if _static else "sharedlib",
      "defs": defs,
      "cflags": cflags,
      "srcs": ["expat/lib/xmltok.c",
               "expat/lib/xmlparse.c",
               "expat/lib/xmlrole.c"]
   }
]

"""
if sys.platform != "win32":
   envcf = os.environ.get("CFLAGS", "")
   if not "-fPIC" in envcf:
      envcf += " -fPIC"
      os.environ["CFLAGS"] = envcf

prjs = [
   {
      "name": "expat",
      "type": "cmake",
      "cmake-root": "expat",
      "cmake-opts": {"EXPAT_BUILD_TOOLS": 0,
                     "EXPAT_BUILD_EXAMPLES": 0,
                     "EXPAT_BUILD_TESTS": 0,
                     "EXPAT_BUILD_DOCS": 0,
                     "EXPAT_ENABLE_INSTALL": 1,
                     "EXPAT_BUILD_PKGCONFIG": 0,
                     "EXPAT_SHARED_LIBS": 0 if _static else 1,
                     "EXPAT_LARGE_SIZE": 1,
                     "EXPAT_CHAR_TYPE": "char",
                     "EXPAT_MSVC_STATIC_CRT": 0,
                     "CMAKE_INSTALL_LIBDIR": "lib"},
      "cmake-cfgs": excons.CollectFiles(".", patterns=["CMakeLists.txt"], recursive=True),
      "cmake-srcs": excons.CollectFiles("expat/src", patterns=["*.c"]),
      "cmake-outputs": ["include/expat.h",
                        "include/expat_config.h",
                        "include/expat_external.h",
                        ExpatPath(static=_static)]
   }
]
"""

excons.AddHelpOptions(expat="""EXPAT OPTIONS
  expat-static=0|1   : Toggle between static and shared library build [1]""")

excons.DeclareTargets(env, prjs)

SCons.Script.Export("ExpatName ExpatPath RequireExpat")
