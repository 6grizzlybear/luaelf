Overview
========

**LuaELF** is a Lua 5.1 library for parsing ELF files. It is written almost completely in
Lua, using C modules only when absolutely needed (for example when bitwise operations
are involved). It uses OOP to represent different entities of an ELF file as objects.
It conforms to the most recent [ELF specifications](http://www.sco.com/developers/gabi/latest/contents.html).

Features
========

* Machine independent
* Little and big endian support
* ELF header parsing
* ELF section parsing
* ELF string table parsing
* ELF symbol table parsing
* ELF relocations parsing (REL and RELA)
* Written and tested under Linux, but it should work on Windows and any other OS on which Lua and luarocks are available).

Missing
=======

* Parsing program headers
* Parsing shared library data (DT\_INTERP, DT\_NEEDED...)
* Machine-specific support
* Write support (read-only for now)
* Proper documentation

License
=======

**LuaELF** is licensed under the terms of the MIT license. See the LICENSE file for more details.

Installation
============

**In Windows**: install [LuaForWindows](http://code.google.com/p/luaforwindows) (not tested, should work)

**In Linux**: you need Lua and two C modules: [lpack](http://www.tecgraf.puc-rio.br/~lhf/ftp/lua/) and [Lua BitO](http://bitop.luajit.org/). The
simplest way to install them is by using [luarocks](http://luarocks.org/). In Ubuntu:

    $ sudo apt-get install luarocks
    $ sudo luarocks install lpack
    $ sudo luarocks install luabitop

After the above prerequisites are met, simply download the library into a directory and run a test to make sure everything is OK:

    $ git clone git://github.com/bogdanm/luaelf.git
    $ cd luaelf
    $ lua test.lua <path_to_elf_file>

Usage
-----

**LuaELF** exposes the ELF structure by a series of classes. Use this code to retrieve the class hierarchy:

    local cl = require "classes"
    cl.print_class_tree()

The output will be similar to this:

    object                         base class for all classes
      elfct                        ELF constants
      elfhdr                       ELF header data
      elfistream                   ELF stream interface
        elffstream                 ELF file stream
      elfsect                      ELF section handler
        elfstrtab                  ELF string tab section handler
        elfsymtab                  ELF symbol table section handler
        elfrelsect                 ELF REL/RELA section handler
      elfsym                       ELF symbol data
      elfrel                       ELF REL relocation data
        elfrela                    ELF RELA relocation data
      elf                          ELF file handler

The main classes you'll use for manipulating the various ELF entities are:

* **elf**: the main ELF handler. It initiates the ELF parsing process and it can return
  instances of the header parser (**elfhdr**) and any section object (**elfsect** and subclasses).
* **elfhdr**: ELF header data   
* **elfsect**: represents one ELF section.
* **elfsym**: represents one ELF symbol (from a symbol table section)
* **elfrel** and **elfrela**: ELF relocation information (from a REL or RELA section respectively).

More information will be added to this section as the project evolves. For now, a commented version of
**test.lua** is listed below, it should provide a "quick start" guide to the library API:

    -- ELF test program

    -- IMPORTANT: setup package.path so it can properly access all LuaELF files
    package.path = package.path .. ";lib/?.lua;lib/class/?.lua"
    local sf = string.format
    local ct = require "elfct"
    local elf = require 'elf'

    local args = { ... }
    -- Get a new 'elf' object instance and load the corresponding ELF file.
    -- The 'new' function gets the ELF file name as argument.
    local f = elf.new( args[ 1 ] ):load()
    -- Once loaded, we can retrieve various entities of the ELF file.
    -- The next line retrieves the header object (an 'elfhdr' instance).
    local hdr = f:get_header()
    -- 'filter_sections' can be used to return an array of sections
    -- that respect a set of constraints. It's possible to filter sections
    -- by type and flags
    local sym = f:filter_sections{ type = ct.SHT_SYMTAB }[ 1 ]

    -- Print ELF information as decoded by our 'elfhdr' instance
    print( sf( "File type:  %s", hdr:get_type_str() ) )
    print( sf( "Machine:    %s", hdr:get_machine_str() ) )
    print( sf( "Endianness: %s", hdr:get_endianness() ) )
    print( sf( "ABI:        %s (version %d)", hdr:get_osabi_str(), hdr:get_abi_version() ) )
    print( sf( "Entry:      %08X", hdr:get_entry() ) )
    print( sf( "Sections:   %d, section table at %08X", f:get_num_sections(), hdr:get_sh_off() ) )
    print( sf( "Symbols:    %d, starting at offset %08X", sym:get_num_symbols(), sym:get_offset() ) )
    print ""

    print( "Section table:" )
    print( "Idx  Name                Type         Address  Offset   Size     Flg  Al" )
    -- Ask the 'elf' instance for an iterator for all the sections in the ELF file
    -- The iterator can receive a filter, exactly like filter_sections above
    for _, elfidx, s in f:sect_iterator() do
      print( sf( "%3d %-20s %-12s %08X %08X %08X %3s %3d", elfidx, s:get_name(), s:get_type_str(), s:get_address(),
                  s:get_offset(), s:get_size(), s:get_flags_str(), s:get_alignment() ) )
    end
    print ""

    -- Now we move to symbols. We already got a reference to the symbol table handler
    -- (an instance of 'elfysmtab')
    print "List of the first 10 symbols of type 'function' and global binding:"
    local cnt = 0
    print( "Name                                  Value     Type        Binding   Sect" )
    -- One can also iterate over symbols with a filter. Symbols can be filtered by
    -- type, binding, the section to which the link and visibility. Below we only
    -- filter by type and binding
    for _, s in sym:iterator{ type = ct.STT_FUNC, binding = ct.STB_GLOBAL } do
      print( sf( "%-37s %08X  %-10s  %-8s  %s", s:get_name(), s:get_value(), s:get_type_str(), s:get_binding_str(), f:get_section_name( s:get_section_idx() ) ) )
      cnt = cnt + 1
      if cnt == 10 then break end
    end

In order to use the library in your own projects, you need to setup _package.path_ properly. **test.lua** (above) shows one way to do this.

