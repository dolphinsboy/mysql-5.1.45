# Copyright (C) 2005-2006 MySQL AB
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301  USA

#called from the top level Makefile

MYSQLDATAdir =          $(localstatedir)
MYSQLSHAREdir =         $(pkgdatadir)
MYSQLBASEdir=           $(prefix)
MYSQLLIBdir=            $(pkglibdir)
pkgplugindir =		$(pkglibdir)/plugin
INCLUDES =              -I$(top_srcdir)/include -I$(top_builddir)/include \
			-I$(top_srcdir)/regex \
			-I$(top_srcdir)/sql \
                        -I$(srcdir)
WRAPLIBS=

LDADD =

DEFS =                  @DEFS@
#BEGIN GUOSONG MODIFICATION
noinst_HEADERS =	ha_spartan.h spartan_data.h spartan_index.h
#END GUOSONG MODIFICATION

EXTRA_LTLIBRARIES =	ha_spartan.la
pkgplugin_LTLIBRARIES =	@plugin_spartan_shared_target@
ha_spartan_la_LDFLAGS =	-module -rpath $(pkgplugindir)
ha_spartan_la_CXXFLAGS=	$(AM_CFLAGS) -DMYSQL_DYNAMIC_PLUGIN
ha_spartan_la_CFLAGS =	$(AM_CFLAGS) -DMYSQL_DYNAMIC_PLUGIN
ha_spartan_la_SOURCES =	ha_spartan.cc


EXTRA_LIBRARIES =	libspartan.a
noinst_LIBRARIES =	@plugin_spartan_static_target@
libspartan_a_CXXFLAGS =	$(AM_CFLAGS)
libspartan_a_CFLAGS =	$(AM_CFLAGS)
#BEGIN GUOSONG MODIFICATION
libspartan_a_SOURCES=	ha_spartan.cc spartan_data.cc spartan_index.cc
#END GUOSONG MODIFICATION


EXTRA_DIST =		CMakeLists.txt plug.in
# Don't update the files from bitkeeper
%::SCCS/s.%
